
strongSwan의 VRF(Virtual Routing and Forwarding) 지원 및 extsock 플러그인 관련 질문에 대해 아래와 같이 정리할 수 있습니다.

---

### 1. strongSwan의 VRF 지원 여부

- **코드 내 직접적인 'VRF' 지원**:  
  strongSwan 소스코드 내에서 'VRF'라는 키워드, 관련 함수(setvrf 등), 설정 옵션 등은 **직접적으로 존재하지 않습니다**.
- **SO_BINDTODEVICE 사용**:  
  `src/libcharon/network/pf_handler.c`에서 `setsockopt(fd, SOL_SOCKET, SO_BINDTODEVICE, ...)`를 사용하여 소켓을 특정 네트워크 인터페이스에 바인딩하는 기능이 있습니다.  
  리눅스에서 VRF는 일반적으로 별도의 인터페이스(예: `vrf-blue`)로 노출되므로, 이 기능을 통해 strongSwan의 트래픽을 특정 VRF 인터페이스에 바인딩할 수 있습니다.
- **child config의 interface 옵션**:  
  `src/libcharon/config/child_cfg.c` 및 관련 헤더를 보면, CHILD_SA 단위로 `interface`(문자열) 필드를 통해 정책을 특정 인터페이스에 제한할 수 있습니다.  
  실제로 `swanctl` 등에서 connection의 child에 `interface = vrf-blue`와 같이 지정하면, 해당 인터페이스(즉, VRF)에만 정책이 적용됩니다.

#### 결론  
- strongSwan은 **직접적으로 'VRF'라는 개념을 인식하지 않지만**,  
- 리눅스 커널의 VRF가 네트워크 인터페이스로 노출되는 구조를 활용하여,  
- **CHILD_SA의 interface 옵션** 또는 내부적으로 SO_BINDTODEVICE를 통해  
- **VRF 인터페이스에 정책/트래픽을 바인딩하는 방식으로 VRF 환경을 지원**할 수 있습니다.

---

### 2. extsock 플러그인에서 VRF(인터페이스) 설정 가능 여부

- **extsock 플러그인**은 외부 프로그램이 strongSwan에 JSON으로 동적으로 설정을 전달할 수 있게 해주는 플러그인입니다.
- 현재 구현된 JSON 예시(README.md 등)에는 `interface`나 `vrf` 관련 필드는 없지만,
- strongSwan의 child config가 `interface` 필드를 지원하므로,
- **extsock 플러그인에서 전달하는 JSON에 `interface` 필드를 추가**하고,  
  이를 strongSwan의 child config 생성 시 반영하도록 구현하면  
  **VRF 인터페이스 지정이 가능합니다**.

#### 예시 (확장 JSON)
```json
{
  "name": "vpn-conn1",
  "local": "192.168.1.10",
  "remote": "203.0.113.5",
  "interface": "vrf-blue",   // <--- VRF 인터페이스 지정
  ...
}
```
- 실제 적용은 extsock의 `apply_ipsec_config()`에서 해당 필드를 파싱해 child_cfg 생성 시 넘겨주면 됩니다.

---

## 요약

- strongSwan은 리눅스의 VRF를 **'인터페이스'로 인식**하여,  
  CHILD_SA의 `interface` 옵션 및 SO_BINDTODEVICE를 통해 **VRF 환경을 지원**합니다.
- extsock 플러그인에서도 JSON에 `interface` 필드를 추가하면 **VRF 인터페이스 지정이 가능합니다**.
- 단, extsock의 실제 strongSwan 설정 적용 로직은 "TODO" 상태이므로,  
  VRF 지원을 위해서는 해당 부분의 구현이 필요합니다.

---

**VRF별로 strongSwan 데몬을 여러 개 띄울 필요는 없습니다.**  
아래와 같이 동작합니다.

---

## 1. VRF별 strongSwan 데몬 여러 개?  
- **필요 없음.**
- strongSwan(charon)은 **하나의 데몬**이 여러 VRF(인터페이스)에 걸친 터널을 동시에 관리할 수 있습니다.
- 리눅스 커널의 VRF는 네트워크 인터페이스(예: `vrf-blue`, `vrf-red`)로 노출되므로,  
  strongSwan의 child 설정에서 `interface = vrf-blue`처럼 지정하면  
  해당 정책/SA가 그 인터페이스(VRF)에만 적용됩니다.

---

