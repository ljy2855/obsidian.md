
### TCP/IP 란?
TCP/IP는 인터넷에서 사용되는 핵심 프로토콜 스택으로, 데이터를 신뢰성 있게 전달하는 **전송 계층(Transport Layer)의 TCP**와, 목적지까지의 **라우팅을 담당하는 네트워크 계층(Network Layer)의 IP** 프로토콜을 함께 사용하는 것을 의미합니다.

- **IP (Internet Protocol, 인터넷 프로토콜)**
    
    - 패킷을 목적지까지 전달하는 역할
    - 호스트의 주소 지정 (IP 주소)
    - 패킷이 최적의 경로를 통해 이동하도록 라우팅
- **TCP (Transmission Control Protocol, 전송 제어 프로토콜)**
    
    - 신뢰성 있는 데이터 전송 제공 (패킷 손실, 순서 보장, 흐름/혼잡 제어)
    - 연결 지향적 프로토콜 (3-way handshake, 4-way teardown)

TCP/IP는 계층적으로 설계되어 있어 **각 계층이 독립적으로 동작**하면서도, 전체적으로 신뢰성 있는 네트워크 통신을 가능하게 합니다.

### IP 프로토콜의 라우팅 방법
IP 프로토콜을 패킷 단위로 데이터를 전송하게 되는데, 단 엣지를 통과할 때, 다음 router를 거치게 되고 해당 라우터는 패킷의 IP address에 매핑된 다음 라우터의 목적지로 전송하게 된다.

이 때, 라우터는 라우팅 테이블에 IP 대역마다 다음 hop을 저장하게 된다

주로 동적 라우팅 사용
#### 동적 라우팅
- 라우터들이 서로 정보를 교환하여 최적 경로를 자동으로 설정
- 네트워크 변화에 유동적으로 대응 가능 (물리적 단절, 최적 경로 업데이트)
- 주요 프로토콜:
    - **RIP (Routing Information Protocol)**: 거리 벡터 방식, 홉 수 기반
    - **OSPF (Open Shortest Path First)**: 링크 상태 방식, 다익스트라 알고리즘 기반
    - **BGP (Border Gateway Protocol)**: AS(자율 시스템) 간 라우팅


### 라우팅 테이블?
![[Pasted image 20250308150616.png]]

```bash
~ ❯ netstat -nr                                                               
Routing tables

Internet:
Destination        Gateway            Flags               Netif Expire
default            192.168.1.1        UGScg                 en0
127                127.0.0.1          UCS                   lo0
127.0.0.1          127.0.0.1          UH                    lo0
169.254            link#13            UCS                   en0      !
169.254.9.77       f8:63:3f:f:3f:7f   UHLSW                 en0     33
169.254.40.205     f8:63:3f:9:f1:14   UHLSW                 en0      !
169.254.65.198     90:78:41:9f:63:c0  UHLSW                 en0      !
169.254.92.93      2c:d:a7:b7:fb:dd   UHLSW                 en0   1111
169.254.109.130    bc:38:98:3:83:42   UHLSW                 en0      !
169.254.136.79     9c:29:76:cd:e6:14  UHLSW                 en0      !
169.254.152.208    98:af:65:a1:7c:55  UHLSW                 en0   1147
169.254.156.185    b6:5e:9b:fd:45:ba  UHLSW                 en0      !
169.254.169.183    8:6a:c5:86:a3:f8   UHLSW                 en0   1044
169.254.196.71     aa:66:df:c2:6c:77  UHLSW                 en0      !
169.254.210.25     d4:e9:8a:f9:4a:7   UHLSW                 en0      !
```

요런식으로 IP CIDR에 따라 gateway(외부 네트워크 연결 통로) 지정

### TCP의 특징
- connection oriented
	- 3 handshake를 통해 연결
	- 4 handshake를 통해 종료
		- 종료 이후에도 송신한 패킷 받기 가능
- reliable transport
	-  패킷 손실 발생 시 **재전송** (ACK, Timeout 기반)
	- 패킷 순서를 보장 (Sequence Number 활용)
- flow control
	- 송신자가 수신자의 처리 속도를 초과하지 않도록 조정
- congestion control
	- 네트워크 상태를 고려하여 패킷 전송 속도를 조절



### Congestion control

![[Pasted image 20250308151545.png]]

### TCP 3 handshake 

client : SYNC
server: SYNC + ACK
client : ACK

- **SYN 플러딩 공격** 같은 보안 이슈도 함께 준비하면 좋아!
- 3-way handshake 없이도 통신이 가능한지? (UDP와 비교)

### TCP vs UDP

### flow control vs congestion control

