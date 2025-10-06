# CSR: Certificate Signing Request

### CSR을 Config file을 이용한 생성
커맨드라인에 -subj로 긴 DN을 넣는 대신, **OpenSSL 설정파일(컨피그 파일)**을 만들어서 그걸 인자로 주면 **비대화식(자동화)**으로 CSR을 만들 수 있다.

**OpenSSL 설정파일(컨피그 파일)**을 만들어서 그걸 인자로 주면 **비대화식(자동화)**으로 CSR을 만들 수 있다. 
핵심은 req 섹션과 distinguished_name, 그리고 SAN(Subject Alternative Name) 같은 확장값을 req_extensions로 지정하는 것이다.

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