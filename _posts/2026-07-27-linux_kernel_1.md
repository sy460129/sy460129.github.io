---
title: "Linux Kernel: task_struct와 cred"
date: 2026-07-27 10:00:00 +0900
categories: [Kernel, Pwnable]
tags: [linux, kernel]
---

# 1. task_struct
리눅스 커널은 유저 공간의 프로세스와 쓰레드를 구분하지 않고, 모두 스케줄링의 기본 단위인 **Task**로 관리한다.  
이 Task를 표현하는 거대한 C 구조체가 바로 `task_struct`이다.

### 멤버 변수 필드 (sched.h)
*Bootlin 참조:* [`include/linux/sched.h`](https://elixir.bootlin.com/linux/v7.1.4/source/include/linux/sched.h#L820)

```c
struct task_struct {
    struct thread_info    thread_info;
    unsigned int          flags;         /* 프로세스 상태 플래그 */

    /* 프로세스 식별자 */
    pid_t                 pid;           /* Process ID */
    pid_t                 tgid;          /* Thread Group ID (유저가 보는 PID) */

    /* 메모리 관리 (가상 메모리 맵) */
    struct mm_struct      *mm;           /* 유저 영역 메모리 구조체 */
    struct mm_struct      *active_mm;

    /* 보안 자격 증명 (Credentials) */
    const struct cred __rcu *real_cred;  /* Object의 실제 객관적 자격 증명 */
    const struct cred __rcu *cred;       /* Subject로서 수행할 때의 유효 자격 증명 */

    /* 프로세스 트리 계층 구조 */
    struct task_struct __rcu *real_parent; /* 실제 부모 프로세스 */
    struct task_struct __rcu *parent;      /* 현재 부모 프로세스 */
    struct list_head      children;        /* 자식 프로세스 리스트 Head */
    struct list_head      sibling;         /* 형제 프로세스 리스트 Node */

    /* 열린 파일 시스템 정보 */
    struct files_struct   *files;        /* fd 테이블 정보 */
    /* ... 생략 ... */
};
```

> ### real_cred vs cred 의 차이점
> - real_cred: 프로세스가 생성될 때 부여받은 실제 자격 증명이다. (예: setuid 프로그램 실행 전 원래 유저 ID)  
> - cred: 프로세스가 현재 작업을 수행할 때 행사하는 유효 자격 증명이다.  
> - 보통은 두 포인터가 같은 cred 구조체를 가리키지만, setuid 실행파일(예: sudo, passwd)을 실행하면 cred 포인터만 root의 cred 객체를 가리키도록 전환된다.  

# 2. cred 구조체
cred 구조체는 커널이 파일 접근, 프로세스 신호 전달, 네트워크 바인딩 등 "이 Task가 해당 작업을 수행할 권한이 있는가?"를 검증할 때 참조하는 핵심 객체이다.

### 멤버 변수 필드 (cred.h)
*Bootlin 참조:* [`include/linux/cred.h`](https://elixir.bootlin.com/linux/v7.1.4/source/include/linux/cred.h#L115)

```c
struct cred {
    atomic_t    usage;           /* 참조 카운터 (Reference Count) */

    /* 사용자 및 그룹 식별자 */
    kuid_t      uid;             /* Real User ID */
    kgid_t      gid;             /* Real Group ID */
    kuid_t      suid;            /* Saved User ID */
    kgid_t      sgid;            /* Saved Group ID */
    kuid_t      euid;            /* Effective User ID (접근 제어 판단 기준) */
    kgid_t      egid;            /* Effective Group ID */
    kuid_t      fsuid;           /* File System User ID (파일 접근 검증용) */
    kgid_t      fsgid;           /* File System Group ID */

    /* POSIX Capabilities (세부 권한 분할) */
    kernel_cap_t cap_inheritable;
    kernel_cap_t cap_permitted;   /* 사용 가능한 최대 Capability 집합 */
    kernel_cap_t cap_effective;   /* 현재 활성화된 Capability 집합 */
    kernel_cap_t cap_bset;
    kernel_cap_t cap_ambient;

    /* ... RCU 및 Security Module (SELinux/AppArmor) 생략 ... */
} __randomize_layout;            /* GCC 플러그인에 의해 구조체 멤버 순서가 랜덤화될 수 있음 */
```

# 3. 커널 익스플로잇에서 `cred` 조작 기법
커널 취약점을 확보했을 때, 권한 상승을 이루어내는 대표적인 패턴 3가지이다.

## 1. commit_creds() + prepare_kernel_cred() 함수 호출
가장 고전적이고 확실한 방법이다. 커널 내부 함수를 `ROP/JOP` 또는 실행 흐름 제어로 직접 실행한다.

```c
// prepare_kernel_cred(NULL) 은 root(UID=0) 권한을 가진 새로운 struct cred 객체를 생성하여 반환함
// commit_creds() 는 생성된 cred 객체를 현재 task_struct의 cred 포인터에 덮어씌움
commit_creds(prepare_kernel_cred(NULL));
```

## 2. Direct Cred Overwrite (DKOM - Direct Kernel Object Manipulation)
커널 임의 쓰기(AAW) 주소가 확보된 경우, 현재 프로세스의 `task_struct`를 탐색하거나 `cred` 주소를 Leak한 뒤 `uid`, `gid`, `euid` 필드의 메모리를 직접 0으로 수정한다.
```
[현재 task_struct]
   ├── cred ───> [ struct cred ]
                      ├── uid  : 1000  ──(Overwrite)──> 0 (root)
                      ├── gid  : 1000  ──(Overwrite)──> 0 (root)
                      ├── euid : 1000  ──(Overwrite)──> 0 (root)
                      └── egid : 1000  ──(Overwrite)──> 0 (root)
```

## 3. init_cred 교체 (Cred Pointer Swapping)
- 커널 메모리 상에는 시스템 최초 프로세스인 init의 권한 정보를 담고 있는 전역 변수 `init_cred`가 존재한다.   
- `init_cred`는 기본적으로 `root` 권한(UID=0)을 가지고 있다.  
- 따라서 현재 `task_struct`의 `cred` 및 `real_cred` 포인터가 가리키는 주소를 `init_cred`의 주소로 Overwrite하면 즉시 root 권한을 획득한다.

```c
// 현재 프로세스의 task_struct.cred 포인터를 init_cred의 주소로 덮어쓰기
task->cred = &init_cred;
task->real_cred = &init_cred;
```

# 4. 요약
1. **Task 찾기**: current 매크로나 `init_task` 링크드 리스트를 순회하여 현재 내가 실행 중인 프로세스의 `task_struct` 주소를 확보한다.
2. **cred 포인터 추적**: `task_struct` 내의 `cred` 포인터 오프셋을 계산하여 `cred` 구조체의 위치를 파악한다.
3. **권한 상승 수행**
    * `commit_creds(prepare_kernel_cred(NULL))`을 실행하거나,
    * `cred` 구조체의 `uid`/`euid` 필드를 0으로 직접 패치하거나,
    * `cred` 포인터 자체를 `&init_cred` 주소로 바꾼다.
4. **유저 모드 복귀**: `sysretq` / `swapgs` / `kpti_trampoline` 등을 거쳐 유저 모드로 돌아온 후 system("/bin/sh")를 실행하면 root 쉘을 얻을 수 있다.
