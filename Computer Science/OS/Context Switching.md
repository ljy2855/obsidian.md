

### Instruction execution
프로세스가 시작된 이후 CPU가 프로그램을 실행할 때의 순서

1. 프로세스를 위한 메모리 초기화
2. file -> memory code segment load
3. rip를 메모리 code의 entry point로 이동
4. instruction 실행

### PCB (Process Control Block)

운영체제는 여러 프로세스를 동시에 실행해야 하므로, 실행 중인 프로세스를 전환할 때마다 **현재 프로세스의 상태를 저장**해야 한다. 이를 **PCB(Process Control Block)** 에 저장하며, PCB는 **커널 영역의 메모리**에 위치한다.

PCB에는 다음과 같은 정보가 포함된다.

- **프로세스 ID (PID)**: 프로세스 고유 식별자
- **프로세스 상태 (Process State)**: 실행(Running), 대기(Waiting), 준비(Ready) 등
- **CPU 레지스터 정보**: RIP, RSP, RAX 등의 레지스터 값
- **스케줄링 정보**: 우선순위, 타임 슬라이스(Quota)
- **메모리 정보**: 코드, 데이터, 힙, 스택 영역의 위치
- **파일 핸들 및 입출력 상태**

운영체제는 **PCB를 이용해 프로세스의 문맥(Context)을 저장 및 복구**하며, 이 과정이 **Context Switching**이다.


### Timer Interrupt & Context Switching

운영체제는 특정 프로세스가 CPU를 독점하는 것을 방지하기 위해 **타이머 인터럽트(Timer Interrupt)** 를 사용한다.

1. 운영체제는 각 프로세스가 **CPU를 사용할 수 있는 시간(Quota)** 을 설정한다.
2. CPU 내부의 타이머가 일정 시간이 지나면 **Timer Interrupt** 를 발생시킨다.
3. 인터럽트 핸들러가 호출되어 **현재 실행 중인 프로세스의 시간을 체크**한다.
4. 만약 할당된 시간이 초과되었으면, **현재 프로세스를 중단하고 Context Switching을 수행**한다.

### Context switching


![[Pasted image 20250307141016.png]]

1. A 프로세스 실행 중
2. Timer Interrupt 발생 → 커널 모드 진입
3. 현재 A의 레지스터 값을 Kernel Stack에 Push
4. 커널 모드에서 Context Switching Handler 실행
5. A 프로세스의 정보를 PCB에 저장
6. 스케줄러가 다음 실행할 B 프로세스를 결정
7. B의 PCB에서 레지스터 값을 Kernel Stack에 복원 (Pop)
8. B 프로세스의 Kernel Stack에서 User Stack으로 전환
9. 유저 모드로 돌아가 B 프로세스 실행



### 현재의 리눅스 시스템 변경사항

과거의 운영체제와 비교할 때, 현대의 리눅스 시스템은 **더 최적화된 Context Switching 기법과 효율적인 스케줄러**를 사용한다.

https://en.wikipedia.org/wiki/Linux_kernel_version_history

#### **리눅스 2.6 – 선점형 커널 & Lazy FPU Switching**

- **Preemptible Kernel 도입** → 커널 모드에서도 Context Switching 가능
- **Lazy FPU Context Switching** → 부동소수점 연산이 필요할 때만 FPU 상태 저장/복원 (문맥 전환 오버헤드 감소)


#### **리눅스 3.x – RCU & NUMA 최적화**

- **RCU(Read-Copy-Update) 최적화** → 다중 코어 환경에서 커널 데이터 보호 시 Context Switching 오버헤드 감소
- **NUMA 최적화** → 멀티코어 CPU의 메모리 접근 속도 개선 (TLB 미스 감소)


#### **리눅스 4.x – Lazy TLB Switching & Zero-Copy 최적화**

- **Lazy TLB Switching** → 불필요한 TLB 플러시 최소화하여 문맥 전환 속도 향상
- **Zero-Copy 입출력 최적화** → 파일 I/O 시 Context Switching 감소 (`sendfile()`, `mmap()` 활용)


#### **리눅스 5.x – 커널 스레드 최적화 & Preempt-RT 개선**

- **커널 스레드와 유저 스레드 전환 최적화** → 커널 모드에서 문맥 전환 속도 향상
- **Preempt-RT (Real-time Preempt) 개선** → 실시간 환경에서도 낮은 문맥 전환 비용 유지


#### **리눅스 6.x – Futex2 & MGLRU 도입**

- **Futex2 시스템 콜 도입** → 멀티스레드 환경에서 동기화 오버헤드 감소
- **MGLRU (Multi-Gen LRU) 메모리 관리 기법 적용** → TLB 캐시 미스를 줄여 Context Switching 성능 향상

