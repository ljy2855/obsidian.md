

SSL : secure socket layer
TLS: transport layer secure

이젠 SSL이 사용되지 않고 이후 나온 프토로콜이 TLS (다만 같은 용어처럼 쓰임)

주로 웹통신 HTTP를 암호화하는데 사용되지만, SMTP, VOIP 같은 프토콜도 사용 가능

### TLS 기능

- **기밀성** (Confidentiality): 통신을 암호화해서 packet snipping에서 데이터를 숨김
- **인증** (authenticate): 데이터 송신자가 요청한 사람인 걸 확인
- **무결성**(Integrity): 중간에 데이터가 위조되었는지 방지

### 대칭키, 비대칭키
- 대칭키 (symmetric key) 
	- 키암호화 복호화에 같은 키를 사용
	- 암호화를 위해 키를 교환해야함
- 비대칭키 (asymmetric key) : 암호화 복호화에 각각 다른 키 사용

##### 위키
TLS에서는 **대칭키 암호화**와 **비대칭키 암호화**를 모두 활용하여 보안을 달성합니다​

. **대칭키 암호화**는 하나의 비밀 키를 통신 당사자들이 공유하여 데이터를 암호화/복호화하는 방식으로, 연산 속도가 빠르고 오버헤드가 적어 **대용량 데이터의 실시간 암호화**에 적합합니다. 하지만 사전에 키를 안전하게 공유하는 것이 어렵기 때문에, TLS에서는 세션 시작 시 **키 교환**에 비대칭키 기술을 사용하고 그 후의 본격적인 데이터 전송에는 대칭키를 사용합니다​

. 대표적인 대칭키 알고리즘으로 AES, ChaCha20 등이 TLS에서 사용되며, 이들은 협상된 세션 키로 통신 내용을 암호화하여 기밀성을 확보합니다. 반면 **비대칭키 암호화(공개키 암호)**는 **공개키-개인키 한 쌍**을 이용하는 방식으로, 한 쪽 키로 암호화한 데이터는 짝을 이루는 다른 키로만 복호화가 가능합니다. TLS 핸드셰이크 과정에서는 주로 공개키 암호 기술이 사용되는데, 예를 들어 서버는 자신의 공개키를 인증서에 담아 제공하고 클라이언트는 그 공개키로 **공유 비밀 정보를 암호화**하여 보내거나, **Diffie-Hellman** 알고리즘을 이용해 안전하게 세션 키를 교환합니다​

. 이러한 비대칭키 방식은 **초기 인증과 키 교환**에 활용되어, 사전에 비밀을 공유하지 않은 상태에서도 안전하게 대칭 세션 키를 합의할 수 있게 해줍니다​

. 단, 비대칭 암호화는 대칭키에 비해 연산이 느리고 복잡도가 높으므로, **세션이 성립된 이후의 데이터 통신에는 대칭키 암호화**를 사용하여 효율을 높입니다. 요약하면 **비대칭키**는 TLS의 초기 **핸드셰이크 단계**에서 **신원 확인**과 **세션 키 교환**을 위해 쓰이고, 합의된 **대칭키**는 이후 **데이터 암복호화**에 사용되는 것입니다​

### TLS 인증서?
#### Public Key
- 브라우저가 암호화 키로 사용

#### Private Key
- 서버가 브라우저의 데이터를 복호화하기 위해 사용
- 반대로 **private(암호화) -> public (복호화) 시에는 인증 용도**
- 외부로 유출되면 안됌
### TLS handshake
![[Pasted image 20250312135140.png]]

1. 브라우저 서버 TCP 3 handshake
2. 브라우저 `버전`, `암호 알고리즘`, `압축 방식` 를 담아 암호문 전달
3. 

### 부록
#### TLS가 성능이 미치는 영향?


#### TLS를 사용하지 않아도 되는 상황

#### SSH 프로토콜과 비교