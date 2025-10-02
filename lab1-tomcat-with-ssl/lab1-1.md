# Config SSL/TLS on Tomcat

### 1) 사전 준비: keystore(PKCS12) 생성

```shell
    # 1-1. 새 키페어 생성 (PKCS12)
    keytool -genkeypair \
      -alias tomcat \
      -keyalg RSA -keysize 2048 \
      -validity 365 \
      -storetype PKCS12 \
      -keystore /Users/stevenmin/Downloads/tomcat.p12 \
      -storepass changeit \
      -keypass changeit \
      -dname "CN=myapp.example.com, OU=IT, O=MyOrg, L=Melbourne, ST=VIC, C=AU"
      
    # 이것은 아직 인증서가 발행된것이 아니라 키페어(private+public)이 생성된것이다.
```

잠깐!:
```shell

      # 키와 인증서 확인
      keytool -list -v -keystore /Users/stevenmin/Downloads/tomcat.p12 -storetype PKCS12 -storepass changeit
    
      # 인증서 추출
      openssl pkcs12 -in /Users/stevenmin/Downloads/tomcat.p12 -clcerts -nokeys -out tomcat-cert.pem
    
      # 인증서 내용 확인
      openssl x509 -in tomcat-cert.pem -text -noout
    
      # 위 -genkeypair로부터 생성된 tomcat.p12로부터 private 키 생성하기
      openssl pkcs12 -in /Users/stevenmin/Downloads/tomcat.p12 \
           -nocerts -nodes -out /Users/stevenmin/Downloads/tomcat.key
    
      # 위 -genkeypair로부터 생성된 tomcat.p12로부터 공개키(=서명된 인증서 부분) 추출
      openssl pkcs12 -in /Users/stevenmin/Downloads/tomcat.p12 -nokeys -out /Users/stevenmin/Downloads/tomcat.crt
      # *** 중요 : 지금 보이는 인증서[1] 은 CA 인증서가 아니라, 개인키 생성 시 자동으로 만들어진 self-signed 인증서이다.
      #           즉, tomcat.p12에는 
      #               - 개인키(Private Key)
      #               - 그에 대응하는 공개키(Public Key) 를 담은 인증서 (self-signed)
```

----
### 2) 사전 준비: 생성된 Keypair로 CSR 생성 
```shell
    # 1-2. CSR 생성 (CA 제출용)
    keytool -certreq \
      -alias tomcat \
      -file /Users/stevenmin/Downloads/server.csr \
      -keystore /Users/stevenmin/Downloads/tomcat.p12 \
      -storetype PKCS12 \
      -storepass changeit
    
    ## 여기서 server.csr가 생성되는데, 이 안에는 
    #     - Subject (주체 정보, DN – Distinguished Name)
    #     - CSR에는 개인키에 대응하는 Public Key (공개키)가 포함되어 있습니다.
    #     - 요청된 확장 필드 (Extensions, SAN 등)
    #     - CSR 자체를 개인키로 통째로 암호화 하는 건 아니고, 단지 “본문의 해시값”을 개인키로 서명해서 포함시킴
    #     - *** 개인키는 CSR에 포함되지 않습니다. 즉 CSR에는 공개키와 서명만 들어간다.
```

----
### 3) CSR을 인증기관(CA)에 요청
CSR(server.csr)을 인증기관(CA)에 제출하면 보통 서버 인증서(예: server_cert.crt) 와 **중간/루트 체인(chain.crt 또는 중간/루트 개별 파일)**을 받게된다.

----
### 4) CA 인증서(체인) 및 서버 인증서 가져오기 
받은 파일이 아래 두 가지 형태 중 하나일 수 있음.

- (A) 분리형: server_cert.crt + chain.crt(묶음) 또는 intermediate.crt, root.crt 등
- (B) 서버+체인 합본: server_cert_with_chain.crt (서버 인증서 뒤에 체인이 이어붙은 형태)