## 2. 실제 동작 방식  
- **하나의 strongSwan 데몬**에서 여러 VRF 인터페이스를 대상으로 각각의 터널을 설정할 수 있습니다.
- 예시(swanctl.conf):
    ```ini
    connections {
      blue {
        ...
        children {
          net {
            ...
            interface = vrf-blue
          }
        }
      }
      red {
        ...
        children {
          net {
            ...
            interface = vrf-red
          }
        }
      }
    }
    ```
- 위와 같이 설정하면,  
  - `blue` 터널은 `vrf-blue` 인터페이스(VRF)에만 정책/SA가 설치되고,  
  - `red` 터널은 `vrf-red` 인터페이스(VRF)에만 설치됩니다.

---

## 3. strongSwan의 동작 원리  
- strongSwan은 커널에 정책/SA를 설치할 때,  
  `interface` 옵션이 있으면 해당 인터페이스에만 정책을 제한합니다.
- 리눅스 커널의 VRF는 인터페이스 단위로 트래픽을 분리하므로,  
  strongSwan의 `interface` 옵션만으로 VRF별 트래픽 분리가 가능합니다.

---

## 4. 결론  
- **하나의 strongSwan 데몬**에서 여러 VRF를 동시에 지원할 수 있습니다.
- 각 child(터널) 설정에 `interface = <vrf명>`만 지정하면 됩니다.
- VRF별로 strongSwan을 여러 개 띄울 필요가 없습니다.

---

### 참고
- 단, IKE 패킷(UDP 500/4500)이 어떤 VRF로 들어오는지에 따라 strongSwan이 해당 인터페이스에서 패킷을 수신할 수 있어야 합니다.
- 일반적으로 strongSwan은 모든 인터페이스에서 IKE 패킷을 수신하도록 동작합니다.

---


DPDK와 같은 외부 data plane을 사용할 때 strongSwan의 커널 데이터 플레인(암호화/복호화)을 **off(비활성화)** 하는 방법에 대해 정리합니다.

---

## 1. strongSwan의 기본 구조

- strongSwan은 **IKE(제어 plane)**와 **IPsec(데이터 plane)**을 모두 관리합니다.
- 일반적으로 **커널 모듈(예: XFRM, PF_KEY 등)**을 통해 SA/정책을 커널에 설치하고,  
  커널이 실제 암호화/복호화(데이터 플레인)를 수행합니다.

---

## 2. DPDK 등 외부 data plane만 쓸 때, strongSwan의 암호화/복호화 off 방법

### (1) 정책/SA를 커널에 **설치하지 않기**
- **child 설정에 `mode = pass` 또는 `mode = drop`**  
  → 트래픽을 암호화하지 않거나, 정책만 설치하고 SA는 설치하지 않음(실제 암호화/복호화 없음).
- **child 설정에 `install_policy = no`**  
  → 정책 자체를 커널에 설치하지 않음(트래픽이 암호화되지 않음).

### (2) `OPT_NO_POLICIES` 옵션 사용
- `swanctl.conf` 등에서 child에 다음과 같이 설정:
    ```ini
    children {
      mychild {
        ...
        install_policy = no
      }
    }
    ```
- 또는 legacy ipsec.conf에서는:
    ```
    auto=add
    ...
    installpolicy=no
    ```

### (3) strongSwan의 "libipsec" 소프트웨어 data plane
- strongSwan은 **libipsec** 플러그인을 통해 유저스페이스에서 암호화/복호화를 할 수 있지만,  
  DPDK와 같은 외부 data plane을 쓸 때는 **libipsec도 로드하지 않아야** 합니다.

### (4) extsock/DPDK 연동 시
- extsock 등으로 IKE 협상만 strongSwan이 하고,  
  실제 SA/정책은 DPDK가 직접 관리한다면,  
  **child에 `install_policy = no`를 반드시 명시**해야 커널에 정책/SA가 설치되지 않습니다.

---

## 3. 결론

- **strongSwan에서 data plane 암호화/복호화를 완전히 off하려면**  
  **각 child에 `install_policy = no`(혹은 `installpolicy=no`)를 설정**하세요.
- 그러면 strongSwan은 IKE 협상만 하고,  
  커널에 SA/정책을 설치하지 않으므로,  
  DPDK 등 외부 data plane이 모든 암호화/복호화를 담당할 수 있습니다.

---

### 참고
- 이 설정을 하지 않으면, strongSwan이 커널에 정책/SA를 설치하여  
  커널이 암호화/복호화를 시도할 수 있습니다(원하지 않는 동작).
- `install_policy = no`는 **child(터널) 단위**로 설정해야 합니다.

---

**즉, 별도의 데몬 옵션이 아니라, child 설정에서 `install_policy = no`만 추가하면 됩니다!**  


