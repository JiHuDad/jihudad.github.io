### strongSwan 컴포넌트 관계 (S/W 설계 관점)

strongSwan은 단일한 하나의 프로그램이 아니라 데몬과 라이브러리들의 모음입니다. `charon`은 IPsec SA(Security Association)를 관리하고 IKE(Internet Key Exchange) 프로토콜을 담당하는 핵심 데몬입니다. 언급하신 다른 컴포넌트들은 주로 외부 애플리케이션이 `charon` 데몬과 상호작용하는 방식과 관련이 있습니다.

소프트웨어 설계 관점에서 각 컴포넌트를 설명하면 다음과 같습니다.

1.  **`charon` 데몬:** strongSwan의 핵심 프로세스입니다. 사용자 공간(user space)에서 실행되며 다음을 처리합니다.
    * IKE 협상 (Phase 1 및 Phase 2).
    * 터널 및 Security Association (SA) 상태 관리.
    * 운영체제의 IPsec 스택과의 인터페이스 (예: Linux에서는 Netlink, BSD에서는 PF_KEY 사용)를 통해 커널에 SA 및 SP(Security Policy) 규칙 설치.
    * 인증서, 키, 인증 관리.
    * 외부 제어 및 모니터링을 위한 인터페이스 노출.

2.  **strongSwan 내부 라이브러리 (개념적인 `libcharon`):** `charon` 데몬은 strongSwan 프로젝트 내부의 라이브러리 집합 위에 구축됩니다. 이 라이브러리들은 암호화, 네트워킹 추상화, 구성 파싱, 상태 머신 관리, 로깅 등을 처리합니다. 외부 프로그램이 `charon`처럼 동작하기 위해 링크하는 단일하고 명확한 `libcharon.so` 라이브러리가 있는 것은 아닙니다. `charon`을 구성하는 *코드* 자체가 다양한 내부 라이브러리 컴포넌트로 조직화되어 있습니다. `libcharon`이 개념적으로 언급될 때, 이는 strongSwan 프로젝트 *내부*에서 `charon` 데몬을 빌드하기 위해 사용되는 내부 코드 구조와 API를 의미할 가능성이 높습니다. 외부 프로그램은 일반적으로 이러한 내부 라이브러리에 직접 링크하여 `charon`을 제어하지 않습니다.

3.  **VICI (Versatile IKE Configuration Interface):** 외부 애플리케이션과 `charon` 데몬 간의 통신에 사용되는 *프로토콜*입니다. 일반적으로 유닉스 도메인 소켓을 통해 구현되는 메시지 기반 프로토콜입니다. 이를 통해 외부 클라이언트는 명령(예: 연결 로드, 터널 시작, 터널 종료)을 보내고 응답 또는 이벤트를 수신할 수 있습니다.

4.  **`libvici`:** VICI 프로토콜을 구현한 *클라이언트 라이브러리*입니다. 외부 애플리케이션은 `libvici`에 링크하여 VICI 요청을 쉽게 구성하고, VICI 소켓을 통해 `charon`에 전송하며, 응답을 파싱할 수 있습니다. VICI 상호작용을 위한 편리한 C API를 제공합니다.

5.  **DAVICI (Data and VICI? 또는 Daemon VICI?):** VICI에 대한 확장 또는 보완 기능으로, `charon`으로부터 이벤트 스트림 또는 데이터 플레인 관련 특정 상태 업데이트를 수신하는 것과 관련이 있는 것으로 보입니다. *프로토콜* 자체는 동일한 VICI 소켓을 통해 실행될 수 있지만, `libdavici`는 이러한 비동기 알림 및 상태 변경(SA 상태, DPD 이벤트, 트래픽 통계 등)을 수신하는 *클라이언트 측면*에 특별히 초점을 맞춥니다. 명령을 보내고 직접 응답을 받는 데 더 초점을 맞춘 `libvici`를 보완합니다.

