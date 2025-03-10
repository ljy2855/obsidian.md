

### L4 LB vs L7 LB

#### L4
OSI 4계층 transport layer load balancer의 경우

client의 트래픽 `TCP SYN` 요청을 처리
- TLS/SSL 불가 (L4 상위 레이어)
- 속도 빠름 



#### L7


#### 왜 L4가 L7보다 빠를까
- **IP 주소와 포트 번호만 확인**하여 트래픽을 분산
상위 **프로토콜의 헤더를 파싱할 필요가 없어**서 처리 속도가 훨씬 빠름.

- 커널 파싱

### Load balancing algorithm


#### Round Robbin

#### Sticky RR

#### Weighted RR

#### IP/URL Hash

#### Least Connections

#### Least Time

