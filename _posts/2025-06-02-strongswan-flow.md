정확한 strongSwan의 **remote_addrs(복수 원격 주소) 페일오버 동작**과 관련된 코드를 분석한 결과,  
실제로 "첫 번째 주소가 실패(IKE_INIT 응답 없음 등)하면 다음 주소로 넘어가는" 로직은 다음과 같이 동작합니다.

---

### 핵심 동작 구조

1. **remote_addrs**는 내부적으로 `ike_cfg_t`의 `other_hosts` 리스트에 여러 주소로 저장됩니다.
2. 연결 시도 시, `resolve_other()` → 내부적으로 `resolve()` 함수가 호출되어,  
   리스트의 첫 번째 주소부터 DNS/IP 변환을 시도합니다.
3. **host_create_from_dns()**가 실패(NULL 반환)하면, 다음 주소로 넘어갑니다.
4. **실패/재시도 로직**은 `ike_sa_t`의 `initiate()` 및 `retry_initiate()`에서 관리합니다.

---

### 실제 코드 흐름

#### 1. 주소 리스트 순회 (src/libcharon/config/ike_cfg.c)
```c
static host_t* resolve(linked_list_t *hosts, int family, uint16_t port)
{
    enumerator_t *enumerator;
    host_t *host = NULL;
    bool tried = FALSE;
    char *str;

    enumerator = hosts->create_enumerator(hosts);
    while (enumerator->enumerate(enumerator, &str))
    {
        host = host_create_from_dns(str, family, port);
        if (host)
        {
            break; // 성공하면 바로 반환
        }
        tried = TRUE;
    }
    enumerator->destroy(enumerator);

    if (!host && !tried)
    {
        // 아무 주소도 없으면 %any 반환
        host = host_create_any(family ?: AF_INET);
        host->set_port(host, port);
    }
    return host;
}
```
- 여러 remote_addrs 중 **DNS/IP 변환이 성공하는 첫 번째 주소**를 반환합니다.

#### 2. IKE_INIT 실패 시 재시도 (src/libcharon/sa/ike_sa.c)
```c
METHOD(ike_sa_t, initiate, status_t, private_ike_sa_t *this, ...)
{
    ...
    if (this->other_host->is_anyaddr(this->other_host))
    {
        if (!this->retry_initiate_interval)
        {
            // 모든 주소 실패 시, 재시도 없이 종료
            return DESTROY_ME;
        }
        // 재시도 타이머 설정
        defer_initiate = TRUE;
    }
    ...
    if (defer_initiate)
    {
        if (!this->retry_initiate_queued)
        {
            job_t *job = (job_t*)retry_initiate_job_create(this->ike_sa_id);
            lib->scheduler->schedule_job(lib->scheduler, (job_t*)job,
                                         this->retry_initiate_interval);
            this->retry_initiate_queued = TRUE;
        }
        return SUCCESS;
    }
    ...
}
```
- **IKE_INIT 실패(응답 없음 등)** 시, 다음 주소로 넘어가려면  
  `retry_initiate_interval`이 설정되어 있어야 하며,  
  재시도 시 다시 `resolve_other()`를 호출하여 **다음 주소**를 시도합니다.

---

### 결론 및 요약

- **실패 시 다음 주소로 넘어가는 코드**는 `resolve()`와 `initiate()`/`retry_initiate()`의 조합으로 동작합니다.
- **실패 조건**: IKE_INIT 응답 없음, DNS 실패 등으로 연결이 안 될 때.
- **다음 주소로 넘어가는 시점**:  
  - `host_create_from_dns()`가 NULL을 반환하거나  
  - IKE_INIT이 실패하여 재시도 타이머가 동작할 때.
- **모든 주소가 실패하면**: 더 이상 시도하지 않고 종료(`DESTROY_ME`).

---

#### 실제로 동작하는 코드 부분(핵심):

- `src/libcharon/config/ike_cfg.c` : `resolve()` 함수
- `src/libcharon/sa/ike_sa.c` : `initiate()`, `retry_initiate()`, `resolve_hosts()`

---

#### 참고
- strongSwan의 기본 동작은 **첫 번째 주소가 실패해야만** 다음 주소로 넘어갑니다.
- 여러 주소를 순차적으로 시도하려면, `retry_initiate_interval` 등 재시도 옵션이 활성화되어 있어야 합니다.

---
