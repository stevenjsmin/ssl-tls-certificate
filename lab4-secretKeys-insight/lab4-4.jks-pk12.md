# JKS and PK12

| 구분  | PEM / CER |                           JKS |
|:----| :---: |------------------------------:|
| 형식  | 텍스트 기반 인증서 파일 | Java 전용 바이너리 키스토어 |
| 주용도 | 공개키 인증서(및 체인) 저장 |           개인키 + 인증서 체인을 함께 보관 |
| 표준  | PKCS#7, X.509, Base64 인코딩 | Java KeyStore (Sun/Oracle 형식) |


### 구성 요소
- PEM/CER
  - 일반적으로 인증서(공개키)만 포함 (-----BEGIN CERTIFICATE----- ... -----END CERTIFICATE-----)
  - .pem 파일에는 다수의 인증서를 순서대로 여러 개 포함할 수도 있음 (예: 서버 + 중간 + 루트 체인)
  - 개인키는 별도 파일(.key, .pem)로 관리

- JKS
  - 하나의 파일에 개인키, 공개키 인증서, 체인, 신뢰된 CA 등을 함께 저장
  - Java 기반 서버(Tomcat, Spring Boot 등)에서 SSL 설정 시 자주 사용


### 사용 명령 예시
| 목적  |                               PEM / CER                               | JKS |
|:----|:---------------------------------------------------------------------:|----:|
| 내용 보기  |                openssl x509 -in cert.pem -text -noout                 | keytool -list -v -keystore keystore.jks |
| 서명 요청(CSR)  |           openssl req -new -key private.key -out server.csr           | keytool -certreq -alias tomcat -keystore keystore.jks |
| 변환  | openssl pkcs12 -export -in cert.pem -inkey key.pem -out keystore.p12  | keytool -importkeystore -srckeystore keystore.p12 -srcstoretype PKCS12 -destkeystore keystore.jks |

### 확장 및 변환 관계
PEM/CER은 단일 인증서, JKS는 Java 서버용 통합 저장소로 보면 된다.
![pemjks.png](../../../../Downloads/pemjks.png)


<br/><br/><br/>
# JKS를 이용한 인증서 관리예