(A) 분리형인 경우(가장 흔함)
```shell
# (선택) 체인이 여러 파일로 오면 하나로 합칩니다(중간 → 루트 순서 권장)
    cat intermediate.crt root.crt > /Users/stevenmin/Downloads/chain.crt
    
    # 2-1-1. 체인(중간/루트) 인증서 먼저 가져오기
    keytool -importcert \
      -keystore /Users/stevenmin/Downloads/tomcat.p12 \
      -storetype PKCS12 \
      -storepass changeit \
      -alias ca-chain \
      -file /Users/stevenmin/Downloads/chain.crt \
      -noprompt
    
    # 2-1-2. 서버 인증서를 "동일 alias(tomcat)"로 가져오기
    keytool -importcert \
      -keystore /Users/stevenmin/Downloads/tomcat.p12 \
      -storetype PKCS12 \
      -storepass changeit \
      -alias tomcat \
      -file /Users/stevenmin/Downloads/server_cert.crt
```
(B) 서버+체인 합본 파일만 받은 경우
```shell
    keytool -importcert \
      -keystore /Users/stevenmin/Downloads/tomcat.p12 \
      -storetype PKCS12 \
      -storepass changeit \
      -alias tomcat \
      -file /Users/stevenmin/Downloads/server_cert_with_chain.crt

```

----
### 5) 확인 
```shell
    keytool -list -v \
      -keystore /Users/stevenmin/Downloads/tomcat.p12 \
      -storetype PKCS12 \
      -storepass changeit

```
**  "새 키페어 생성"할때와의 비교
- Entry type은 계속 **PrivateKeyEntry**로 보이는 것이 정상(개인키 보유 항목).
- 달라지는 것은 인증서 체인 길이(체인 길이 > 1)와 Issuer(발행자) 가 CA로 바뀐다는 점.


----
### 6) Tomcat server.xml에 HTTPS 커넥터 추가 
Tomcat conf/server.xml의 기존 8080 커넥터는 유지하고, 8443 HTTPS 커넥터를 추가 또는 활성화 한다.
```html
<!-- conf/server.xml 내, <Service name="Catalina"> 안쪽에 추가 -->
    <Connector
        port="8443"
        protocol="org.apache.coyote.http11.Http11NioProtocol"
        SSLEnabled="true"
        scheme="https"
        secure="true"
        clientAuth="false"
    
        keystoreFile="/Users/stevenmin/Downloads/tomcat.p12"
        keystoreType="PKCS12"
        keystorePass="changeit"
    
        # 최신 Tomcat은 아래 속성 불필요한 경우도 많지만, 명시해도 무방
        sslProtocol="TLS"
        />
```
파일 권한:
- Tomcat 프로세스가 tomcat.p12를 읽을 수 있어야 함.

----
### 7) Tomcat 재시작


----
### 8) 동작확인
```shell
    # 브라우저
    https://myapp.example.com:8443/

    # CLI로 인증서 체인/호스트명 검사
    $> openssl s_client -connect myapp.example.com:8443 -servername myapp.example.com </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates
            subject=CN=myapp.example.com
            issuer=C=US, O=DigiCert Inc, OU=www.digicert.com, CN=GeoTrust TLS RSA CA G1
            notBefore=Sep 15 08:37:19 2025 GMT
            notAfter=Dec  8 08:37:18 2025 GMT

```


----
### 9) 자주 헷갈리는 포인트 정리
- 왜 파일이 두 개(서버 cert + 체인)인가?
서버 인증서는 해당 도메인에 대한 것이고, 체인은 그 인증서를 신뢰 루트까지 이어주는 중간/루트 인증서 묶음이다. 신뢰 사슬을 완성해야 클라이언트(브라우저)가 “아 이건 신뢰할 수 있구나” 하고 인식한다.

- Entry type이 왜 여전히 PrivateKeyEntry인가?
정상입니다. 해당 항목은 개인키를 포함하므로 PrivateKeyEntry로 표시된다. CA를 적용한 뒤에는 그 항목의 체인 길이가 2 이상이 되고, Issuer 가 CA로 변경된다.

- Alias 주의
서버 인증서는 반드시 처음 생성했던 alias(tomcat) 에 가져와야 함. 다른 alias에 가져오면 개인키와 매칭이 안 돼서 실패한다.

- 키/인증서를 OpenSSL로 다루는 경우
외부에서 개인키(server.key)와 인증서(server.crt, chain.crt)를 이미 가지고 있다면, 아래처럼 PKCS#12로 합쳐 Tomcat에서 그대로 쓸 수 있다.
```shell
    cat server.crt chain.crt > server_with_chain.crt
    
    openssl pkcs12 -export \
      -inkey server.key \
      -in server_with_chain.crt \
      -out /Users/stevenmin/Downloads/tomcat.p12 \
      -name tomcat \
      -passout pass:changeit

```





