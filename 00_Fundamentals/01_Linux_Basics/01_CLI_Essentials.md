# 01_CLI_Essentials

리눅스 터미널(Shell) 사용의 가장 기초가 되는 명령어와 단축키 모음.
단순 파일 조작뿐만 아니라, 해킹/CTF 상황에서 효율적으로 움직이기 위한 필수 커맨드를 포함한다.

---

## 1. Navigation (이동 및 탐색)

가장 기본이 되는 디렉토리 이동 및 확인 명령어.

### `pwd` (Print Working Directory)
현재 내가 위치한 경로를 절대 경로로 출력한다.
* **Usage:** `pwd`

### `ls` (List)
현재 디렉토리의 파일 목록을 출력한다.
* **Usage:**
  * `ls -l`: 자세한 정보(권한, 소유자, 크기, 수정일) 포함 출력.
  * `ls -a`: 숨김 파일(`.file` 등)을 포함하여 출력.
  * `ls -al`: 숨김 파일을 포함하여 자세히 출력. **(가장 많이 사용)**
  * `ls -R`: 하위 디렉토리까지 재귀적으로 출력.
> **💡 Security Tip:** CTF나 침투 시 `ls -al`을 생활화해야 한다. `.git`, `.ssh`, `.bash_history` 등 숨겨진 중요 파일을 놓치지 말자.

### `cd` (Change Directory)
디렉토리를 이동한다.
* **Usage:**
  * `cd /path/to/dir`: 특정 경로로 이동.
  * `cd ~`: 홈 디렉토리로 이동.
  * `cd -`: **직전 디렉토리로 이동.** (실수로 이동했을 때 유용)
  * `cd ..`: 상위 디렉토리로 이동.

---

## 2. File Operations (파일 조작)

파일 및 디렉토리를 생성, 복사, 이동, 삭제하는 명령어.

### `touch`
빈 파일을 생성하거나, 파일의 타임스탬프(수정 시간)를 갱신한다.
* **Usage:** `touch filename`

### `mkdir` (Make Directory)
디렉토리를 생성한다.
* **Usage:**
  * `mkdir dirname`
  * `mkdir -p dir1/dir2/dir3`: 하위 디렉토리까지 한 번에 생성.

### `cp` (Copy)
파일이나 디렉토리를 복사한다.
* **Usage:**
  * `cp source_file dest_file`
  * `cp -r source_dir dest_dir`: 디렉토리를 통째로 복사 (Recursive).

### `mv` (Move)
파일을 이동하거나 **이름을 변경**한다.
* **Usage:**
  * `mv old_name new_name`: 이름 변경.
  * `mv file /path/to/dest/`: 경로 이동.

### `rm` (Remove)
파일이나 디렉토리를 삭제한다. **(복구 불가능 주의)**
* **Usage:**
  * `rm filename`
  * `rm -rf dirname`: 디렉토리 강제 삭제 (Recursive + Force).
> **⚠️ Caution:** `rm -rf /` 같은 실수를 하지 않도록 항상 경로를 확인할 것.

---

## 3. Viewing Content (내용 확인)

파일을 열지 않고 터미널에서 내용을 확인하는 명령어.

### `cat` (Concatenate)
파일 내용을 터미널에 전체 출력한다. 짧은 파일 볼 때 유용.
* **Usage:** `cat filename`

### `head` / `tail`
파일의 앞부분(head)이나 뒷부분(tail)만 출력한다.
* **Usage:**
  * `head -n 5 file`: 위에서 5줄만 출력.
  * `tail -n 20 file`: 아래서 20줄만 출력.
  * `tail -f access.log`: **파일 내용이 추가될 때마다 실시간으로 출력.** (로그 분석 시 필수)

### `less`
긴 파일을 페이지 단위로 끊어서 보여준다. (`more`보다 기능이 많음)
* **Usage:** `less filename`
* **Shortcuts:**
  * `Enter`: 한 줄 내리기
  * `Space`: 한 페이지 내리기
  * `/keyword`: 검색 (n으로 다음 찾기)
  * `q`: 종료

---

## 4. Documentation (도움말)

모르는 명령어나 옵션을 확인할 때 사용. 구글링보다 빠를 때가 많다.

### `man` (Manual)
명령어의 상세 매뉴얼을 보여준다.
* **Usage:** `man ls`, `man gcc`
> **💡 Security Tip:** 워게임(System Hacking) 문제에서 특정 함수의 취약점이나 쉘코드 관련 정보를 찾을 때 `man 2 read`, `man 3 system` 처럼 섹션 번호를 활용한다.

### `--help`
간단한 사용법과 옵션 목록을 출력한다.
* **Usage:** `command --help` (예: `python3 --help`)

---

## 5. Essential Shortcuts (필수 단축키)

해킹은 스피드다. 마우스 대신 키보드로 해결하는 습관을 기르자.

| 단축키 | 설명 | 활용 예시 |
| :--- | :--- | :--- |
| **Tab** | 자동 완성 | 파일명/명령어가 길 때 앞글자만 치고 탭. 오타 방지 필수. |
| **Ctrl + C** | 프로세스 강제 종료 (SIGINT) | 잘못된 스크립트 실행 중지. |
| **Ctrl + Z** | 프로세스 일시 정지 (Background) | 실행 중인 작업을 백그라운드로 보낼 때. |
| **Ctrl + L** | 화면 지우기 (clear) | 터미널이 지저분할 때. |
| **Ctrl + R** | 명령어 히스토리 검색 | "아까 썼던 그 긴 명령어 뭐였지?" 할 때 유용. |
| **Ctrl + A / E** | 라인 처음 / 끝으로 이동 | 긴 페이로드를 수정할 때 커서 이동 시간 단축. |
| **Ctrl + U / K** | 커서 기준 앞 / 뒤 내용 삭제 | 명령어 전체를 지우거나 일부 수정할 때. |
