# OpenSSL & Keytool
OpenSSL과 Keytool은 둘 다 인증서, 키, 암호화 관련 도구지만, 태생과 주 사용 목적이 다르다.

---
# 🔑 OpenSSL

- **출신/환경:** 리눅스/유닉스 진영에서 가장 널리 쓰이는 범용 암호화 라이브러리 및 CLI 도구.
- **주요 용도:**
   - RSA, ECC 등 키 쌍 생성 (private/public key)
   - CSR (Certificate Signing Request) 생성 및 검증
   - X.509 인증서 확인/변환 (PEM, DER, PFX 등 다양한 포맷)
   - SSL/TLS 연결 디버깅 (openssl s_client 등으로 서버 인증서 확인)
   - 암호화/복호화, 서명/검증, 해시 기능 제공
- **장점:**
   - 포맷 변환에 매우 강함 (PEM ↔ DER ↔ PKCS12 등)
   - 거의 모든 리눅스/맥 환경에 기본 포함
   - Apache, Nginx, HAProxy 등 서버 설정과 궁합이 좋음

👉 즉, 범용 암호화 및 인증서 툴킷

<br/><br/>
# 🔑 Keytool
**출신/환경:** Java(JDK)에 포함된 기본 도구.
- **주요 용도:**
  - Java Keystore (JKS, PKCS12) 관리
  - 키 쌍 생성 (-genkeypair)
  - CSR 생성 (-certreq)
  - 인증서/체인 import/export (-importcert)
  - keystore 내용 조회 (-list -v)

- **장점:**
  - Java 애플리케이션/Tomcat/Jetty/WebLogic 등에서 바로 사용 가능
  - JKS 같은 Java 전용 저장소 형식에 최적화

- **제한:**
  - OpenSSL처럼 범용 암호화 연산(직접적인 암호화/복호화, 해시 계산 등)은 불가
  - 주로 "키스토어와 인증서 관리"에 집중됨

👉 즉, **Java** 환경을 위한 인증서/키스토어 관리 전용 도구

<br/><br/><br/><br/>