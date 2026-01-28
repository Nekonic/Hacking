# 05_Process_Job

실행 중인 프로그램(Process)을 모니터링하고 제어하는 방법.
특히 리눅스 커널이 프로세스 정보를 노출하는 `/proc` 디렉토리 구조는 **시스템 해킹(Pwnable)과 정보 수집의 핵심**이다.

---

## 1. Monitoring Processes (프로세스 조회)

시스템에서 "누가, 무엇을, 어떻게" 실행 중인지 파악한다.

### `ps` (Process Status)
현재 실행 중인 프로세스의 스냅샷을 보여준다.
* **Usage:**
  * `ps -ef`: System V 스타일. 부모 프로세스(PPID) 확인에 좋음.
  * `ps aux`: BSD 스타일. **CPU/MEM 점유율**과 **상태(STAT)** 확인에 좋음.
* **Key Columns:**
  * **PID (Process ID):** 프로세스 고유 식별 번호. (공격 대상 식별)
  * **UID (User ID):** 프로세스 소유자. (권한 확인: root인가? 일반 유저인가?)
  * **CMD:** 실행된 명령어 및 인자.

### `top` / `htop`
실시간 프로세스 모니터링 도구. (윈도우의 작업 관리자)
* CPU/메모리 점유율이 비정상적으로 높다면 코인 채굴 악성코드(Miner)나 DoS 공격을 의심해볼 수 있다.

---

## 2. Process Control & Signals (제어 및 시그널)

프로세스를 단순히 "죽이는" 것이 아니라, **시그널(Signal)**을 보내 상태를 제어한다.

### `kill`
특정 PID에 시그널을 전송한다. (이름은 kill이지만 종료 외의 신호도 보냄)
* **Usage:** `kill -[Signal] [PID]`

| Signal | 번호 | 설명 | Security Context |
| :---: | :---: | :--- | :--- |
| **SIGINT** | 2 | **Interrupt**. (Ctrl + C) | 키보드로 보내는 중단 요청. |
| **SIGKILL** | 9 | **Force Kill**. | 프로세스가 **무시하거나 막을 수 없는** 강제 종료. 좀비 프로세스 정리 시 사용. |
| **SIGSEGV** | 11 | **Segmentation Fault**. | **[Pwnable]** 메모리 잘못 건드리면 뜨는 오류. 버퍼 오버플로우 성공 여부의 척도. |
| **SIGTERM** | 15 | **Terminate**. (기본값) | "정상적으로 종료해라"라고 요청. (저장할 시간 줌) |

---

## 3. Job Control (백그라운드 작업)

해킹 툴이나 리버스 쉘 리스너(`nc -lvp`)를 백그라운드에서 돌려놓고 다른 작업을 할 때 사용.

* **`&` (Background):** 명령어 뒤에 붙이면 백그라운드에서 실행.
    * `python exploit.py &`
* **`jobs`:** 현재 쉘 세션의 백그라운드 작업 목록 확인.
* **`fg`:** 백그라운드 작업을 포그라운드(화면)로 가져옴.
    * `fg %1` (1번 작업 가져오기)
* **`bg`:** 일시 정지된(Ctrl+Z) 작업을 백그라운드에서 계속 실행.

---

## 4. The `/proc` Filesystem (Pseudo-filesystem)

**[System Hacking의 보물창고]**
리눅스 커널이 메모리에 있는 프로세스 정보를 파일 형태로 보여주는 가상 디렉토리. 디스크에 존재하지 않는다.

### `/proc/[PID]/maps` (Memory Maps)
* **내용:** 프로세스의 메모리 구조(주소 범위, 권한, 매핑된 파일)를 보여줌.
* **Security Use Case:** **[ASLR Bypass]**
    * 바이너리가 실행될 때마다 라이브러리(libc), 스택, 힙의 **Base Address**가 바뀐다.
    * 이 파일을 읽을 수 있다면(LFI 등), 메모리 주소를 유출(Leak)시켜 **ROP(Return Oriented Programming)** 공격을 위한 오프셋을 계산할 수 있다.



### `/proc/[PID]/fd` (File Descriptors)
* **내용:** 해당 프로세스가 열고 있는 파일들의 심볼릭 링크.
* **Security Use Case:**
    * 0(stdin), 1(stdout), 2(stderr) 외에 열려 있는 **소켓 연결**이나 **비밀 파일**을 확인 가능.
    * 레이스 컨디션 공격이나 파일 디스크립터 하이재킹 시 참조.

### `/proc/[PID]/cmdline`
* **내용:** 프로세스 실행 시 입력된 **명령어 인자(Arguments)** 전체.
* **Security Use Case:**
    * 관리자가 실수로 `mysql -u root -pPassword123` 처럼 인자에 비밀번호를 넣고 실행했다면, 이 파일을 통해 평문 비밀번호를 탈취할 수 있다.

### `/proc/[PID]/environ`
* **내용:** 프로세스에 적용된 **환경 변수** 목록.
* **Security Use Case:**
    * API Key나 DB 접속 정보가 환경 변수에 저장된 경우 유출 가능.

### `/proc/self`
* **내용:** 현재 명령어를 실행하고 있는 **자기 자신**의 프로세스 디렉토리로 연결되는 심볼릭 링크.
* **Security Use Case:** PID를 몰라도 공격 가능.
    * 웹 해킹(LFI) 시 `../../proc/self/maps` 등을 요청하여 현재 웹 서버 프로세스의 정보를 캔다.
