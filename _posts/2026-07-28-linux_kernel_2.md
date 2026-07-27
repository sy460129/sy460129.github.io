---
title: "Linux Kernel: 주요 API"
date: 2026-07-28 00:00:00 +0900
categories: [Kernel, Pwnable]
tags: [linux, kernel]
---

# 1. 사용자-커널 공간 데이터 복사 API (메모리 손상 및 정보 유출)
User Space와 Kernel Space 간에 데이터를 주고받을 때 입력값 검증이 누락되면 버퍼 오버플로우나 커널 메모리 유출이 발생한다.

### 1) `copy_from_user(to, from, n)`
사용자 공간의 데이터를 커널 버퍼로 복사한다.
- `n` 값이 커널 버퍼 크기보다 크거나 적절히 검증되지 않으면, 커널 스택이나 힙 영역에 **Kernel Buffer Overflow**가 발생한다.

### 2) `copy_to_user(to, from, n)`
커널 공간의 데이터를 사용자 공간으로 복사한다.
- 커널 내부의 초기화되지 않은 메모리나 **Infoleak**의 단골 API이다. 
- KASLR이나 Stack Canary를 우회할 때 주로 사용된다.

### 3) `__copy_from_user()` / `__copy_to_user()`
주소 범위 유효성 검사 매크로인 `access_ok()`를 생략하는 오버헤드 단축 버전의 API이다.
- 개발자가 사전에 `access_ok()` 검증을 누락한 채 이 API를 오용할 경우, SMAP 등 커널 방어기법을 우회하고 커널 메모리를 직접 읽고 쓰는 치명적인 취약점으로 이어진다.

# 2. 커널 동적 메모리 할당 API (힙 취약점)
리눅스 커널의 **SLUB/SLAB Allocator** 환경에서 발생하는 힙 오버플로우 및 Use-After-Free(UAF) 취약점과 직접적으로 연결되는 API이다.

```
+-----------------------------------------------------------+
|                    kmalloc / kzalloc                      |
+-----------------------------------------------------------+
|
v
+-----------------------------------------------------------+
|                      SLUB Allocator                       |
|   +------------------+  +-------------------+             |
|   | kmalloc-64 Cache |  | kmalloc-512 Cache |   ...       |
|   +------------------+  +-------------------+             |
+-----------------------------------------------------------+
| (Slab allocation)
v
+-----------------------------------------------------------+
|                      Physical Pages                       |
|   [Chunk][Chunk][Chunk] ...                               |
+-----------------------------------------------------------+
```

### 1) `kmalloc(size, flags)` / `kzalloc(size, flags)`
커널 내부에서 동적 메모리를 할당한다. (`kzalloc`은 할당 후 0으로 초기화)
- `size`를 계산하는 과정에서 **Integer Overflow**가 발생하면 실제 필요한 크기보다 작게 메모리가 할당된다. 이후 데이터를 복사할 때 **Kernel Heap Overflow**로 이어진다.

### 2) `kfree(ptr)`
`kmalloc`으로 할당받은 메모리를 해제한다.
- 메모리를 해제한 후 포인터를 `NULL`로 초기화하지 않거나 로직 결함이 존재하는 경우, 해제된 메모리를 재참조하는 **Use-After-Free** 또는 **Double Free** 취약점이 발생한다.

# 3. 권한 상승 및 크리덴셜 관리 API (LPE 취약점)
공격자가 커널 메모리 조작(AAW)이나 실행 흐름 제어(ROP)에 성공한 뒤, 최종적으로 현재 프로세스의 권한을 root로 바꾸기 위해 타겟팅하는 API이다.

### 1) `prepare_kernel_cred(daemon)`
원하는 신원 정보의 `struct cred` 구조체를 생성하는 함수이다.
- **v6.2 이전**: `prepare_kernel_cred(NULL)` (또는 인자로 `0`) 형태로 호출 시, 커널이 자동으로 **root(UID=0) 권한의 자격 증명 객체**를 반환했다.
```c
/* Linux v6.2 이전 커널 소스코드 */
struct cred *prepare_kernel_cred(struct task_struct *daemon)
{
    const struct cred *old;
    struct cred *new;

    new = kmem_cache_alloc(cred_jar, GFP_KERNEL);
    if (!new)
        return NULL;

    /*
     * daemon 인자가 NULL(0)이면 root 권한 객체인 &init_cred를 가져옴!
     */
    if (daemon)
        old = get_task_cred(daemon);
    else
        old = get_cred(&init_cred); // <--- 공격자가 인자로 NULL(0)을 주면 여기서 root cred가 됨

    validate_creds(old);
    *new = *old;

    /* ... 생략 ... */
    return new;
}
```

- **v6.2 이후**: Hardening 패치로 인자에 `NULL` 전달이 차단되었다. 대신 root 권한으로 실행 중인 최상위 프로세스의 심볼인 **`&init_task`**를 넘겨 `prepare_kernel_cred(&init_task)` 형태로 사용한다.
```c
/* Linux v6.2 이후 패치된 커널 소스코드 */
struct cred *prepare_kernel_cred(struct task_struct *daemon)
{
    const struct cred *old;
    struct cred *new;

    /*
     * 이후 패치
     * daemon이 NULL이면 더 이상 init_cred를 가져오지 않고,
     * WARN_ON_ONCE를 발생시키거나 바로 return NULL 처리
     */
    if (WARN_ON_ONCE(!daemon))
        return NULL;

    new = kmem_cache_alloc(cred_jar, GFP_KERNEL);
    if (!new)
        return NULL;

    /* 무조건 넘겨받은 daemon 프로세스의 cred를 기반으로 작성 */
    old = get_task_cred(daemon);

    validate_creds(old);
    *new = *old;

    /* ... 생략 ... */
    return new;
}
```

> - v6.2 이전: ROP 체인을 구성할 때 RDI = 0으로 만들고 `prepare_kernel_cred(0)`를 호출하면 `init_cred(root)` 정보가 들어간 객체가 반환되어 `commit_creds()`에 그대로 넘겨 권한을 상승시킴.  
> - v6.2 이후: `prepare_kernel_cred(0)` 호출 시 NULL을 반환하여 익스플로잇이 실패함. 따라서 항상 `root` 권한을 가지고 있는 최상위 프로세스 주소인 `&init_task`를 찾아 `prepare_kernel_cred(&init_task)`를 호출하도록 대응함.  

### 2) `commit_creds(new_cred)`
인자로 들어온 `cred` 구조체의 자격 증명을 현재 프로세스의 신원(`current->cred`)으로 적용하는 함수이다.  
- 공격 코드가 커널 내부에서 `commit_creds(prepare_kernel_cred(&init_task))`를 실행하게 만들면, 프로세스가 즉시 root 권한으로 승격되며 LPE가 완성된다.

---

# 4. 요약: 취약점 분류 및 공격 흐름

| 분류 | 관련 API | 발생 가능한 취약점 / 익스플로잇 연관성 |
|---|---|---|
| **메모리 복사** | `copy_from_user`<br>`copy_to_user`<br>`__copy_from/to_user` | Buffer Overflow, Infoleak (KASLR/Canary Leak), SMAP Bypass |
| **동적 할당** | `kmalloc` / `kzalloc`<br>`kfree` | Integer Overflow ➔ Heap Overflow, Use-After-Free (UAF), Double Free |
| **권한 상승** | `prepare_kernel_cred`<br>`commit_creds` | Root Credential 생성 및 프로세스 권한 승격 (LPE 최종 단계) |
