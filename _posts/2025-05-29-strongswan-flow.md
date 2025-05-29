
---

## 1. charon에서 IKE 협상 및 SAD/SPD 설정 흐름

1. **tunnel 설정 전달 및 IKE 협상 시작**
   - charon은 swanctl, starter 등에서 전달받은 tunnel(연결) 설정을 바탕으로 IKE 협상을 시작합니다.
   - IKE_SA가 성공적으로 협상되면, CHILD_SA(실제 IPsec 터널)가 생성됩니다.

2. **CHILD_SA 생성 시 커널에 SAD/SPD 설정**
   - IKEv2: `src/libcharon/sa/ikev2/tasks/child_create.c`
   - IKEv1: `src/libcharon/sa/ikev1/tasks/quick_mode.c`
   - 위 파일에서 `child_sa->install()` 및 `child_sa->install_policies()`가 호출됩니다.
     - `install()` → 커널에 SA(SAD) 설치
     - `install_policies()` → 커널에 정책(SPD) 설치

3. **커널 인터페이스를 통한 실제 설치**
   - charon 내부적으로 커널 연동은 `kernel_interface_t`를 통해 추상화되어 있습니다.
   - 실제 커널 연동은 플러그인(`kernel-netlink`, `kernel-pfkey`, `kernel-libipsec` 등)이 `kernel_ipsec_t` 인터페이스를 구현하여 처리합니다.
   - 예시: `kernel_netlink_ipsec.c`, `kernel_pfkey_ipsec.c`, `kernel_libipsec_ipsec.c` 등

---

## 2. kernel-netlink 플러그인을 사용하지 않고 dummy 플러그인을 사용하는 경우

### 대표적인 dummy/fake 플러그인: `load_tester`의 `load_tester_ipsec`

- `src/libcharon/plugins/load_tester/load_tester_ipsec.c`를 보면,
  - `add_sa`, `add_policy` 등 모든 커널 연동 함수가 실제 커널에 아무 작업도 하지 않고 `SUCCESS`만 반환합니다.
  - 즉, 실제 커널 SAD/SPD에 아무런 변화가 없습니다.
  - 이 플러그인은 strongSwan의 부하 테스트, 개발, 시뮬레이션 용도로 사용됩니다.

#### 주요 구현 예시:
```c
METHOD(kernel_ipsec_t, add_sa, status_t, ...)
{
    return SUCCESS; // 실제 커널에 아무 작업도 하지 않음
}
METHOD(kernel_ipsec_t, add_policy, status_t, ...)
{
    return SUCCESS; // 실제 커널에 아무 작업도 하지 않음
}
```

### 결론

- **kernel-netlink**(리눅스), **kernel-pfkey**(BSD/일부 리눅스), **kernel-libipsec**(유저스페이스) 등 실제 커널과 연동되는 플러그인을 사용하면, IKE 협상 후 커널 SAD/SPD가 실제로 설정됩니다.
- **load_tester**와 같은 dummy/fake 플러그인을 사용하면, IKE 협상 및 내부 로직은 정상적으로 동작하지만 커널에는 아무런 SAD/SPD가 설치되지 않습니다. 즉, 실제 트래픽 보호는 일어나지 않습니다.

---

### 참고: 커널 플러그인 선택 구조

- strongSwan은 플러그인 구조로 되어 있어, `charon`이 시작될 때 설정에 따라 하나의 커널 플러그인만 활성화됩니다.
- 플러그인들은 `kernel_ipsec_t` 인터페이스를 구현하며, 실제 동작은 각 플러그인에 위임됩니다.

---


`charon->kernel`이 실제로 등록(초기화)되는 부분은 strongSwan의 데몬 초기화 과정에서 이루어집니다.

### 핵심 위치

- **파일:** `src/libcharon/daemon.c`
- **함수:** `private_daemon_t *daemon_create()`

```c
private_daemon_t *daemon_create()
{
    private_daemon_t *this;

    INIT(this,
        // ... 생략 ...
    );
    charon = &this->public;
    this->public.kernel = kernel_interface_create();
    // ... 생략 ...
    return this;
}
```

여기서 `this->public.kernel`이 바로 `charon->kernel`이 되며, `kernel_interface_create()`를 통해 커널 인터페이스가 생성됩니다.

---

### 플러그인에 의한 실제 커널 백엔드 등록

- `kernel_interface_create()`로 생성된 커널 인터페이스는 플러그인 로딩 시점에 각 커널 플러그인(`kernel-netlink`, `kernel-pfkey`, `kernel-libipsec`, `load_tester` 등)이
  - `kernel_ipsec_register()` 또는 `kernel_net_register()`를 통해
  - 자신이 구현한 커널 백엔드(예: `kernel_netlink_ipsec_create()`)를 등록합니다.
- 이 등록은 플러그인 내부의 `PLUGIN_CALLBACK(kernel_ipsec_register, ...)` 매크로로 자동 처리됩니다.

---

### 요약

- **charon->kernel**은 `daemon_create()`에서 `kernel_interface_create()`로 생성됨
- 실제 커널 백엔드(플러그인)는 strongSwan 플러그인 시스템을 통해 런타임에 등록됨