**요약하자면:** `charon`은 데몬입니다. `libvici`와 `libdavici`는 외부 애플리케이션이 VICI/DAVICI 프로토콜을 사용하여 소켓 인터페이스를 통해 `charon`과 통신하기 위한 클라이언트 라이브러리입니다. `charon` 내부에서 내부 라이브러리를 사용하는 핵심 로직은 개념적으로 `libcharon`을 지칭할 수 있지만, 이는 외부에서 사용할 수 있는 API는 아닙니다.

### 관계 다이어그램

이러한 관계를 보여주는 컴포넌트 다이어그램입니다.

```mermaid
graph TD
    subgraph "사용자 공간 (User Space)"
        subgraph "외부 애플리케이션 (External Applications)"
            App1[ipsec command] --> LibVICI
            NewApp[신규 프로그램 (libdavici 사용)] --> LibDAVICI
            NewApp --> LibVICI
        end

        LibVICI(libvici) --> ViciSocket[VICI 소켓 (Unix Domain Socket)]
        LibDAVICI(libdavici) --> ViciSocket

        subgraph "strongSwan 데몬 (charon)"
            ViciSocket --> CharonViciServer[VICI/DAVICI 서버 구현]
            CharonViciServer --> CharonCore[Charon 핵심 로직]
            CharonCore --> InternalLibs[strongSwan 내부 라이브러리 (crypto, net, state 등)]
        end

        CharonCore --> KernelInterface[커널 IPsec 인터페이스 (Netlink/PF_KEY)]
    end

    subgraph "커널 공간 (Kernel Space)"
        KernelInterface --> KernelIPsec[OS 커널 IPsec 스택]
    end

    KernelIPsec -- 암호화/복호화 트래픽 --> NetworkInterface[(네트워크 인터페이스)]

    %% Relationship descriptions
    CharonCore -.-> |SA/SP 관리| KernelIPsec
    CharonViciServer -.-> |명령 수신/이벤트 전송| ViciSocket
    LibVICI -.-> |명령 전송, 응답 수신| ViciSocket
    LibDAVICI -.-> |이벤트/상태 업데이트 수신| ViciSocket
    NewApp -.-> |API 사용| LibDAVICI
    NewApp -.-> |API 사용 (선택 사항)| LibVICI
    App1 -.-> |API 사용| LibVICI
    CharonCore -.-> |기반으로 구축됨| InternalLibs
```

**다이어그램 설명:**

* **외부 애플리케이션:** `strongSwan`과 상호작용하는 사용자 공간 프로그램을 나타냅니다. 여기에는 `ipsec` 명령줄 유틸리티와 같은 표준 도구 및 **신규 프로그램**이 포함됩니다.
* **`libvici` 및 `libdavici`:** 외부 애플리케이션에 의해 링크되는 클라이언트 라이브러리입니다. VICI/DAVICI 프로토콜의 세부 사항을 추상화합니다.
* **VICI 소켓:** 통신 채널로, 일반적으로 유닉스 도메인 소켓(` /var/run/charon.vici`가 일반적)입니다. 외부 클라이언트는 이 소켓에 연결합니다.
* **`charon` 데몬:** strongSwan의 주요 프로세스입니다.
    * **VICI/DAVICI 서버 구현:** VICI 소켓에서 수신 대기하고, 들어오는 클라이언트 연결을 처리하며, VICI 명령을 처리하고, 비동기 DAVICI 이벤트를 전송하는 `charon` 내부 컴포넌트입니다.
    * **Charon 핵심 로직:** IKE 상태 머신, SA/SP 데이터베이스, 인증 등을 관리하는 데몬의 핵심입니다.
    * **strongSwan 내부 라이브러리:** 암호화, 네트워킹, 파싱, 상태 머신 등 Charon 핵심 로직이 구축된 기반 코드 모듈입니다. 개념적인 `libcharon`이 내부적으로 적용되는 부분입니다.
