# 04_Text_Manipulation

리눅스의 강력한 텍스트 처리 도구들을 다룬다.
로그 분석, 코드 오디팅, CTF 문제 풀이 시 수많은 데이터 속에서 핵심 정보(Key/Flag)를 추출하는 기술이다.

---

## 1. Grep (Global Regular Expression Print)

**[가장 중요]** 파일 내용이나 파이프라인 출력에서 특정 문자열 패턴을 검색한다.

* **Usage:** `grep [options] "pattern" [file]`

| 옵션 | 설명 | 🔒 Security Use Case |
| :---: | :--- | :--- |
| **`-i`** | 대소문자 무시 | `grep -i "pass" file` (Password, password 모두 찾기) |
| **`-r`** | 재귀적 검색 (하위 폴더 포함) | **소스코드 오디팅 시** `grep -r "TODO" .` 혹은 `grep -r "API_KEY" .` |
| **`-v`** | 매칭되지 **않는** 줄만 출력 | 로그 분석 시 내 IP나 정상 트래픽을 제외할 때 (`grep -v "127.0.0.1"`) |
| **`-n`** | 줄 번호 출력 | 코드 분석 시 몇 번째 줄에 취약한 함수가 있는지 확인. |
| **`-E`** | 확장 정규식 사용 | 복잡한 패턴(IP 주소, 이메일 등) 검색 시. |

> **💡 CTF Tip:** 폴더 어딘가에 숨겨진 플래그를 찾을 때:
> `grep -r "DH{" /challenge/path`

---

## 2. Find (File Search)

파일의 **내용**이 아닌 **속성**(이름, 크기, 권한, 시간)으로 파일을 찾는다.

* **Usage:** `find [path] [expression]`

### 주요 옵션
* **`-name`:** 이름으로 찾기.
    * `find / -name "*.conf"`: 설정 파일 모두 찾기.
* **`-size`:** 크기로 찾기.
    * `find . -size 1033c`: 정확히 1033바이트인 파일 찾기 (CTF 단골 문제).
* **`-perm`:** **[핵심]** 권한으로 찾기.
    * `find / -perm -4000 2>/dev/null`: **시스템 전체에서 SetUID가 걸린 파일 찾기.** (권한 상승 공격의 첫 단계)
* **`-exec`:** 찾은 파일에 대해 명령어 실행.
    * `find . -name "*.log" -exec rm {} \;`: 로그 파일 일괄 삭제.

---

## 3. Extraction (Cut & Awk)

로그 파일이나 `/etc/passwd` 같이 정형화된 데이터에서 **특정 열(Column)**만 뽑아낸다.

### `cut`
구분자(Delimiter)가 명확할 때 사용.
* **Usage:** `cut -d "구분자" -f "필드번호" filename`
* **Example:** `/etc/passwd`에서 사용자명만 뽑기.
    * `cat /etc/passwd | cut -d ":" -f 1`

### `awk`
**[Swiss Army Knife]** 공백 처리가 유동적이거나 복잡한 연산이 필요할 때 사용.
* **Basic Usage:** `awk '{print $1, $3}'` (1번째와 3번째 단어만 출력)
* **Example:** `ps -ef` 결과에서 PID만 추출하여 kill 명령어로 넘길 때.
    * `ps -ef | grep "apache" | awk '{print $2}'`

---

## 4. Replacement (Sed)

**S**tream **Ed**itor. 파일을 열지 않고 내용을 치환하거나 편집한다.

* **Usage:** `sed 's/찾을거/바꿀거/g' filename`
* **CTF/Exploit Use Case:**
    * 대량의 페이로드에서 특정 쉘코드 주소만 일괄 변경할 때.
    * 익스플로잇 스크립트 실행 전, 설정 파일 값을 자동화하여 바꿀 때.
    * **Example:** 파일 내의 모든 `http`를 `https`로 변경.
        * `sed -i 's/http/https/g' config.xml` (`-i`는 원본 파일 덮어쓰기)

---

## 5. Sorting & Statistics (Sort & Uniq)

로그 분석의 꽃. "누가 가장 많이 접속했는가?"를 알아낼 때 쓰는 국룰 조합.

1.  **`sort`:** 데이터를 정렬한다. (기본은 알파벳순)
    * `-n`: 숫자 기준 정렬.
    * `-r`: 내림차순(Reverse) 정렬.
2.  **`uniq`:** **연속된** 중복 라인을 제거한다. (반드시 `sort`와 함께 써야 함)
    * `-c`: 중복된 횟수(Count)를 함께 출력.

### 🚀 Log Analysis Pipeline (Top 5 Attacker IP 찾기)
웹 서버 로그(`access.log`)에서 공격자 IP 순위를 매기는 명령어.

```bash
cat access.log | awk '{print $1}' | sort | uniq -c | sort -nr | head -n 5

```

1. `awk`: IP 주소(1번째 필드)만 추출.
2. `sort`: IP끼리 뭉쳐놓음.
3. `uniq -c`: IP별 접속 횟수 카운트. (출력: `100 192.168.1.1`)
4. `sort -nr`: 횟수(숫자) 기준으로 내림차순 정렬.
5. `head -n 5`: 상위 5개만 출력.

---

## 6. Binary Text (Strings)

바이너리(실행 파일)나 덤프 파일 내에서 **"사람이 읽을 수 있는 문자열"**만 추출한다.

* **Usage:** `strings filename`
* **Security Use Case:**
* 리버스 엔지니어링 전, 하드코딩된 비밀번호나 API Key가 있는지 쓱 훑어볼 때(Quick Look).
* 악성코드 분석 시, 연결하려는 C&C 서버의 IP나 URL을 찾을 때.



