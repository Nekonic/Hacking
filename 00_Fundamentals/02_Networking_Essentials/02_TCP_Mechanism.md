# 02_TCP_Mechanism

신뢰성 있는 연결(Reliability)을 보장하기 위한 TCP의 메커니즘과 이를 역이용한 공격 기법을 다룬다.
해커에게 TCP는 단순한 통신 수단이 아니라, **상태(State)를 조작하여 방화벽을 속이거나 서버 리소스를 고갈시키는 공격 도구**이다.

---

## 1. TCP Header Flags (Control Bits)

6개의 비트(Flag)를 조합하여 현재 통신의 상태와 목적을 알린다.
Nmap 같은 스캐너는 이 플래그를 비정상적으로 조작하여 방화벽을 뚫거나 OS를 탐지한다.

| Flag | Full Name | 의미 | 🔒 Security Perspective (Attack Use) |
| :---: | :--- | :--- | :--- |
| **SYN** | Synchronize | 연결 요청. 시퀀스 번호 동기화. | **SYN Flood:** 응답(ACK)을 안 주고 요청만 무한히 보내 서버 큐를 채움. |
| **ACK** | Acknowledgment | 응답. 이전 패킷 잘 받았음. | **ACK Scan:** 방화벽이 Stateful인지 Stateless인지 탐지할 때 사용. |
| **FIN** | Finish | 연결 종료 요청. | **FIN Scan:** 몰래 포트 오픈 여부를 찔러볼 때 사용. |
| **RST** | Reset | **연결 강제 초기화.** (비정상 종료) | **RST Injection:** 통신 중인 두 사람 사이에 끼어들어 강제로 연결을 끊음. |
| **PSH** | Push | 버퍼링 없이 즉시 상위 계층으로 전달. | 데이터를 꽉 채우지 않고 바로 처리하게 할 때. |
| **URG** | Urgent | 긴급 데이터. (거의 안 쓰임) | 일부 IDS 우회나 특수목적 통신에 악용. |

---

## 2. The State Machine (3-Way Handshake)

연결을 맺는 과정에서 양쪽의 상태(State) 변화를 이해해야 한다.



1.  **SYN (Client → Server):**
    * Client: `SENT_SYN` 상태.
    * **ISN (Initial Sequence Number):** 보안을 위해 난수로 생성된 초기 시퀀스 번호 전송.
2.  **SYN+ACK (Server → Client):**
    * Server: `SYN_RCVD` 상태. (반쯤 열린 상태, **Half-Open**)
    * 서버는 이 연결 정보를 **Backlog Queue**에 저장하고 클라이언트의 응답을 기다림.
    * **Attack Vector (SYN Flood):** 공격자가 여기서 마지막 ACK를 안 보내면, 서버의 Queue가 꽉 차서 정상 사용자의 접속을 못 받게 된다.
3.  **ACK (Client → Server):**
    * Client/Server: `ESTABLISHED` 상태.
    * 비로소 데이터 전송 가능.

---

## 3. Port Scanning Mechanics (Stealth & Evasion)

포트 스캐닝은 "서버가 패킷에 어떻게 반응하는가(RFC 표준)"를 이용해 포트가 열렸는지 닫혔는지 판별하는 기술이다.

### 1) TCP Connect Scan (Full Open)
* **방식:** 3-Way Handshake를 끝까지 완료함. (`SYN` -> `SYN/ACK` -> `ACK`)
* **특징:** 정확하지만 로그(Log)가 남아서 들키기 쉽다.

### 2) SYN Scan (Half-Open) - *Standard*
* **방식:** `SYN` 보냄 -> 서버가 `SYN/ACK` 주면 -> **즉시 `RST` 보내고 도망감.**
* **특징:** 연결이 완전히 맺어지지 않았으므로 애플리케이션 로그에 남지 않을 확률이 높음. (빠르고 은밀함)

### 3) Inverse Mapping (Null, FIN, Xmas) - *Advanced*
* RFC 표준상, **닫힌 포트(Closed Port)**에 이상한 패킷을 보내면 `RST`를 반환해야 한다.
* **방식:**
    * **Null:** 플래그 없음.
    * **FIN:** 뜬금없이 종료 요청.
    * **Xmas:** `FIN + PSH + URG` (전구 켜지듯 다 켜서 보냄).
* **판별:**
    * 응답이 없다? -> Open (혹은 방화벽 필터링).
    * `RST`가 왔다? -> Closed.
* **OS Fingerprinting:** **윈도우(Windows)**는 표준을 지키지 않고 닫힌 포트든 열린 포트든 무조건 `RST`를 보낸다. 이를 통해 타겟이 리눅스인지 윈도우인지 구별한다.

---

## 4. Sequence Number Prediction (Session Hijacking)

TCP의 신뢰성을 보장하는 시퀀스 번호(SEQ)가 예측 가능하면, 인증 과정을 무시하고 세션을 가로챌 수 있다.



### 작동 원리
* **SEQ:** 내가 보내는 데이터의 시작 바이트 번호.
* **ACK:** 상대방한테 "다음에 이 번호(SEQ) 보내줘"라고 요청하는 번호.
    * `Next SEQ` = `Current SEQ` + `Data Length`

### Attack Scenario (Blind Hijacking)
1.  공격자는 A(클라이언트)와 B(서버)가 통신 중임을 안다. (스니핑 등)
2.  공격자는 A의 IP로 위장(Spoofing)한다.
3.  현재 오고 가는 패킷의 SEQ 번호를 보고, **다음에 올 SEQ 번호를 계산(예측)**한다.
4.  B(서버)에게 A보다 먼저, **예측한 SEQ를 담은 악성 패킷(명령어)**을 보낸다.
5.  B는 SEQ가 맞으므로 정상적인 패킷으로 착각하고 명령어를 실행한다.
6.  진짜 A가 뒤늦게 패킷을 보내면, 이미 SEQ가 지나갔으므로 B는 A의 패킷을 버린다(ACK Storm 발생).

> **💡 Security Note:** 과거에는 ISN(초기 시퀀스 번호)이 시간 기반으로 생성되어 예측이 쉬웠으나, 최신 OS는 난수화(Randomization)가 잘 되어 있어 예측이 어렵다. 하지만 ARP Spoofing으로 패킷을 볼 수 있는 로컬 환경(LAN)에서는 여전히 유효하다.

---

## 5. Connection Termination (4-Way Handshake)

종료할 때는 4번의 과정이 필요하다. 여기서 발생하는 `TIME_WAIT` 상태는 서버 엔지니어와 해커 모두에게 중요하다.

1.  **FIN_WAIT:** "나 끊을래" (Active Close).
2.  **CLOSE_WAIT:** "어 알겠어, 나도 정리 좀 하고." (Passive Close).
3.  **LAST_ACK:** "정리 끝났어, 나도 끊을게."
4.  **TIME_WAIT:** "OK, 잘 가." **(중요)**

### TIME_WAIT의 존재 이유와 위험성
* **이유:** 마지막 `ACK`가 네트워크에서 손실되었을 때를 대비해, 소켓을 바로 닫지 않고 잠시 기다려주는 시간.
* **DoS 공격:** 공격자가 엄청나게 많은 연결을 맺고 끊기를 반복하면, 서버의 가용 포트가 모두 `TIME_WAIT` 상태로 잠겨버려 더 이상 연결을 받을 수 없게 된다.