* **커널 IPsec 인터페이스:** `charon`은 OS별 인터페이스(예: Linux의 Netlink)를 사용하여 커널의 IPsec 스택을 프로그래밍하여 패킷을 암호화/복호화하고 어떤 트래픽을 보호/우회할지 지시합니다.
* **OS 커널 IPsec 스택:** 실제로 네트워크 패킷에 대한 IPsec 처리(암호화, 복호화, 인증, SP 필터링)를 수행하는 운영체제 커널 부분입니다.
* **네트워크 인터페이스:** 트래픽이 시스템에 들어오고 나가는 곳입니다.

다이어그램은 외부 애플리케이션이 VICI 소켓을 통해 통신하는 `libvici` 및 `libdavici`를 통해 `charon`과 간접적으로 상호작용함을 보여줍니다. `charon` 자체는 커널과 상호작용하여 IPsec 정책을 적용합니다.

### 신규 프로그램 설계 문서

`libdavici` (및 잠재적으로 `libvici`)를 사용하여 `strongSwan`과 상호작용하고 DPDK를 구성하는 신규 프로그램에 대한 설계 문서 개요입니다.

---

**문서 제목:** strongSwan-DPDK IPsec 상태 동기화 프로그램

**버전:** 1.0

**날짜:** 2023-10-27

**1. 서론**

본 문서는 `strongSwan` 데몬(`charon`)으로부터 IPsec SA(Security Association) 및 SP(Security Policy) 상태, 그리고 DPD(Dead Peer Detection) 상태를 동기화하여 DPDK(Data Plane Development Kit) 기반 패킷 처리 애플리케이션을 구성하는 새로운 사용자 공간 프로그램의 설계를 설명합니다. 이 프로그램은 주로 `libdavici` 라이브러리를 사용하여 `charon`으로부터 비동기 상태 업데이트 및 이벤트를 수신합니다.

**1.1. 목적**

주요 목적은 IPsec 제어부(`strongSwan`이 처리)와 DPDK를 사용하여 구현된 고성능 사용자 공간 데이터부를 분리하는 것입니다. 이 프로그램은 `strongSwan`이 관리하는 IPsec 상태를 DPDK 애플리케이션의 IPsec 처리 모듈을 위한 구성 매개변수로 변환하는 브릿지 역할을 합니다.

**1.2. 범위**

* `charon` VICI/DAVICI 소켓에 연결.
* `libdavici` 초기화 및 이벤트 수신 설정.
* SA 상태 변경 이벤트(CREATED, UPDATED, REKEYING, DELETING, DELETED) 수신 및 파싱.
* DPD 이벤트(예: strongSwan의 DPD를 통한 피어 연결 불가능 알림) 수신 및 파싱.
* (선택 사항이나 권장) 프로그램 시작 또는 재연결 시 `libvici`를 사용하여 초기 SA/SP 상태 쿼리.
* strongSwan SA/SP 매개변수(SPI, 키, 알고리즘, 주소, 정책)를 DPDK 애플리케이션의 IPsec 모듈이 사용할 수 있는 형식으로 변환.
* DPDK 애플리케이션의 구성 API와 상호작용하여 SA 및 SP 설치, 업데이트, 제거.
* DPD 상태 관련 이벤트 처리 및 그에 따라 DPDK 상태 업데이트(예: SA를 비활성으로 표시).
* `charon`과의 통신 문제 및 DPDK 구성 실패에 대한 오류 처리 구현.

**1.3. 범위 외**

* IPsec 프로토콜 로직 자체 구현(이는 `strongSwan`의 역할).
* DPDK 패킷 처리 로직 구현(이는 별도의 DPDK 애플리케이션 역할).
* 완전한 `strongSwan` 구성 도구 역할(상태 동기화에 초점, 필요한 경우 제한적인 제어를 위해 `libvici` 사용 가능).
* DPD 협상 또는 능동적 DPD 시작(이는 `strongSwan`의 역할).

**2. 아키텍처**

프로그램은 이벤트 기반 아키텍처를 따르며, 별도의 데몬 프로세스로 실행됩니다.

