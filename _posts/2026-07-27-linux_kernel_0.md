---
title: "Linux Kernel Basics"
date: 2026-07-27 22:50:00 +0900
categories: [Kernel, Pwnable]
tags: [linux, kernel]
---


# 1. 커널 모드와 유저 모드 (CPU Privilege Levels)
CPU는 실행 권한을 나누어 관리한다.

x86_64 아키텍처에서는 Ring 0 ~ 3으로 구분한다.

```
+--------------------------------------------------------+
| Ring 3 : User Mode (일반 응용 프로그램, 웹 브라우저 등) |
|   - 하드웨어 직접 접근 불가, 제한된 메모리 영역 접근    |
+--------------------------------------------------------+
| Ring 1, 2 : (보통 사용되지 않음 / 하이퍼바이저 등)      |
+--------------------------------------------------------+
| Ring 0 : Kernel Mode (리눅스 커널, 드라이버)            |
|   - 모든 CPU 명령 실행 가능, 모든 메모리/하드웨어 제어  |
+--------------------------------------------------------+
```

- **유저 모드 (Ring 3)**: 일반 프로그램이 실행되는 영역. 디스크에 파일을 쓰거나 네트워크 패킷을 보내는 등 하드웨어 제어가 필요한 경우 커널에 요청을 보낸다.

- **커널 모드 (Ring 0)**: OS의 핵심 코드와 드라이버가 실행되는 최고 권한 영역.



# 2. System Call
유저 공간의 프로그램이 `read`, `write`, `fork`, `brk`, `mmap` 등 커널의 도움이 필요할 때 사용하는 인터페이스.

### 시스템 콜 실행 흐름 (x86_64)
1. **요청 준비**: `RAX` 레지스터에 시스템 콜 번호를 세팅하고, 매개변수는 `RDI`, `RSI`, `RDX`, `R10`, `R8`, `R9` 순으로 설정.
2. **모드 전환**: `syscall` 어셈블리 명령어를 실행한다. CPU는 MSR(Model Specific Register)에 저장된 `LSTAR` 주소를 참조하여 즉시 Ring 3에서 Ring 0으로 전환.
3. **커널 스택 전환 및 핸들러 호출**: 실행 흐름이 커널의 `entry_SYSCALL_64` 핸들러로 이동하며, 유저 레지스터 상태를 커널 스택(`pt_regs`)에 저장.
4. **검증 및 실행**: 커널은 전달받은 인자와 유저 공간 포인터를 검증한 후 시스템 콜 루틴을 수행.
5. **복귀**: 작업 완료 후 `sysret` 또는 `iretq` 명령어를 통해 저장된 유저 레지스터를 복원하고 유저 모드로 복귀.

> **유저** <-> **커널 데이터 교환 함수**  
> SMAP 등 보호 기법으로 인해 커널은 유저 메모리에 직접 접근하지 않고, 검증 루틴이 포함된 API를 사용.
> - `copy_from_user(to, from, n)`: 유저 공간(`from`)의 데이터를 커널 공간(`to`)으로 복사
> - `copy_to_user(to, from, n)`: 커널 공간(`from`)의 데이터를 유저 공간(`to`)으로 복사

# 3. 프로세스와 권한 관리 (task_struct와 cred)
bootlin: [https://elixir.bootlin.com/linux/v7.1.4/source/include/linux/sched.h#L820](https://elixir.bootlin.com/linux/v7.1.4/source/include/linux/sched.h#L820)

커널은 실행 중인 모든 작업(프로세스 및 쓰레드)을 `task_struct`라는 PCB라는 구조체로 관리.

- **task 구분**: 유저 공간에서는 프로세스와 쓰레드를 구분하지만, 커널 내부에서는 모두 스케줄링 단위인 `task_struct` 객체로 동일하게 취급한다.
- **권한 객체 (cred)**: `task_struct` 내부의 `cred` 포인터가 해당 task의 보안 자격 증명(UID, GID, Capabilities 등)을 담고 있다.

```
+-------------------------------------------------------+
|                    task_struct                        |
|                                                       |
|   pid: 1001                                           |
|   tgid: 1001                                          |
|   mm:  0x7fff8000... (유저 메모리 주소)               |
|                                                       |
|   real_cred ------+                                   |
|   cred -----------|---> +---------------------------+ |
|                   |     |       struct cred         | |
|                   +---> |                           | |
|                         |  uid: 1000  (실제 사용자) | |
|                         |  gid: 1000                | |
|                         |  euid: 0    (실행 권한)   | |
|                         |  egid: 0                  | |
|                         |  cap_inheritable          | |
|                         |  cap_permitted            | |
|                         |  cap_effective            | |
|                         +---------------------------+ |
+-------------------------------------------------------+
```

> ### **Privilege Escalation 관점**  
> root의 UID/GID 값은 `0`.  
> 커널 AAW 취약점을 통해 현재 프로세스의 `cred` 구조체 내 UID/EUID 필드를 `0`으로 덮어쓰거나, 커널 내 `commit_creds(prepare_kernel_cred(NULL))` 함수를 호출하여 권한을 상승시키는 것이 커널 익스플로잇의 주요 목표.

# 4. 커널 메모리 관리와 할당자
커널 메모리는 가상 메모리로 관리되며, Page Table을 통해 물리 메모리에 매핑된다.

- **User Space**: `0x0000000000000000 ~ 0x00007FFFFFFFFFFF`
- **Kernel Space**: `0xFFFF888000000000 ~ 0xFFFFFFFFFFFFFFFF`

### 주요 커널 메모리 할당자
- **`kmalloc()`**: 작은 크기의 연속된 메모리를 할당한다. 내부적으로 **SLUB Allocator**를 이용해 미리 확보된 슬랩 캐시(`kmalloc-32`, `kmalloc-64`, `kmalloc-512` 등)에서 메모리를 할당한다.
- **`vmalloc()`**: 물리적으로는 연속되지 않더라도 가상 메모리 상에서 연속된 큰 메모리 영역을 할당한다.
- **`kfree()`**: `kmalloc`으로 할당받은 메모리를 해제한다. (해제 후 포인터를 초기화하지 않으면 Kernel Use-After-Free 취약점 발생)



# 5. 핵심 커널 보호기법
커널 취약점 공격을 방어하기 위해 적용된 주요 보호 기법들이다.


- **KASLR** (Kernel Address Space Layout Randomization): 부팅 시 커널 코드, 데이터, 구조체가 메모리에 로드되는 기본 주소를 임의로 배치한다.
- **SMEP** (Supervisor Mode Execution Prevention): 커널 모드(Ring 0) 상태에서 유저 공간 메모리에 위치한 코드(쉘코드 등)를 실행하지 못하도록 CPU 차원에서 차단한다.
- **SMAP** (Supervisor Mode Access Prevention): 커널 모드 상태에서 명시적인 허용 명령어(`stac`/`clac`) 없이 유저 공간 메모리를 읽거나 쓰지 못하도록 차단한다.
- **KPTI** (Kernel Page Table Isolation): 유저 모드 실행 중에는 커널 Page Table을 매핑에서 제외하여, Meltdown 공격 및 커널 메모리 주소 유출을 방지한다.
- **kstack canary**: 유저 공간의 Stack Canary와 동일하게 커널 스택 오버플로우를 방지하기 위해 스택에 랜덤 값을 삽입한다.
- **kCFI** (Kernel Control Flow Integrity): 간접 호출(Indirect Call) 지점에서 정당한 함수 목표인지 검증하여 ROP/JOP 공격을 방어한다.
