

### L4 LB vs L7 LB

#### L4
OSI 4계층 transport layer load balancer의 경우

client의 트래픽 `TCP SYN` 요청을 처리
- TLS/SSL 불가 (L4 상위 레이어)
- IP, port 만 보고 로드 밸런싱
- 속도 빠름 

#### L7
Application layer load balancer

- HTTP, HTTPS 와 같은 특정 L7 프로토콜을 통한 로드밸런싱만 가능함
- TLS/SSL 처리를 통해 client <-> LB 간 통신 암호화 가능, server는 관리 안해도 ㄱㅊ
- server의 상태에 따라 다양한 LB 알고리즘 적용 가능: stateful
`nginx`, `HAproxy`

보통 

#### 왜 L4가 L7보다 빠를까
- **IP 주소와 포트 번호만 확인**하여 트래픽을 분산
상위 **프로토콜의 헤더를 파싱할 필요가 없어**서 처리 속도가 훨씬 빠름.

- L7 LB는 대부분 어플리케이션 형태로 시스템 콜을 통한 오버헤드 발생
- L4 는 커널 영여

### Load balancing algorithm
호스트들

#### Round Robbin
- host에 평등하게 하나씩 트래픽 분산

#### Sticky RR
- 같은 source로부터 온 request들은 동일한 host로 라우팅
	- cache, session, cookie 관리에 이점

#### Weighted RR
- 호스트에 가중치를 주어 이를 기준으로 RR
	- 현재 가용성 있는 host들 or 부하가 적은 host들에게 분산
	- 

#### IP/URL Hash
- Sticky RR처럼 동일한 client의 req를 동일한 서버로 보내기 위함
- source IP를 hash input으로 넣어 서버에 매핑

#### Least Connections
- 최근 연결이 적은 host들에게 트래픽 분산
- host들의 connection 정보를 알아야하므로 L7 LB에서 가능
#### Least Time
- 최근 응답이 가장 빠른 호스트로 트래픽 분산
- 마찬가지로 L7 LB만 가능


---
#### Firewall, Router 차이
LB는 어떻게보면 router 