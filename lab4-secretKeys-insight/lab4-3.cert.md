# Certification

“인증서 파일 구조”는 단순히 텍스트 파일이 아니라, 암호화된 데이터 블록과 메타데이터(Subject, Issuer, 키, 서명 등) 가 표준 포맷으로 저장된 것이다. 인증서 파일은 보통 X.509 형식(Base64 또는 DER 인코딩) 으로 저장되어 있으며, 이를 확인할 때는 openssl 또는 keytool 명령을 사용한다.

https://github.com/istio/istio/tree/master/samples/certs 에서는 테스트를 위한 여러종류의 인증서 파일들을 다운로드 받아서 테스트해볼수있다.

## 1. 인증서 파일이란?

인증서 파일은 보통 X.509 표준을 따른다.
즉, “이 공개키가 이 사람/서버/기관에 속한다”라는 것을 인증해주는 전자 문서다.

일반적으로 .crt, .cer, .pem, .der 같은 확장자로 제공되며, 안에는 다음 정보가 들어있다:

- **Subject (소유자 정보):** CN=도메인명, O=회사명, C=국가 등
- **Issuer (발급자 정보):** 인증서를 발급한 CA(인증기관)
- **Public Key (공개키):** 서버/사용자의 공개키
- **Validity (유효기간):** 시작일, 만료일
- **Serial Number (일련번호):** 고유 ID
- **Signature (서명 값):** 발급자가 자신의 Private Key로 서명한 값
- **Extensions:** 인증서 용도(서버 인증, 클라이언트 인증, 코드 서명 등), Key Usage, SAN(Subject Alternative Names, 여러 도메인 포함 가능) 등



## 2. 내부 구조 (DER vs PEM)
인증서 데이터는 크게 두 가지 표현 방식으로 저장된다.

#### 1. DER (Distinguished Encoding Rules)
    - 순수 이진(binary) 포맷
    - 기계가 읽기 좋은 형태
Windows에서 .cer, .der 확장자에 많이 쓰임

#### 2. PEM (Privacy Enhanced Mail)
    - Base64로 인코딩된 텍스트 포맷
    - 헤더/푸터 포함:
    - Linux/Unix, OpenSSL 등에서 주로 사용
```shell
    -----BEGIN CERTIFICATE-----
    (Base64 인코딩된 DER 데이터)
    -----END CERTIFICATE-----
```
- 👉 둘 다 같은 데이터 구조를 담지만, 표현 방식만 다르다.
- 👉 openssl x509 -in cert.pem -text 같은 명령으로 내부 내용을 사람이 읽을 수 있는 형태로 볼 수 있다.

---
## 3. 관리되는 파일 종류
실무에서는 여러 형태의 인증서/키 파일이 함께 쓰인다.
- 개인키 파일 (.key, sometimes in .pem, .p12)
    - 서버 자신이 보관 (절대 외부에 공개하면 안 됨)

- 서버 인증서 (.crt, .pem)
    - CA가 발급해준 서버 인증서
- 중간/루트 인증서 (chain.crt, ca-bundle.crt)
    - 브라우저/클라이언트가 신뢰할 수 있게 하는 체인
- PKCS#12 (.p12, .pfx)
    - 개인키 + 인증서 체인을 하나로 묶은 바이너리 파일 (비밀번호로 보호)