```mermaid
graph TD
    subgraph "OS 사용자 공간 (OS User Space)"
        NewProgram[신규 프로그램 (strongSwan-DPDK Sync)] --> LibDAVICI(libdavici)
        NewProgram --> LibVICI(libvici - 초기 상태 쿼리 옵션)
        NewProgram --> DPDK_Config_API[DPDK 애플리케이션 구성 API]

        LibDAVICI --> ViciSocket[VICI 소켓]
        LibVICI --> ViciSocket

        ViciSocket --> Charon[strongSwan 데몬 (charon)]

        DPDK_Config_API --> DPDK_App[DPDK 애플리케이션 (패킷 처리)]
    end

    Charon -.-> |IPsec 상태 관리| NewProgram
    NewProgram -.-> |IPsec 상태 구성| DPDK_App
```

**2.1. 상위 수준 컴포넌트**

* **DAVICI 클라이언트 모듈:** VICI 소켓을 통해 `charon`에 연결하고 `libdavici`를 사용하여 비동기 이벤트를 수신하는 역할.
* **VICI 쿼리 모듈 (선택 사항):** 연결 성공 시 초기 상태 동기화 및 필요한 경우 상태 재쿼리 역할을 `libvici`를 사용하여 수행.
* **이벤트 처리 모듈:** `libdavici` (및 잠재적으로 `libvici` 쿼리)로부터 수신된 이벤트를 파싱하고 관련 IPsec SA/SP 및 DPD 정보를 추출.
* **DPDK 동기화 모듈:** 추출된 IPsec 상태 정보를 DPDK 애플리케이션의 구성 API를 위한 명령으로 변환. strongSwan SA/SP와 DPDK 표현 간의 매핑 관리.
* **주요 애플리케이션 로직:** 모든 모듈을 초기화하고, 이벤트 루프를 관리하며, 시그널 처리를 담당하고, 시작/종료를 조율.

**3. 상세 설계**

**3.1. DAVICI 클라이언트 모듈**

* **초기화:**
    * `vici_init()` 호출 ( `libdavici`는 VICI 계열이므로).
    * `charon` VICI 소켓(설정 가능한 경로)에 연결.
    * `libdavici`의 등록 함수를 사용하여 이벤트 핸들러 설정. 등록할 주요 이벤트는 다음과 같습니다.
        * `list-sas`: 일반적으로 요청/응답이지만, strongSwan은 SA 상태 *변경*을 비동기적으로 유사한 구조 또는 전용 이벤트 이름(예: `sa-up`, `sa-down`, `sa-rekey` 등 SA 상태 변경 관련 이벤트의 정확한 이름은 strongSwan VICI 문서를 확인)으로 전송하기도 합니다.
        * `peer-unreachable` 또는 유사한 DPD 관련 이벤트.
        * `childsa-up`, `childsa-down`.
* **이벤트 루프:**
    * VICI 소켓에서 이벤트를 기다리는 루프 실행.
    * 루프 내에서 `vici_read_event()` 또는 유사한 `libdavici` 함수 사용.
    * 수신된 이벤트를 이벤트 처리 모듈로 전달.
* **재연결:** `charon`과의 연결 끊김을 감지하고 지수 백오프를 사용하여 재연결을 시도하는 로직 구현. 재연결 성공 시 VICI 쿼리 모듈을 통해 전체 상태 동기화 트리거.

**3.2. VICI 쿼리 모듈 (선택 사항이나 권장)**

* **초기화 쿼리:** `charon`에 성공적으로 연결되면(시작 시 및 재연결 후), `libvici`를 사용하여 `list-sas` 요청을 보내 현재 활성 상태인 모든 SA 목록을 가져옵니다.
* **상태 파싱:** `list-sas` 응답을 파싱하여 활성 IPsec 상태의 초기 뷰 구축.
* **전달:** 파싱된 초기 상태를 DPDK 동기화 모듈로 전달.

**3.3. 이벤트 처리 모듈**

