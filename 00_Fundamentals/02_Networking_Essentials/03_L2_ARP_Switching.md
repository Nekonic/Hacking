# 03_L2_ARP_Switching

이더넷(Ethernet) 환경에서 장비들이 서로를 식별하는 MAC 주소와, 이를 IP 주소와 연결해주는 ARP 프로토콜을 다룬다.
L2 영역은 **"상호 신뢰(Mutual Trust)"**를 기반으로 설계되었기에, 인증 절차가 없어 **LAN 해킹(MITM, Sniffing)**에 가장 취약하다.

---

## 1. MAC Address & The Switch

물리 계층과 데이터 계층의 연결 고리.

### MAC Address (Media Access Control)
* **구조:** 48비트 (6바이트). 예: `00:1A:2B:3C:4D:5E`
    * 앞 3바이트(OUI): 제조사 식별 코드 (Samsung, Intel 등).
    * 뒤 3바이트: NIC 고유 번호.
* **Security Context (MAC Spoofing):**
    * MAC 주소는 하드웨어에 박혀있지만, **OS 상에서 소프트웨어적으로 변경 가능하다.**
    * **목적:**
        1.  **MAC 필터링 우회:** 공유기에서 특정 기기만 허용할 때, 허용된 기기의 MAC으로 위장.
        2.  **추적 회피:** 공용 와이파이 사용 시 내 진짜 기기 정보 숨기기.

### Switch Operation (Learning & Forwarding)
* **Hub vs Switch:**
    * **Hub:** 들어온 데이터를 모든 포트에 뿌린다. (Dummy). -> **누구나 남의 패킷을 도청 가능.**
    * **Switch:** "누가 어느 포트에 있는지" 학습하여 해당 포트로만 보낸다. (Intelligent). -> **남의 패킷이 나한테 안 옴(스니핑 불가).**
* **CAM Table (Mac Address Table):** 스위치가 기억하는 [MAC 주소 - 포트 번호] 매핑 테이블.
* **Attack Vector (MAC Flooding):**
    * 공격자가 가짜 MAC 주소를 무한대로 생성해 스위치에 쏟아붓는다.
    * CAM Table이 꽉 차면 스위치는 더 이상 학습을 못하고, **모든 패킷을 브로드캐스팅하는 'Fail-Open' 상태(더미 허브 모드)**가 된다.
    * 이때부터 스니핑이 가능해진다.

---

## 2. ARP (Address Resolution Protocol)

**"논리 주소(IP)를 물리 주소(MAC)로 바꾸는 과정."**
LAN 통신은 최종적으로 MAC 주소를 알아야만 가능하다.

### 동작 원리 (Request & Reply)
1.  **Request (Broadcast):** "IP 192.168.0.10 누구야? MAC 주소 좀 알려줘!" (모두에게 소리침)
2.  **Reply (Unicast):** "나야! 내 MAC은 AA:BB:CC... 야." (질문자에게만 귓속말)
3.  **Caching:** "아, 192.168.0.10 = AA:BB:CC... 구나." 하고 **ARP Table**에 저장.

### ⚠️ The Fatal Flaw (Stateless Protocol)
ARP는 **"인증"**도 없고 **"상태 확인(Stateless)"**도 안 한다.
* **문제점:**
    1.  내가 물어보지도 않았는데 누군가 "내가 걔야!"라고 정보를 줘도 **무조건 믿고 테이블을 갱신한다.** (Gratuitous ARP)
    2.  누가 진짜 주인인지 검증할 방법이 없다.

---

## 3. ARP Spoofing (MITM Attack)

ARP의 취약점을 이용해 네트워크 흐름을 가로채는 **중간자 공격(Man-In-The-Middle)**의 핵심.



### 공격 시나리오
공격자(Attacker), 피해자(Victim), 게이트웨이(Router)가 있다고 가정.

1.  **Infection (감염):**
    * 공격자는 Victim에게: "나(Attacker)가 Router야." 라고 거짓 ARP Reply를 지속적으로 보냄.
    * 공격자는 Router에게: "나(Attacker)가 Victim이야." 라고 거짓 ARP Reply를 지속적으로 보냄.
2.  **Routing (패킷 릴레이):**
    * Victim이 네이버(인터넷)에 접속하려 하면, 패킷은 Router가 아닌 **Attacker에게 먼저 간다.**
    * Attacker는 이 패킷을 엿보거나(Sniffing) 변조한 뒤, 진짜 Router에게 전달해준다. (Victim은 인터넷이 잘 되니 해킹당한 줄 모름).
3.  **Result:**
    * HTTP 평문 통신 시 ID/PW 유출.
    * DNS Spoofing과 연계하여 가짜 사이트로 유도 가능.

> **💡 Tools:** `arpspoof` (dsniff 패키지), `Ettercap`, `Bettercap` 등이 이 과정을 자동화해준다.

---

## 4. Promiscuous Mode (무차별 모드)

네트워크 인터페이스 카드(NIC)의 동작 모드 중 하나.

* **Normal Mode:** 패킷의 수신처 MAC 주소가 '나'이거나 '브로드캐스트'인 경우만 받아들이고, 나머지는 버린다(Drop).
* **Promiscuous Mode:** **지나가는 모든 패킷을 다 받아들여 CPU로 보낸다.**
    * 와이어샤크(Wireshark)나 Tcpdump를 켤 때 이 모드가 활성화된다.
    * **주의:** 암호화된 와이파이 환경(WPA2/3)에서는 이 모드만으로 남의 데이터를 볼 수 없다. (복호화 키가 없으므로). 하지만 ARP Spoofing으로 패킷을 나한테 오게 만들면 볼 수 있다.

---

## 5. Mitigation (방어 대책)

이 강력한 L2 공격을 막기 위해 네트워크 관리자가 사용하는 기술들.

### Static ARP (정적 ARP)
* ARP 테이블을 수동으로 고정해버린다.
* `arp -s 192.168.0.1 AA:BB:CC:DD:EE:FF`
* **단점:** 기기가 수백 대면 관리가 불가능하다.

### Port Security (Switch Feature)
* 스위치 포트에 특정 MAC 주소만 접속하도록 제한한다. (MAC Flooding 방지)

### DAI (Dynamic ARP Inspection)
* **[가장 효과적]** 스위치가 DHCP 스누핑 정보를 바탕으로, "이 포트에서 이 IP가 이 MAC을 쓰는 게 맞아?"라고 검사한다.
* ARP Reply가 가짜(Spoofing)라고 판단되면 패킷을 차단한다.
