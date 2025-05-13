strongSwan에서 인증서 기반(IPsec/IKEv2) 인증을 사용할 때, **개인키(Private Key)의 전달 및 보호**는 다음과 같이 처리됩니다.

---

### 1. 개인키의 전달 방식

- **로컬 파일**: 가장 일반적으로, 개인키는 strongSwan이 실행되는 시스템의 파일(예: `/etc/ipsec.d/private/`)에 저장됩니다.
- **스마트카드/TPM/HSM**: 개인키를 외부 하드웨어(스마트카드, TPM, HSM 등)에 저장하고, strongSwan이 PKCS#11, TPM, 또는 기타 플러그인을 통해 접근할 수 있습니다.
- **OS/플랫폼 키체인**: macOS, Windows 등에서는 OS의 키체인/키스토어를 통해 개인키를 관리할 수 있습니다.

**중요:** 개인키는 네트워크를 통해 strongSwan에 "전달"되는 것이 아니라, strongSwan 프로세스가 직접 접근할 수 있는 위치(파일, HSM 등)에 있어야 합니다.

---

### 2. 개인키의 암호화/복호화 및 안전한 사용

#### (A) 파일에 저장된 개인키

- strongSwan은 PEM/DER 형식의 개인키 파일을 읽을 수 있습니다.
- **암호화된 개인키(Encrypted Private Key)**:  
  - 개인키 파일이 암호화되어 있다면(예: 암호로 보호된 PEM), strongSwan은 부팅 시 또는 연결 시 암호(passphrase)를 입력받아 복호화합니다.
  - 암호 입력은 `ipsec` 명령어를 통해 수동으로 하거나, `charon` 데몬이 `pin`/`pass` 프롬프트를 통해 받을 수 있습니다.
- **암호화/복호화 로직**:  
  - strongSwan의 `libstrongswan` 라이브러리가 OpenSSL, gcrypt, wolfSSL 등 다양한 암호화 백엔드를 통해 암호화된 개인키를 복호화합니다.
  - 복호화된 개인키는 메모리 내에서만 사용되며, 디스크에 평문으로 저장되지 않습니다.

#### (B) 하드웨어 보안 모듈(HSM/TPM/스마트카드)

- strongSwan은 PKCS#11, TPM, Windows CAPI, macOS Keychain 등 다양한 플러그인을 통해 외부 장치에 저장된 개인키를 사용할 수 있습니다.
- 이 경우, 개인키는 strongSwan 프로세스의 메모리로 **직접 복사되지 않고**, 서명 연산 등만 HSM/TPM에서 수행됩니다.
- PIN/비밀번호 등은 strongSwan이 사용자로부터 입력받거나, 환경변수/설정파일을 통해 전달받을 수 있습니다.

#### (C) 안전한 사용

- strongSwan은 개인키를 메모리 내에서만 사용하고, 필요 이상으로 오래 보관하지 않도록 설계되어 있습니다.
- 암호화된 개인키 파일을 지원하며, HSM/TPM을 통한 키 보호도 지원합니다.
- strongSwan의 플러그인(`libstrongswan/plugins/`) 중 `pkcs11`, `tpm`, `agent`, `keychain`, `winhttp` 등이 키 보호와 관련된 기능을 제공합니다.

---

### 3. 관련 strongSwan 설정 예시

**암호화된 개인키 사용 예시:**
```conf
# /etc/ipsec.secrets
: RSA mykey.pem
```
- `mykey.pem`이 암호화되어 있으면, 연결 시 암호를 입력하라는 프롬프트가 뜹니다.

**PKCS#11(HSM) 사용 예시:**
```conf
# /etc/ipsec.secrets
: PIN "1234"
: PKCS11 "pkcs11:object=mykey;type=private"
```

---

### 4. 결론

- **개인키는 strongSwan이 직접 접근할 수 있는 위치(파일, HSM 등)에 있어야 하며, 네트워크를 통해 전달하지 않습니다.**
- **암호화된 개인키 파일, HSM/TPM, OS 키체인 등 다양한 안전한 저장/사용 방법을 strongSwan이 지원합니다.**
- **복호화 및 안전한 사용은 strongSwan의 내부 라이브러리와 플러그인에서 처리합니다.**

---