* **입력:** DAVICI 클라이언트 모듈로부터 수신된 원시 이벤트 메시지.
* **파싱:** `libdavici`의 파싱 함수(`vici_parse_chunk` 등)를 사용하여 이벤트 메시지에서 키-값 쌍 추출.
* **이벤트 유형 처리:** 이벤트 이름(예: `sa-up`, `sa-down`, `peer-unreachable`)에 따라 특정 데이터 추출:
    * **SA 이벤트:** 소스/대상 IP 주소, SPI(인바운드/아웃바운드), IPsec 프로토콜(ESP/AH), SA 상태, 암호화 알고리즘, 인증 알고리즘, 키 자료(노출되고 필요한 경우, 일반적으로 추상화됨), 터널/트랜스포트 모드.
    * **DPD 이벤트:** 도달 불가능한 피어와 관련된 피어 식별자 또는 IP 주소. 해당 SA에 매핑할 수 있습니다.
* **출력:** IPsec 상태 변경 또는 DPD 이벤트를 나타내는 구조화된 데이터, DPDK 동기화 모듈로 전달.

**3.4. DPDK 동기화 모듈**

* **상태 관리:** DPDK 애플리케이션에 구성된 IPsec SA 및 SP의 로컬 표현을 유지합니다. 이를 통해 어떤 strongSwan SA가 어떤 DPDK 표현에 해당하는지 추적할 수 있습니다. strongSwan SA 식별자(예: 피어 IP, SPI 및 프로토콜 조합)를 DPDK 내부 식별자에 매핑하는 맵 또는 해시 테이블이 유용할 것입니다.
* **변환 로직:**
    * strongSwan SA 매개변수(알고리즘, 키, SPI, 주소)를 DPDK 애플리케이션의 구성 API가 요구하는 형식으로 변환.
    * strongSwan SP(셀렉터 세부 정보)를 DPDK 플로우 규칙 또는 이와 동등한 구조로 변환.
* **DPDK API 상호작용:**
    * **SA 생성/업데이트:** `sa-up` 또는 `sa-rekey` 이벤트가 수신되면 해당 SA를 DPDK 애플리케이션에 추가 또는 업데이트하기 위해 DPDK API 호출.
    * **SA 삭제:** `sa-down` 또는 `sa-deleted` 이벤트가 수신되면 해당 SA를 제거하기 위해 DPDK API 호출.
    * **DPD 처리:** `peer-unreachable` 이벤트가 수신되면 영향을 받는 SA 식별 (피어를 SA에 매핑). 도달 불가능한 피어로 인해 SA를 더 이상 사용할 수 없음을 나타내기 위해 해당 DPDK API 함수 호출. 구체적인 동작은 DPDK 애플리케이션의 기능에 따라 달라집니다.
    * **SP 관리:** strongSwan 상태 변경에 따라 DPDK에 SP를 추가/제거 (비동기 이벤트로는 덜 일반적이며, 초기 `list-sas` 또는 특정 구성 변경에 더 의존할 수 있음).
* **오류 처리:** DPDK 구성 API에서 반환되는 잠재적 오류(예: 잘못된 매개변수, 리소스 고갈) 처리. 오류를 로깅하고, 적절한 메커니즘이 있다면 `strongSwan`에 보고하거나 단순히 재시도/실패 처리.

**3.5. 주요 애플리케이션 로직**

* **초기화:**
    * 명령줄 인수 또는 구성 파일 파싱 (예: VICI 소켓 경로, 로깅 레벨, DPDK 특정 매개변수).
    * 로깅 초기화.
    * DAVICI 클라이언트 모듈 초기화.
    * DPDK 동기화 모듈 초기화 (DPDK 애플리케이션의 API에 대한 연결 초기화를 포함할 수 있음).
    * (선택 사항) VICI 쿼리 모듈 초기화 및 초기 상태 동기화 수행.
