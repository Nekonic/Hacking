# 02_File_System

리눅스 파일 시스템의 구조(FHS)와 동작 원리를 다룬다.
해킹 시 공격 표면(Attack Surface)이 되는 주요 디렉토리와, 시스템 해킹(Pwnable)의 핵심인 파일 디스크립터(FD) 개념을 포함한다.

---

## 1. Linux Directory Structure (FHS)

Filesystem Hierarchy Standard(FHS)에 따른 주요 디렉토리별 용도와 보안 관점의 중요성.

| 경로 | 설명 | 🔒 Security Perspective (Hacker's View) |
| :--- | :--- | :--- |
| **`/` (Root)** | 최상위 디렉토리. | 모든 경로의 시작점. |
| **`/bin`, `/usr/bin`** | 기본 실행 파일(명령어) 모음. (`ls`, `cat`, `bash` 등) | 공격 시 쉘(`/bin/sh`)을 실행하거나 시스템 도구를 활용(LOLBins)할 때 참조. |
| **`/sbin`, `/usr/sbin`** | 시스템 관리자용 실행 파일. (`iptables`, `reboot` 등) | 루트 권한이 있어야 실행 가능한 중요한 도구들. |
| **`/etc`** | 시스템 설정 파일 모음. | **[Target 1순위]** `/etc/passwd`(사용자 목록), `/etc/shadow`(패스워드 해시), `/etc/ssh/sshd_config` 등 정보 유출의 핵심. |
| **`/home`** | 일반 사용자들의 홈 디렉토리. | 웹 해킹 시 상위 경로 탐색(`../`)으로 다른 유저의 파일을 훔쳐볼 때 접근. |
| **`/root`** | `root` 사용자의 홈 디렉토리. | **[Final Goal]** 시스템 장악의 증거(Flag)가 보통 여기에 있음. 일반 유저는 접근 불가. |
| **`/tmp`** | 임시 파일 저장소. 누구나 쓰기(Write) 가능. | **[Staging Area]** 익스플로잇 코드나 악성 스크립트를 올리고 실행하는 공간으로 자주 악용됨. Race Condition 취약점의 주 무대. |
| **`/var`** | 가변 데이터 저장소 (로그, 웹 문서 등). | `/var/log` (침투 흔적 지우기 대상), `/var/www/html` (웹쉘 업로드 경로). |
| **`/proc`** | 프로세스 정보가 파일로 매핑된 가상 파일 시스템. | **[Info Disclosure]** 메모리 구조, 실행 인자 등을 확인하여 ASLR 우회나 정보 수집에 활용. |
| **`/dev`** | 하드웨어 장치 파일. | `/dev/null` (출력 버리기), `/dev/urandom` (암호학적 난수 생성) 등. |

---

## 2. Everything is a File (파일의 종류)

리눅스에서는 텍스트뿐만 아니라 장치, 소켓, 파이프도 파일로 취급한다. `ls -l`의 맨 앞글자로 구별한다.

* **`-` (Regular File):** 일반적인 데이터 파일 (텍스트, 이미지, 바이너리 등).
* **`d` (Directory):** 디렉토리.
* **`l` (Symbolic Link):** 윈도우의 '바로가기'와 유사. 원본을 가리키는 포인터.
* **`c` / `b` (Device File):** 문자(Character) 장치 / 블록(Block) 장치.
* **`s` (Socket):** 프로세스 간 통신(IPC) 혹은 네트워크 통신을 위한 엔드포인트.
* **`p` (Named Pipe):** 프로세스 간 데이터 통로.

> **💡 Security Tip:** 웹 해킹(LFI) 공격 시, 일반 파일뿐만 아니라 `/proc/self/environ` 같은 가상 파일이나 로그 파일을 include 하여 RCE(원격 코드 실행)로 연계하는 기법이 있다.

---

## 3. Inode (Index Node)

파일의 실제 데이터와 메타데이터(속성)는 분리되어 저장된다.

* **Inode:** 파일의 권한, 소유자, 크기, 시간 정보, **데이터 블록의 위치** 등을 담고 있는 자료구조. (파일 이름은 디렉토리 엔트리에 저장됨)
* **Hard Link vs Symbolic Link:**
    * **Hard Link:** 동일한 Inode를 가리키는 다른 이름. 원본을 지워도 데이터는 살아있음.
    * **Symbolic Link:** 원본 파일의 *경로*를 담고 있는 별도의 파일. 원본을 지우면 깨짐(Broken link).

> **💡 Forensics Tip:** 파일을 `rm`으로 삭제해도 Inode와 데이터 블록의 연결만 끊어질 뿐, 실제 데이터는 덮어쓰기 전까지 디스크에 남아있다. 이것이 삭제된 파일 복구의 원리이다.

---

## 4. File Descriptors (FD)

**[Binary Exploitation 핵심]**
프로세스가 파일(혹은 소켓, 파이프)에 접근할 때 사용하는 추상적인 식별자(정수).

| FD 번호 | 명칭 | 설명 |
| :---: | :--- | :--- |
| **0** | **STDIN** (Standard Input) | 표준 입력 (키보드 등) |
| **1** | **STDOUT** (Standard Output) | 표준 출력 (터미널 화면 등) |
| **2** | **STDERR** (Standard Error) | 표준 에러 출력 |

* 리눅스에서 파일을 `open()` 하면 3번부터 순차적으로 FD가 할당된다.
* **Pwnable 활용:** * 쉘코드 작성 시 `execve("/bin/sh")`를 실행하기 전에, 소켓 연결을 유지하기 위해 `dup2(socket_fd, 0)`, `dup2(socket_fd, 1)` 등을 사용하여 입출력을 공격자의 연결로 재지정(Redirection)한다.
    * `close(1)` 등으로 출력을 닫아버린 문제(Blind)를 해결할 때 FD 개념이 필수적이다.

---

## 5. File Timestamps (MAC Time)

포렌식 분석 시 타임라인 재구성의 핵심 요소.

* **mtime (Modification):** 파일의 **내용**이 변경된 시간.
* **atime (Access):** 파일을 **읽거나 실행**한 시간. (성능상 noatime 옵션을 쓰기도 함)
* **ctime (Change):** 파일의 **메타데이터(권한, 소유자 등)**가 변경된 시간.

> **⚠️ Caution:** `touch` 명령어나 조작 도구를 통해 타임스탬프는 위변조가 가능하다(Anti-Forensics). 따라서 타임스탬프만 믿어서는 안 되며, 로그나 파일 시스템 저널링 등 교차 검증이 필요하다.