- JKS (Java KeyStore, .jks)
    - Java/Tomcat 같은 환경에서 쓰는 전용 저장소 (역시 PKCS#12와 비슷한 구조)

---
## 4. 어떻게 이해하면 좋은가?

- 계층 구조로 이해해야 함.
    - 내 서버 인증서는 반드시 상위 CA(중간 CA, 루트 CA)와 연결되어야 함
    - 최종적으로 루트 CA까지 이어지는 Chain of Trust 개념이 중요

- 공개키–개인키 쌍을 분리해서 생각.
    - 인증서 파일에는 보통 공개키와 소유자 정보, 서명 정보가 들어 있음
    - 개인키는 별도의 .key 파일에 따로 관리

---
## 5. ✅ 요약:
- 인증서 파일은 단순 텍스트가 아니라 공개키, 소유자/발급자 정보, 유효기간, CA 서명 등이 들어있는 전자 신분증이다.
- PEM/DER/JKS/PKCS12 같은 형태로 저장되며, 관리의 핵심은 개인키 보호, 체인 연결, 만료 주기 관리이다.

<br/><br/>

# 하나의 파일에 하나이상의 인증서
하나의 파일에 여러 개의 인증서를 넣을 수 있다.

### 1. 여러 인증서가 들어갈 수 있는 이유
- PEM 형식(.pem, .crt)은 단순히
```shell
        -----BEGIN CERTIFICATE-----
        ... (Base64 인코딩된 DER 데이터)
        -----END CERTIFICATE-----
    
```
블록들의 나열입니다.
즉, 여러 개의 -----BEGIN CERTIFICATE----- ... -----END CERTIFICATE----- 블록을 이어붙이면 한 파일 안에 여러 인증서를 보관할 수 있다.

### 2. 대표적인 사용 예시

#### 1. 인증서 체인(Chain File)
- 서버 인증서(server.crt) + 중간 인증서(intermediate.crt) + 루트 인증서(root.crt)를 하나의 PEM 파일로 합쳐서 fullchain.pem 형태로 제공
- 예:
    ```shell
        -----BEGIN CERTIFICATE-----
        (Server Certificate)
        -----END CERTIFICATE-----
        -----BEGIN CERTIFICATE-----
        (Intermediate CA)
        -----END CERTIFICATE-----
        -----BEGIN CERTIFICATE-----
        (Root CA)
        -----END CERTIFICATE-----
    
    ```


#### CA Bundle
- 여러 개의 CA 인증서를 묶은 파일
- 브라우저, curl, OpenSSL 같은 클라이언트가 서버 인증서 체인을 검증할 때 사용
- Linux에서 보통 /etc/ssl/certs/ca-bundle.crt 같은 경로에 있음

#### PKCS#7 (P7B, .p7b, .p7c)
- ASN.1 구조 안에 여러 인증서가 들어있는 컨테이너
- 보통 체인을 한 번에 담을 때 사용됨 (Windows 환경에서 자주 씀)

### 3. 주의할 점
- <u>**순서가 중요**</u>
    - <u>보통 서버 인증서 → 중간 CA → 루트 순서로 정렬해야 함</u>
    - <u>순서가 틀리면 certificate verify failed 오류 발생</u>

- 개인키는 별도 보관
    - 여러 인증서를 넣는 건 가능하지만, 개인키(.key)는 따로 관리해야 함
 
- 형식 구분 필요
    - PEM 기반이라면 여러 개 가능
    - DER 형식(.der)은 하나의 인증서만 담을 수 있음 (바이너리라 구분 불가)

### ✅ 요약:
- PEM 파일 하나에 여러 개의 인증서를 넣을 수 있다.
- 보통 서버 인증서 + 체인 인증서를 한 파일에 넣어서 fullchain.pem 으로 사용한다.
- 순서를 잘 맞추는 것이 핵심이다.

<br/><br/>
# 인증서 내용확인
OpenSSL로 확인 (가장 일반적)이며 인증서 파일의 상세정보(Subject, Issuer, 유효기간 등)를 출력할 수 있습니다.

### OpenSSL
```shell
    # PEM(Base64) 형식의 인증서:
    openssl x509 -in mycert.cer -text -noout
    
    # DER(바이너리) 형식의 인증서:
    openssl x509 -in mycert.cer -inform der -text -noout
    
    # 간단하게 유효기간만 보고 싶을 때
    openssl x509 -in mycert.cer -noout -dates

```

##### OpenSSL의 주요 옵션
- **-noout** : PEM/DER 인코딩된 본문(-----BEGIN ...----- 블록)을 출력하지 말라는 뜻.
- **-text** : 사람이 읽기 쉬운 상세 정보(텍스트 디코딩)를 출력, 예: Subject/Issuer, 유효기간, 서명 알고리즘, 확장(Key Usage, SAN) 등.
- **-subject** : 인증서이름
- **-issure** : 발행자
- **-dates** : 유효기간
                            
```shell
        # 요약만 보고 싶을 때:
        openssl x509 -in cert.pem -noout -subject -issuer -dates

        # Subject 형식을 깔끔하게:
        openssl x509 -in cert.pem -noout -subject -nameopt RFC2253
    
```

### Java 환경 (JKS 등)에서 keytool 사용 시
```shell
    keytool -printcert -file mycert.cer
    
```

# 인증서 관리

.cer(또는 .cert, .pem) 파일은 사실상 **Base64로 인코딩된 X.509 인증서(들)**을 담고 있는 텍스트 파일이다.

- 단일 인증서인 경우, mycert.cer 파일에 하나의 인증서만 들어 있다면, 교체는 간단히 아래처럼 덮어씌우면 된다. 
```shell
  cp new_cert.cer mycert.cer
```

- 여러 인증서가 포함된 체인(chain) 파일인 경우
```shell
    -----BEGIN CERTIFICATE-----
    (서버 인증서)
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
    (중간 인증서)
    -----END CERTIFICATE-----
    -----BEGIN CERTIFICATE-----
    (루트 인증서)
    -----END CERTIFICATE-----

```

- 특정 인증서를 “제거”하는 방법 - 가장 직관적인 방법이다.
  - chain.cer 파일을 텍스트 에디터(예: vi, nano, VSCode)로 엽니다.
  - 제거하고 싶은 인증서의 블록(-----BEGIN CERTIFICATE----- ~ -----END CERTIFICATE-----)을 통째로 삭제한다.
  - 저장 후 닫는다.

#### 주의
<U>**순서가 중요하다!. 보통 순서는 아래와 같다.**</U>
```shell
    [1] 서버 인증서
    [2] 중간 인증서
    [3] 루트 인증서

```


## KeyStore(JKS, PKCS12) 내부에서 인증서를 추가/삭제할 때
만약 .cer 파일이 아니라 JKS(.jks) 나 PKCS12(.p12/.pfx) 내부에 들어 있는 인증서를 추가·제거하고 싶다면, keytool 명령을 사용해야 한다:

**추가**
```shell
  keytool -importcert -alias mycert -file server.cer -keystore keystore.p12 -storetype PKCS12
```

**제거**
```shell
  keytool -delete -alias mycert -keystore keystore.p12 -storetype PKCS12
```