* **이벤트 루프:** 주로 소켓 이벤트를 기다리는 DAVICI 클라이언트 모듈에 의해 구동되는 주요 이벤트 루프 진입.
* **시그널 처리:** SIGINT, SIGTERM 등의 시그널에 대한 핸들러 구현하여 정상적인 종료 처리. 종료 시 DPDK 동기화 모듈에게 DPDK에 설치된 SA/SP 정리를 지시하고 연결 종료.
* **오류 관리:** 중앙 집중식 오류 보고 및 처리 메커니즘 구현.

**4. 데이터 구조**

* strongSwan으로부터 수신된 파싱된 SA 매개변수를 저장하기 위한 구조체.
* 파싱된 DPD 이벤트 정보를 저장하기 위한 구조체.
* DPDK 동기화 모듈 내에서 strongSwan SA 식별자를 DPDK 특정 SA 핸들/식별자에 매핑하기 위한 내부 구조체/맵.
* 프로그램 설정을 저장하기 위한 구성 구조체.

**5. 오류 처리**

* **연결 오류:** VICI 소켓 연결 실패 및 연결 끊김 처리. 재시도 로직 구현.
* **VICI/DAVICI 프로토콜 오류:** `charon`으로부터의 잘못된 형식의 메시지 또는 오류 응답 처리. 이 오류들을 로깅.
* **이벤트 파싱 오류:** 예상치 못한 이벤트 형식 처리. 로깅하고 이벤트를 무시할 수 있음.
* **DPDK 구성 오류:** DPDK 애플리케이션의 API 호출 실패 처리. 오류 로깅, 잠재적 재시도, 또는 특정 SA/SP를 동기화 실패로 표시.
* **자원 관리:** 오류 또는 종료 시 VICI 연결, DPDK 자원, 메모리 등의 적절한 정리 보장.

**6. DPD 처리 세부 사항**

* `strongSwan`의 DPD 메커니즘은 `charon` 데몬 내에서 실행됩니다. `charon`이 피어에 도달할 수 없다고 감지하면(DPD keepalives 기반), VICI/DAVICI 인터페이스를 통해 특정 이벤트(예: `peer-unreachable`)를 트리거합니다.
* 신규 프로그램의 이벤트 처리 모듈은 이 이벤트를 특별히 수신 대기합니다.
* `peer-unreachable` 이벤트를 수신하면 이벤트 처리 모듈은 피어 또는 영향을 받는 SA를 식별하는 정보를 추출합니다.
* DPDK 동기화 모듈은 이 정보를 사용하여 DPDK 애플리케이션에 구성된 해당 SA를 찾습니다.
* DPDK 동기화 모듈은 피어 도달 불가능으로 인해 해당 SA를 더 이상 사용할 수 없음을 나타내기 위해 적절한 DPDK API 함수를 호출합니다. 여기에는 다음이 포함될 수 있습니다.
    * DPDK의 내부 상태에서 SA를 '다운' 또는 '비활성'으로 표시.
    * DPDK가 이 SA를 사용하여 아웃바운드 트래픽을 시도하는 것을 방지.
    * (DPDK 애플리케이션 설계에 따라) strongSwan이 피어가 계속 도달 불가능할 경우 결국 SA를 삭제하므로 인바운드 상태도 지울 수 있습니다.
* 중요하게, 신규 프로그램은 DPD 자체를 수행하지 않습니다. `strongSwan`이 보고한 DPD 결과에 반응합니다.

**7. 구성**

프로그램은 다음 매개변수에 대해 명령줄 인수 또는 구성 파일을 통해 구성 가능해야 합니다.

* `charon` VICI 소켓 경로.
* 로깅 레벨 및 출력 대상.
* DPDK 애플리케이션의 구성 API에 연결/초기화하는 데 필요한 매개변수.

**8. 향후 개선 사항**

* SP(Security Policy)를 더 동적으로 동기화하는 지원.
* strongSwan으로부터의 트래픽 통계 이벤트(DAVICI를 통해 사용 가능한 경우) 처리 및 DPDK 업데이트 또는 보고.
* 더 정교한 오류 복구 및 상태 조정 메커니즘.

---
