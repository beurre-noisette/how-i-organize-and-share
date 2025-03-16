# GitLab, 0x80090325 오류

# 개요

최초 개발환경 세팅 간 `IntelliJ`에서 `GitLab`의 레포지토리를 클론하려고 시도하였을 때 `unable to access 'https://gitlab.mng.example.com,/project-name/repo-name.git/': schannel: SEC_E_UNTRUSTED_ROOT (0x80090325)`과 같은 오류가 발생했습니다. 이 문서는 해당 오류의 원인을 파악하고 이를 해결하는 방법을 서술합니다.

---

# 환경

1. OS : Windows11
2. Git : 로컬 PC에 설치된 Git
3. GitLab : 내부망에서 운영중[^1]
4. IDE : IntelliJ IDEA

---

# 문제상황

내부망의 `GitLab` 레포지토리에서 소스를 클론하려고 할 때 다음과 같은 오류 메세지가 등장합니다.

```text
Clone failed

unable to access 'https://gitlab.mng.example.com,/project-name/repo-name.git/': schannel: SEC_E_UNTRUSTED_ROOT (0x80090325)
```

==이 오류는 Git이 GitLab의 SSL 인증서를 신뢰하지 않아서 발생합니다.==

---

# 원인분석

+ `Git`은 HTTPS 연결 시 기본적으로 Windows의 `Secure Channel("schannel")`을 사용하여 SSL 인증서를 검증합니다.[^2]
+ 내부망 `GitLab` 서버에서 사용하는 SSL 인증서가 Windows 인증서 저장소(로컬 PC)에 등록되어 있지 않거나, 체인이 불완전하여 신뢰되지 않으면 검증에 실패합니다.
+ 따라서 `SEC_E_UNTRUSTED_ROOT` 오류를 발생시킵니다.

---

# 해결방법

## Git SSL 백엔드를 OpenSSL로 변경

Windows의 기본 `schannel` 대신 `OpenSSL`을 이용하면, `Git`이 `OpenSSL` 라이브러리의 CA[^3]번들 또는 지정된 인증서를 사용하여 검증하게 됩니다.

```bash
git config --global http.sslBackend openssl
```

위  설정을 통해 `Git`이 `schannel` 대신 `OpenSSL`을 활용할 수 있게 전역설정을 할 수 있습니다.

## 그런데 인증서 검증 방식이 `schannel`에서 `OpenSSL`로 바뀌었다고 왜 해결이 된건가요?

이러한 의문을 해결하기 위해서는 `schannel`과 `OpenSSL`이 SSL 인증서를 검증하는 방식에 차이가 있다는 점을 알아야 합니다.

### `Schannel(Windows Secure Channel)`

+ `Schannel`은 Windows 운영체제의 네이티브 SSL/TLS 구현입니다.
+ **Windows 인증서 저장소(Certificate Store)**를 사용하여 루트 및 중간 인증서를 검증합니다.
+ 만약 서버가 제공한 SSL 체인에 중간 인증서가 누락되었거나, 루트 CA가 Windows 인증서 저장소에 등록되어 있지 않다면, Schannel은 검증에 실패하고 오류를 반환합니다.
+ `Schannel`은 기본적으로 클라이언트(로컬 PC)가 신뢰 저장소에 등록된 인증서만 신뢰합니다. 자체적으로 추가적인 CA번들을 사용하지 않습니다.

### `OpenSSL`

+ `OpenSSL`은 독립적인 SSL/TLS 라이브러리로, Windows 인증서 저장소를 사용하지 않습니다.
+ 대신 `OpenSSL`은 자체적으로 제공하는 **CA번들 또는 `Git` 설정에서 지정된 사용자 정의 CA 파일을 기반으로 인증서를 검증합니다.**
+ 서버가 중간 인증서를 누락했더라도, `OpenSSL`은 자체적으로 체인을 완성하거나 더 유연하게 처리할 수 있습니다.

