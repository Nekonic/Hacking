# 01_Encapsulation_Headers

데이터가 네트워크를 통해 전송되기 위해 포장(Encapsulation)되는 과정과, 각 헤더(Header)의 구조적 취약점을 다룬다.
해커에게 헤더는 단순한 배송 정보가 아니라, **조작 가능한 공격 벡터(Attack Vector)**이자 **정보 은닉의 공간**이다.

---

## 1. The Protocol Stack (Data Flow)

현대 네트워크는 **TCP/IP 4계층** 모델을 따른다. 데이터는 내려가면서 헤더가 붙고, 올라가면서 떨어진다.

| 계층 (Layer) | 데이터 단위 (PDU) | 핵심 역할 | 주요 헤더 정보 |
| :--- | :--- | :--- | :--- |
| **4. Application** | **Data / Stream** | 응용 프로그램 데이터 | HTTP, SSH, FTP 데이터 자체 |
| **3. Transport** | **Segment** (TCP) / **Datagram** (UDP) | 프로세스 식별, 신뢰성 | Port Number, Sequence Num |
| **2. Internet** | **Packet** | 경로 지정 (Routing) | Source/Dest IP, TTL |
| **1. Network Interface** | **Frame** | 물리적 인접 장비 통신 | Source/Dest MAC |

> **💡 Security Insight:** 방화벽이나 IDS(침입 탐지 시스템)는 주로 3~4계층 정보를 보고 차단한다. 
> 웹 해킹(SQLi, XSS)은 4계층(Application) 페이로드에 숨어있기 때문에, 이를 막으려면 WAF(Web Application Firewall)가 필요하다.

---

## 2. IP Header Structure (IPv4)

패킷의 주민등록증. C언어의 `<netinet/ip.h>` 구조체와 매핑해서 이해하면 좋다.

### Key Fields for Attackers
1.  **Source IP Address (32bit):**
    * **역할:** 보낸 사람의 주소.
    * **Attack:** **IP Spoofing**. UDP와 같이 비연결형 프로토콜이나, 단순히 서버를 다운시키는 DoS 공격(SYN Flood)에서는 이 주소를 랜덤으로 변조하여 추적을 피한다.
2.  **TTL (Time To Live, 8bit):**
    * **역할:** 패킷이 라우터를 거칠 때마다 1씩 감소. 0이 되면 폐기(Loop 방지).
    * **Attack:** **OS Fingerprinting**. 운영체제마다 기본 TTL 값이 다르다. (Linux=64, Windows=128 등). 해커는 응답 패킷의 TTL만 보고도 타겟 서버의 OS를 추측할 수 있다.
3.  **Identification (16bit) & Flags (3bit) & Fragment Offset (13bit):**
    * **역할:** 패킷 단편화(Fragmentation) 및 재조립에 사용.
    * **Attack:** **Fragmentation Attack**. (아래 4번 항목 참조)

---

## 3. TCP/UDP Header

### TCP Header (20 Bytes + Option)
* **Source/Dest Port:** 서비스 식별. (80=Web, 22=SSH)
* **Sequence/Acknowledgment Number:** 데이터 순서 맞추기.
    * **Attack:** **TCP Session Hijacking**. 통신 중인 두 PC 사이에서 Seq 번호를 예측해 끼어들면, 인증 없이 세션을 탈취할 수 있다.
* **Flags (Control Bits):** SYN, ACK, FIN, RST 등. (스캔 공격의 핵심. 다음 챕터에서 상세 기술)

### UDP Header (8 Bytes)
* 매우 단순함 (Source Port, Dest Port, Length, Checksum).
* **Attack:** Handshake 과정이 없으므로 **소스 IP를 속이기 매우 쉽다.**
    * **DNS Amplification Attack:** 소스 IP를 피해자 IP로 속이고, DNS 서버에 엄청 큰 응답을 요청하여 피해자에게 트래픽 폭탄을 던짐.

---

## 4. MTU & Fragmentation (Firewall Evasion)

보안 장비를 우회하기 위한 고전적이지만 강력한 기법.

### 기본 개념
* **MTU (Maximum Transmission Unit):** 한 번에 보낼 수 있는 최대 데이터 크기. (이더넷 기본값: **1500 Bytes**)
* **Fragmentation (단편화):** 데이터가 MTU보다 크면, IP 계층에서 여러 개의 패킷으로 쪼갠다. 수신 측에서 `Identification` 번호를 보고 다시 조립한다.

### ⚔️ 공격 기법 (Attack Vector)
1.  **IDS/IPS 우회 (Evasion):**
    * 악성코드 시그니처가 "GET /etc/passwd"라고 가정하자.
    * 해커가 이 패킷을 아주 잘게 쪼개서 보낸다.
        * Fragment 1: `GET /e`
        * Fragment 2: `tc/pas`
        * Fragment 3: `swd`
    * 단순한 IDS는 조각난 패킷 각각은 정상으로 판단하여 통과시킨다. 수신 측 서버에서 재조립될 때 공격이 성공한다.
2.  **Teardrop / Bonk / Boink (DoS):**
    * 패킷을 쪼갤 때 `Offset` 값을 조작하여, 재조립 시 데이터가 **서로 겹치거나(Overlap) 빈 공간이 생기게** 만든다.
    * 과거의 OS들은 이를 처리하다가 커널 패닉(블루스크린)을 일으키며 뻗어버렸다. (최신 OS는 패치됨)

---

## 5. Encapsulation Abuse (Tunneling)

상위 계층의 데이터를 하위 계층에 숨기거나, 허용된 프로토콜 안에 금지된 프로토콜을 숨기는 기술.

### Tunneling Examples
* **SSH over DNS (DNS Tunneling):**
    * 방화벽이 인터넷(80, 443)은 막고 DNS(53)만 열어둔 경우.
    * 해커는 데이터를 인코딩해서 DNS 쿼리(`ABCD.hacker.com`)의 서브도메인 부분에 넣어서 보낸다.
    * 방화벽은 이를 단순 DNS 질의로 착각하고 통과시킨다.
* **ICMP Tunneling:**
    * `ping` (ICMP Echo Request) 패킷의 데이터 영역(Payload)에 실제 명령어나 데이터를 숨겨서 통신한다.

> **💡 Practical Tip:** 와이어샤크로 패킷을 볼 때, **"헤더에 정의된 표준 크기보다 데이터가 비정상적으로 큰지"** 확인해야 한다. 예를 들어, 내용이 꽉 찬 ICMP 패킷이나 DNS 질의는 터널링 공격일 가능성이 높다.
