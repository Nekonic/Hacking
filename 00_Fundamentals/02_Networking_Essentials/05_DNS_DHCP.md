# 05_DNS_DHCP

인터넷의 전화번호부인 DNS와, IP 주소를 자동 할당해주는 DHCP를 다룬다.
이들은 기본적으로 **"신뢰(Trust)"**를 바탕으로 설계되었기에, 인증 절차가 미흡하여 스푸핑(Spoofing) 공격에 매우 취약하다.

---

## 1. DNS (Domain Name System)

사람이 읽을 수 있는 도메인(`www.google.com`)을 컴퓨터가 이해하는 IP 주소(`142.250.x.x`)로 변환한다.

### Resolution Process (계층 구조)
내가 브라우저에 주소를 쳤을 때 일어나는 일.
1.  **Local Cache & `/etc/hosts`:** 내 PC에 저장된 기록 확인. **(해커가 피싱 사이트로 유도할 때 가장 먼저 건드리는 파일)**
2.  **Recursive DNS (ISP):** KT/SKT/Google(`8.8.8.8`) DNS 서버에 물어봄.
3.  **Root Hints (.)**: 전 세계 최상위 루트 서버에게 "com 관리자 누구야?" 물어봄.
4.  **TLD (.com)**: "google.com 관리자 누구야?" 물어봄.
5.  **Authoritative (ns1.google.com)**: "www.google.com의 진짜 IP 줘." -> **최종 응답**.

### DNS Record Types
* **A:** IPv4 주소.
* **AAAA:** IPv6 주소.
* **CNAME:** 별칭(Canonical Name). (예: `www.naver.com` -> `www.g.naver.com`)
* **TXT:** 텍스트 정보. **(SPF 스팸 방지 설정이나 해커의 C2 서버 통신용으로 악용됨)**
* **PTR:** IP 주소로 도메인을 찾음 (Reverse DNS).

---

## 2. DNS Attacks (Attack Vectors)

### 1) DNS Cache Poisoning (Spoofing)
* **원리:** Recursive DNS 서버가 진짜 응답을 받기 전에, 해커가 조작된 IP(가짜 사이트)를 담은 응답을 먼저 보내 캐시(Cache)에 저장시킨다.
* **결과:** 해당 DNS 서버를 쓰는 모든 사용자가 정상 도메인을 쳐도 해커의 피싱 사이트로 접속하게 된다.

### 2) Zone Transfer (AXFR) - [Information Leakage]
* **개념:** 메인 DNS 서버(Primary)가 백업 서버(Secondary)에게 도메인 정보를 동기화하는 기능.
* **취약점:** 관리자가 권한 설정을 안 하면, **누구나 이 요청을 보내 해당 도메인의 모든 서브도메인 목록(Map)을 통째로 털어갈 수 있다.**
    * `dig axfr @ns.target.com target.com`
    * 숨겨진 `dev.target.com`, `admin.target.com` 등의 내부 서버 주소가 노출된다.

### 3) DNS Rebinding
* **[Advanced Web Hacking]**
* 브라우저의 **SOP(Same Origin Policy)** 보안 정책을 우회하여, 외부 웹페이지가 로컬 네트워크(`127.0.0.1` 등)의 장비를 제어하게 만드는 고난도 기법.
* DNS 응답의 TTL을 아주 짧게 설정하여, 첫 번째 접속은 해커 서버로, 두 번째 접속(AJAX 요청 등)은 피해자의 내부망 IP로 연결되게 조작한다.

---

## 3. DHCP (Dynamic Host Configuration Protocol)

클라이언트가 네트워크에 접속할 때 IP, 서브넷 마스크, 게이트웨이, DNS 주소를 자동으로 할당받는 과정. **UDP**를 사용한다.

### DORA Process (4단계)
1.  **D**iscover (Client -> Broadcast): "DHCP 서버님 계세요? IP 좀 주세요!"
2.  **O**ffer (Server -> Client): "여기 `192.168.0.10` 어때?"
3.  **R**equest (Client -> Broadcast): "좋아요! 저거 쓸게요."
4.  **A**cknowledge (Server -> Client): "오케이, 등록 완료. 임대 시간은 2시간이야."

> **💡 Practical Tip:** 라즈베리파이로 홈랩을 구축할 때, 서버 IP가 자꾸 바뀌면 곤란하다. 공유기 설정에서 **DHCP Static Lease(고정 할당)** 기능을 써서 MAC 주소와 IP를 매핑해두면 편하다.

---

## 4. DHCP Attacks

LAN 환경에서 공격자가 네트워크의 주도권을 잡기 위해 사용한다.

### 1) DHCP Starvation (고갈 공격)
* **원리:** 공격자가 가짜 MAC 주소를 무한대로 생성하여 `DHCP Discover` 패킷을 쏟아붓는다.
* **결과:** DHCP 서버의 할당 가능한 **IP 풀(Pool)이 모두 소진**된다. 이후 정상적인 사용자는 IP를 받지 못해 인터넷이 끊긴다(DoS).

### 2) Rogue DHCP Server (가짜 서버)
* **[MITM의 시작]** Starvation 공격으로 진짜 DHCP 서버를 바보로 만든 뒤, **공격자가 가짜 DHCP 서버를 켠다.**
* 사용자가 IP를 요청하면 공격자가 대신 응답하면서 악의적인 설정을 내려준다.
    * **Gateway:** 공격자의 PC IP로 설정 -> **모든 트래픽이 공격자를 경유함 (Sniffing).**
    * **DNS:** 공격자의 가짜 DNS IP로 설정 -> **피싱 사이트로 유도.**

---

## 5. Etc (Local Config)

### `/etc/hosts` File
* **OS:** Linux/Mac (`/etc/hosts`), Windows (`C:\Windows\System32\drivers\etc\hosts`)
* DNS 서버에 물어보기 전에 **가장 먼저(Priority 1순위)** 참조하는 파일.
* **Security Use Case:**
    * **악성코드:** 백신 업데이트 사이트 주소(`update.antivirus.com`)를 `127.0.0.1`로 설정하여 업데이트를 차단한다.
    * **모의해킹/CTF:** 타겟 서버가 가상 호스트(Virtual Host) 설정이 되어 있어 IP만으로는 접속이 안 될 때, 이 파일에 `<Target_IP> <Target_Domain>`을 등록하여 접속한다.
