# 1. 파일 형식별 개념적 차이

#### PEM (.pem)
- 텍스트(Base64 인코딩) 형식
- -----BEGIN CERTIFICATE----- 같은 헤더/푸터가 붙음
- 인증서, 공개키, 개인키, 체인(루트/중간 인증서) 모두 담을 수 있음

#### CRT / CERT (.crt / .cert)
- 보통 PEM 또는 DER 인코딩된 인증서
- 브라우저/서버에서 신뢰 체인용으로 사용

#### PRIVATE (.key, .private)
- 개인키 저장용 (PEM 형식이거나 PKCS#8)
- 서버가 SSL/TLS 통신 시 서명에 사용
 
#### PUBLIC (.pub, .public)
- 공개키 저장용
- PEM 텍스트나 OpenSSH 형식

#### PKCS12 (.p12, .pfx)
- 바이너리 포맷
- 개인키 + 인증서 체인(서버 인증서 + 중간/루트)을 하나의 파일에 패키징
- 암호로 보호됨 → keytool, openssl pkcs12 명령어로 내용 확인 가능

#### JKS (.jks)
- Java Keystore 포맷 (바이너리)
- 여러 개의 키/인증서를 별칭(alias)으로 저장 가능
- Tomcat, Spring Boot 같은 Java 앱에서 자주 사용

----
# 2. 내부 구조를 이해하는 방법
#### (1) openssl 이용

- PEM/CRT/KEY/CSR 파일 확인
```shell
    openssl x509 -in server.crt -text -noout
    openssl rsa -in server.key -text -noout
    openssl req -in server.csr -text -noout

```


PKCS12 확인
```shell
  openssl pkcs12 -in keystore.p12 -info -nodes
  # → 인증서 체인, 개인키가 어떻게 묶여있는지 확인 가능

```


#### (2) keytool 이용 (Java 환경)
JKS / PKCS12 내부 보기
```shell
  keytool -list -v -keystore keystore.jks
  # → 별칭(alias), 유효기간, 발행자, 서명 알고리즘까지 확인 가능

```

#### (3) Base64 디코딩해서 직접 보기
PEM은 결국 Base64 → DER(바이너리 ASN.1 구조)
```shell
    base64 -d server.crt > server.der
    openssl asn1parse -in server.der -inform DER
    # → ASN.1 구조 트리로 파싱 가능 (Subject, Issuer, Validity 등)
```


----
# 3. Insight 접근 방법

#### PEM부터 시작
- → 헤더/푸터와 Base64 구조를 직접 디코딩해보면 "아, 이게 결국 DER 바이너리 포맷을 텍스트로 감싼 거구나" 이해됨.

#### PKCS12 / JKS는 패키징 개념으로 접근
- → 여러 키와 인증서를 하나의 컨테이너에 넣고 암호화/관리하기 위한 포맷.

#### 도구(OpenSSL, keytool)로 구조 확인
- → 단순히 확장자로 구분하지 말고 실제 내용을 열어보면서 “이 파일이 인증서인지, 키인지, 체인인지” 파악.


----
# 4. 추천 실습 루트
```shell
    openssl genrsa -out private.key 2048 → 개인키 생성
    openssl rsa -in private.key -pubout -out public.key → 공개키 추출
    openssl req -new -key private.key -out server.csr → CSR 생성
    openssl x509 -req -in server.csr -signkey private.key -out server.crt → Self-signed cert
    openssl pkcs12 -export -inkey private.key -in server.crt -out keystore.p12 → PKCS12 생성
    keytool -importkeystore -srckeystore keystore.p12 -destkeystore keystore.jks → JKS 변환

```
이 과정을 따라가면 각 파일 내부 구조와 차이를 손으로 직접 체감하면서 이해할 수 있습니다.