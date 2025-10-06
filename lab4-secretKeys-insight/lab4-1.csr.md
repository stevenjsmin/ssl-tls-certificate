# CSR: Certificate Signing Request
### CSR생성 절차

#### 1) 개인키(Private Key) 생성
CSR은 개인키를 기반으로 생성되므로, 먼저 개인키를 만들어야 한다.<br/>
**예] (RSA 2048bit)**
```shell
  openssl genrsa -out myserver.key 2048
  # → myserver.key 파일이 생성되며, 이는 비밀리에 보관해야 한다. (나중에 인증서 적용 시에도 이 파일이 필하다.)
```

#### 2) CSR 생성 (비대화형)
**예] 명령 예시**
```shell
  openssl req -new -key myserver.key -out myserver.csr \
  -subj "/C=AU/ST=VIC/L=Melbourne/O=MyOrg/OU=IT/CN=myapp.example.com"

  # -new  : 새 요청 생성
  # -key  : 위에서 만든 개인키 사용
  # -out  : 결과물 파일명 (.csr)
  # -subj : 인증서의 주체(subject) 정보 (비대화형 입력), 인증서를 사용하는 서버나 조직의 신원 정보를 의미한다.
  #          그래서 “이 인증서를 사용하게 되는 서버”의 정체를 나타내는 정보가 들어간다.
  
  ## 주요필드 의미
  # - C  : 국가 코드 (예: AU, KR, US)
  # - ST : 주/도 (예: VIC, NSW, Seoul)
  # - L  : 도시 (예: Melbourne)
  # - O  : 조직명 (예: MyOrg)
  # - OU : 부서명(Organizational Unit)(예: IT)
  # - CN : 공통 이름 (예: myapp.example.com)

```
** 여기서 **CN 필드**는 특히 중요하다. → 브라우저나 클라이언트가 인증서의 CN 또는 SAN 항목이 자신이 접속한 호스트명과 일치하는지 확인하기 때문이다.

<br>

#### 주의
과거에는 CN(Common Name) 필드 하나만으로 도메인을 식별했다.  하지만 지금은 SAN(Subject Alternative Name) 필드가 표준이자 필수이다.

| 항목  | 의미            | 예시                |
|:----|:--------------|:------------------|
| CN  | 인증서의 “대표 도메인” (예전 표준) | myapp.example.com |
| SAN  | 인증서가 유효한 추가 도메인 목록 (현행 표준) | myapp.example.com, www.myapp.example.com, api.myapp.example.com |

📌 **중요한 점:**<br>
브라우저(Chrome, Firefox, Edge 등)는 CN을 무시하고 SAN만 검사한다. → SAN에 대상 도메인을 반드시 포함해야 한다.
SAN에는 여러 도메인을 넣을 수 있고, 와일드카드(*.example.com)도 가능. <br>

아래 섹션은 CSR configuration filed에서 "subjectAltName = @alt_names"를 설정해서 alt_names섹센을 참조할수 있도록 할 수 있다.

```shell
[ alt_names ]
DNS.1 = myapp.example.com
DNS.2 = www.myapp.example.com
DNS.3 = api.myapp.example.com

```


----
### CSR을 Config file을 이용한 생성
커맨드라인에 -subj로 긴 DN을 넣는 대신, **OpenSSL 설정파일(컨피그 파일)** 을 만들어서 그걸 인자로 주면 **비대화식(자동화)**으로 CSR을 만들 수 있다. 
핵심은 req 섹션과 distinguished_name, 그리고 SAN(Subject Alternative Name) 같은 확장값을 req_extensions로 지정하는 것이다. 이 환경설정 파일은 INI형식으로 작성된다.


#### Example - (csr.conf)
```shell
# csr.conf
[ req ]
default_bits       = 2048
default_md         = sha256
prompt             = no                 # 비대화식
encrypt_key        = no                 # key 암호 미설정(자동화 용이)
distinguished_name = dn
req_extensions     = req_ext            # CSR에 포함할 확장 섹션 지정

[ dn ]
C  = AU
ST = VIC
L  = Melbourne
O  = MyOrg
OU = IT
CN = myapp.example.com                 # Common Name (브라우저는 SAN 우선 사용)

[ req_ext ]
# 현대 브라우저/서버는 CN보다 SAN을 신뢰합니다.
subjectAltName = @alt_names
keyUsage = critical, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth

[ alt_names ]
DNS.1 = myapp.example.com
DNS.2 = www.myapp.example.com
# 와일드카드도 가능(공인 CA 정책 확인 필요)
# DNS.3 = *.example.com
# IP 주소가 필요하면:
# IP.1 = 10.0.0.15

```

**포인트**
- prompt = no: 질문 없이 자동 생성
- req_extensions = req_ext: CSR에 확장값(SAN 등) 포함
- SAN은 subjectAltName = @alt_names → [alt_names]에서 DNS/IP 나열

#### CSR 생성 커맨드

**(A) RSA 새 키와 CSR 동시 생성**
<br/><u>공개키(public key)는 개인키(private key)만 있어도 수학적으로 추출(derive)할 수 있다. 이건 비대칭키 암호화(예: RSA, ECDSA, ED25519 등) 의 핵심 원리 중 하나이다.</u>
```shell
openssl req -new \
  -newkey rsa:2048 -nodes \
  -keyout server.key \
  -out server.csr \
  -config csr.conf

```

**기존 키로 CSR만 생성(키 재사용)**
```shell
openssl req -new \
  -key server.key \
  -out server.csr \
  -config csr.conf
```

#### CSR 내용 검증
```shell
# 전체 내용(확장 포함) 확인
openssl req -in server.csr -noout -text

# 주체(Subject)만 확인
openssl req -in server.csr -noout -subject

# SAN만 깔끔하게 추출(지원되는 openssl 버전)
openssl req -in server.csr -noout -text | grep -A1 "Subject Alternative Name"

```


#### (참고) PKCS#12/JKS를 쓰는 자바/톰캣 환경에서의 CSR
OpenSSL 컨피그 방식이 가장 유연하지만, 이미 PKCS#12/JKS 키스토어를 만든 뒤 CSR이 필요하면 keytool도 사용 가능(컨피그 파일은 아니지만 SAN 지정 가능).
```shell
# PKCS12에 있는 키/인증서로 CSR 생성 (SAN 포함)
keytool -certreq \
  -keystore tomcat.p12 -storetype PKCS12 \
  -alias tomcat \
  -file tomcat.csr \
  -ext SAN=dns:myapp.example.com,dns:www.myapp.example.com

```