`Schannel`과 `OpenSSL`의 차이를 알았다면 이제 왜 `OpenSSL`로 변경했을 때 문제상황이 해결되었는지 살펴보겠습니다.

1. 중간 인증서 처리 방식의 차이에서 기인합니다.
	+ `Schannel은 서버가 제공하는 SSL 체인만을 신뢰하며, 서버가 중간 인증서를 누락한 경우 이를 복구하지 못합니다. (이번 문제의 경우 여기에 해당했습니다.)
	+ 반면 `OpenSSL`은 자체 CA 번들 파일에 포함된 루트 및 중간 인증서를 활용하여 누락된 체인을 보완할 수 있습니다.
2. 루트 CA 신뢰 문제일 수 있습니다.
	+ 내부망 GitLab이 사용하는 루트 CA가 Windows의 `Schannel`에서는 신뢰되지 않았지만,  `OpenSSL` 에서는 기본적으로 제공되는 CA 번들에 포함되어 있었을 가능성이 있습니다.


---

# 결론

문제상황을 보면 내부망의 GitLab 서버가 제공한 SSL 체인이 불완전하거나 잘못 구성되어 있었을 것이라고 생각합니다.[^4] 기본적으로 설정된 `Schannel`은 Windows 인증서 저장소만 참조하기 때문에 이 문제를 해결하지 못했고 결과적으로 `unable access`라는 오류가 발생했습니다. 반면 우리가 설정한 `OpenSSL`은 자체 CA 번들을 사용하거나 더 유연한 방식으로 체인을 검증하여 문제를 해결할 수 있기 때문에 이번 문제를 해결할 수 있었다고 생각합니다.

---

# 문제해결 방법으로써 추가로 고려해볼 수 있는 방법들

## 방법 1: SSL 검증 비활성화 (비추천)

임시로 SSL 검증을 비활성화하면 오류를 우회할 수 있지만, 보안상 취약점이 생기므로 권장되지 않습니다.

```bash
# 인증서 검증 비활성화
git config --global http.sslVerify false

# 작업 후 인증서 검증 재활성화
git config --global http.sslVerify true
```

## 방법 2: 자체 인증서를 Git에 등록

내부망 GitLab 서버의 SSL 인증서를 다운로드한 후, Git이 해당 인증서를 신뢰하도록 설정합니다.

```bash
# SSL 인증서를 준비한 뒤
git config --global http.sslCAInfo /path/to/your-certificate.pem
```

## 방법 3: SSH 프로토콜 사용

HTTPS 대신 SSH 방식을 사용하면 SSL 인증서 검증 과정을 생략할 수 있습니다. SSH 키를 생성한 후 GitLab에 등록하여 SSH URL로 레포지토리를 클론합니다.

```bash
# 1. SSH 키 생성
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# GitLab에 SSH 키 등록 후 클론:
git clone git@gitlab.example.com:group/project-name.git
```

---

# 참고

1. https://m.blog.naver.com/alice_k106/221468341565
2. https://youtrack.jetbrains.com/issue/IJPL-84948/fatal-unable-to-access-https-gitlab.com-project.git-schannel-next-InitializeSecurityContext-failed-SECEUNTRUSTEDROOT-0x80090325
3. docs.gitlab.com/omnibus/settings/ssl/ssl_troubleshooting/

[^1]: 자체 서명 인증서 또는 신뢰할 수 없는 루트 인증서를 사용할 가능성이 있습니다.
[^2]: 오류 내용을 살펴보면 `schannel: SEC_E_UNTRUSTED_ROOT`라는 부분이 보입니다.
[^3]: Certificate Authority
[^4]: 예를 들어, Git이 사용하는 `OpenSSL`의 기본 CA 번들(`ca-bundle.crt)`에는 공인된 루트 및 중간 CA목록이 포함되어 있어서 GitLab 서버에서 누락된 중간 인증서를 보완할 가능성이 높다고 판단합니다.
