# PEM @AWS
AWS EC2에서 키페어를 생성하면 .pem 파일이 제공되는데, 실제로 이 파일은 Private Key만 포함하고 있다.
왜 Public Key가 안 보이냐면:


###  1. PEM 파일에는 Private Key만 저장
    - AWS에서 EC2 Key Pair를 생성할 때, 클라이언트(당신)에게 내려주는 건 **비밀키(Private Key)**예요.
    - Public Key는 따로 .pem에 담기지 않는다.
      즉:
    - Private Key → 당신의 로컬에 보관 (.pem 파일).
    - Public Key → AWS가 EC2 인스턴스 안에 자동으로 넣준다. (예: /home/ec2-user/.ssh/authorized_keys)

###  2. Public Key는 어디 있나?
    - AWS는 Key Pair 생성 시 Public Key를 AWS 시스템에 저장한다.
    - 그리고 새 EC2를 띄울 때, 해당 인스턴스의 OS 이미지 초기화 과정에서 Public Key를 자동으로 인스턴스에 삽입한다.
        - Amazon Linux 같은 경우 /home/ec2-user/.ssh/authorized_keys에 기록됨.
        - 만약 root 계정을 바로 열 수 있도록 했다면 /root/.ssh/authorized_keys에 들어감.

###  3. 접속 과정 요약
    1. ssh -i mykey.pem ec2-user@<EC2-IP> 실행
    2. 클라이언트(ssh)는 .pem 안의 Private Key를 사용해 인증 시도.
    3. 서버는 ~/.ssh/authorized_keys에 있는 Public Key와 비교.
    4. 매칭되면 암호학적 검증이 완료 → 로그인 성공.

###  4. 왜 AWS는 Public Key를 안 주나?
    - Public Key는 노출되어도 보안상 큰 문제 없음.
    - 하지만 Private Key는 절대 유출되면 안 되기 때문에, AWS는 Private Key만 1회 다운로드 가능하게 해 준다.
    - Public Key는 AWS 내부에서 이미 관리하고 있기 때문에, 사용자가 따로 가질 필요가 없다.

###  5. 👉 정리하면,
    - .pem = Private Key 전용 파일
    - Public Key는 AWS가 EC2 안에 자동으로 넣어두기 때문에 따로 내려주지 않는다.

###  6. pem 파일에서 Public Key를 직접 추출하는 방법
AWS에서 받은 .pem 파일은 Private Key만 담겨 있지만, 거기서 Public Key를 직접 뽑아낼 수도 있다.
```shell
    ssh-keygen -y -f steventest.pem > steventest.pub
      **********************
    ## 여기서 steventest.pub파일에 생성된 Public는 사실 
    #     ec2-user@[EC2-IP]의 ~.ssh/authorized_keys 에 저장된 Public키들중에 EC2를 생성할때 지정한 키이름에 해당하는 것과 동일하다.
```

----
# PEM @AWS :  RSA vs ED25519
이것은 어떤 공개키 암호화 알고리즘을 사용할지를 정하는 차이다.

###  1. RSA (Rivest–Shamir–Adleman)
    - 가장 오래되고 널리 쓰이는 공개키 알고리즘 (1980년대부터 사용).
    - 키 길이를 2048bit, 4096bit 등으로 조절 가능.

    장점:
    - 호환성이 매우 좋음 (거의 모든 OS, 네트워크 장비, 구버전 소프트웨어에서도 사용 가능).
    - 오랜 검증 역사가 있어 안정성이 입증됨.

    단점:
    - 키 길이가 길어질수록 속도가 느림 (특히 4096bit 이상).
    - 상대적으로 최신 알고리즘에 비해 비효율적임.

###  2. ED25519 (Edwards-curve Digital Signature Algorithm)
    - 비교적 최신 방식의 타원곡선 기반 암호학(ECC) 알고리즘.
    - 키 길이는 항상 256bit (RSA처럼 길이 옵션이 없음).

    장점:
    - 짧은 키 길이에도 RSA 3072~4096bit 수준의 보안 강도 제공.
    - 생성/검증 속도가 빠름.
    - 키 파일 크기가 작음.

    단점:
    - 아주 오래된 SSH 클라이언트/서버에서는 지원하지 않을 수 있음 (하지만 대부분 최신 Linux/Windows/Mac은 지원).
    - RSA만 지원하는 레거시 시스템과 호환성 문제 발생 가능.

###  3. 보안 수준 비교
    - RSA 2048bit ≈ ED25519 (256bit)
    - RSA 3072bit ~ 4096bit ≈ ED25519 (256bit)
👉 ED25519는 훨씬 짧은 키로도 동일한 보안성을 제공

###  4. AWS에서 어떤 걸 선택해야 할까?
    - 레거시/호환성 중시 → RSA (특히 회사 내부에 오래된 장비, 옛날 OpenSSH 버전이 섞여 있다면)
    - 최신 환경, 성능·보안 중시 → ED25519 (추천)
        - 더 안전하고 빠르고, 키 크기도 작아서 관리하기 편리

###  ✅ 정리
    - RSA: 전통적이고 호환성 최고, 하지만 무겁다.
    - ED25519: 최신, 가볍고 빠르고 안전하다. (가능하면 이것 선택 👍)