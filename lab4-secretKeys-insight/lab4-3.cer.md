# Certification

“인증서 파일 구조”는 단순히 텍스트 파일이 아니라, 암호화된 데이터 블록과 메타데이터(Subject, Issuer, 키, 서명 등) 가 표준 포맷으로 저장된 것이다.

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