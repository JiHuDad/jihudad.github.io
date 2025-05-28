  

`charon->controller->initiate()`의 call flow를 strongSwan 소스 구조에 맞춰 다시 설명드리겠습니다.

---

## 1. 호출 시작점

**extsock_plugin.c**  
```c
charon->controller->initiate(charon->controller, peer_cfg, child_cfg, ...);
```
- 이때 `charon->controller->initiate`는 **함수 포인터**이고,
- 실제로는 `src/libcharon/control/controller.c`의 **static status_t initiate(...)** 함수가 호출됩니다.

---

## 2. 실제 함수 연결 구조

### 2-1. 인터페이스
`controller_t` 구조체 (src/libcharon/control/controller.h)
```c
struct controller_t {
    ...
    status_t (*initiate)(controller_t *this,
                         peer_cfg_t *peer_cfg, child_cfg_t *child_cfg,
                         controller_cb_t callback, void *param,
                         level_t max_level, u_int timeout, bool limits);
    ...
};
```

### 2-2. 함수 포인터 할당
`controller_create()` (src/libcharon/control/controller.c)
```c
this->public.initiate = _initiate;
```
- 여기서 `_initiate`는 매크로로 생성된 **함수 포인터 변수**이고,
- 실제 함수 이름은 **initiate**입니다.

---

## 3. 실제 함수 정의 (METHOD 매크로 확장 결과)

```c
static status_t initiate(private_controller_t *this, peer_cfg_t *peer_cfg, child_cfg_t *child_cfg,
                         controller_cb_t callback, void *param, level_t max_level, u_int timeout, bool limits)
{
    // 1. interface_job_t 구조체 생성 및 초기화
    // 2. 콜백이 없으면 initiate_execute(job) 직접 실행
    // 3. 콜백이 있으면 wait_for_listener(job, timeout)로 대기
    // 4. status 반환
}
```

---

## 4. initiate 함수 내부 주요 흐름

1. **interface_job_t 구조체 생성 및 초기화**
   - 내부적으로 listener, logger, child_cfg, peer_cfg 등 세팅

2. **콜백이 없으면 initiate_execute(job) 직접 실행**
   - initiate_execute는 실제 IKE_SA 생성, task 큐잉, 상태머신 실행 등 핵심 로직 담당

3. **콜백이 있으면 wait_for_listener(job, timeout)로 대기**
   - 비동기 처리를 위한 대기

4. **status 반환**
   - 연결 성공/실패 등 결과 반환

---

## 5. initiate_execute의 역할

- **실제 IKE_SA 생성/검색**:  
  ike_sa_manager->checkout_by_config() 등 호출
- **task 추가**:  
  ike_sa->queue_task() 등으로 IKE_AUTH, CHILD_CREATE 등 task 추가
- **상태머신 실행**:  
  ike_sa->process()로 실제 패킷 송수신 및 협상 진행

---

## 6. 전체 call flow (함수명 기준)

```
extsock_plugin.c
  └─▶ charon->controller->initiate()
         │
         └─▶ initiate()   (src/libcharon/control/controller.c)
                │
                ├─▶ interface_job_t 생성/초기화
                ├─▶ initiate_execute(job) 호출
                │     │
                │     └─▶ IKE_SA 생성/검색 (ike_sa_manager->checkout_by_config)
                │     └─▶ IKE_SA에 task 추가 (ike_sa->queue_task)
                │     └─▶ IKE_SA 상태머신 실행 (ike_sa->process)
                │
                └─▶ status 반환
```

---

## 7. 연결 완료 후

- 연결이 성공하면, child_updown 이벤트가 발생
- extsock_plugin의 리스너(`extsock_child_updown`)가 호출되어 외부로 이벤트 전송

---

## 8. 요약

- **charon->controller->initiate()** → **initiate()** (METHOD 매크로로 생성)
- initiate() 내부에서 **initiate_execute()** 호출
- initiate_execute()에서 IKE_SA 생성/검색, task 추가, 상태머신 실행
- 연결 성공/실패 등 status 반환

---



아래는 **initiate_execute** 함수 내부의 실제 call flow를 함수 정의에 맞춰 상세하게 설명한 내용입니다.

---

## 1. initiate_execute 함수 정의

```c
METHOD(job_t, initiate_execute, job_requeue_t,
	interface_job_t *job)
{
	ike_sa_t *ike_sa;
	interface_listener_t *listener = &job->listener;
	peer_cfg_t *peer_cfg = listener->peer_cfg;

	// 1. IKE_SA 생성 또는 기존 것 checkout
	ike_sa = charon->ike_sa_manager->checkout_by_config(charon->ike_sa_manager, peer_cfg);
	peer_cfg->destroy(peer_cfg);

	if (!ike_sa)
	{
		DESTROY_IF(listener->child_cfg);
		listener->status = FAILED;
		listener_done(listener);
		return JOB_REQUEUE_NONE;
	}
	listener->lock->lock(listener->lock);
	listener->ike_sa = ike_sa;
	listener->lock->unlock(listener->lock);

	// 2. initiation limit 체크 (옵션)
	if (listener->options.limits && ike_sa->get_state(ike_sa) == IKE_CREATED)
	{
		// half-open IKE_SA, job load 등 시스템 제한 체크
		// 제한 초과 시 IKE_SA 파기 및 실패 처리
	}

	// 3. IKE_SA에 initiate 요청 (실제 협상 시작)
	if (ike_sa->initiate(ike_sa, listener->child_cfg, NULL) == SUCCESS)
	{
		// 4. 비동기/동기 처리에 따라 바로 성공 처리
		listener->status = SUCCESS;
		listener_done(listener);
		charon->ike_sa_manager->checkin(charon->ike_sa_manager, ike_sa);
	}
	else
	{
		listener->status = FAILED;
		charon->ike_sa_manager->checkin_and_destroy(charon->ike_sa_manager, ike_sa);
	}
	return JOB_REQUEUE_NONE;
}
```

---

## 2. 단계별 상세 call flow

### 1) IKE_SA 생성/검색
- `charon->ike_sa_manager->checkout_by_config(...)`
  - peer_cfg에 맞는 IKE_SA가 이미 있으면 반환, 없으면 새로 생성
  - 반환된 peer_cfg는 destroy (참조 해제)

### 2) initiation limit 체크 (옵션)
- `listener->options.limits`가 true이고, 새로 생성된 IKE_SA라면
  - half-open IKE_SA 개수, job load 등 시스템 제한을 체크
  - 초과 시 IKE_SA 파기, child_cfg 파기, 실패 처리

### 3) IKE_SA에 initiate 요청
- `ike_sa->initiate(ike_sa, listener->child_cfg, NULL)`
  - 실제 IKE_SA 협상(task 큐잉, 패킷 송수신, 상태머신 실행 등)
  - child_cfg가 있으면 CHILD_SA도 함께 생성

### 4) 결과 처리
- initiate가 성공하면:
  - listener->status = SUCCESS
  - listener_done(listener) 호출(동기/비동기 처리)
  - IKE_SA를 manager에 checkin
- initiate가 실패하면:
  - listener->status = FAILED
  - IKE_SA를 manager에 checkin_and_destroy

---

## 3. initiate_execute 이후의 흐름

- initiate_execute는 job 프레임워크에서 실행됨
- initiate()에서 동기/비동기 여부에 따라 바로 리턴하거나, 콜백/타임아웃 대기
- 연결 성공/실패 등은 status로 반환

---

## 4. 요약 다이어그램

```
initiate_execute(job)
   │
   ├─▶ ike_sa = charon->ike_sa_manager->checkout_by_config(...)
   │      └─ (peer_cfg destroy)
   │
   ├─▶ (옵션) initiation limit 체크
   │      └─ 초과 시 IKE_SA 파기, 실패 처리
   │
   ├─▶ ike_sa->initiate(ike_sa, child_cfg, NULL)
   │      └─ (실제 IKE_SA/CHILD_SA 협상, task 큐잉, 상태머신 실행)
   │
   └─▶ 성공/실패에 따라 status, IKE_SA checkin/파기, listener_done
```

---

## 5. initiate_execute에서 핵심적으로 호출하는 함수들

- **checkout_by_config**: IKE_SA 생성/검색
- **ike_sa->initiate**: 실제 IKE/CHILD 협상 시작 (상태머신 진입)
- **checkin/checkin_and_destroy**: IKE_SA 반환/파기
- **listener_done**: job 완료 처리

---


아래는 `ike_sa->initiate()`의 **정의 위치**와 **내부 동작**에 대한 상세 설명입니다.

---

## 1. 정의 위치

- **파일**: `src/libcharon/sa/ike_sa.c`
- **라인**: 1576
- **매크로**: `METHOD(ike_sa_t, initiate, status_t, private_ike_sa_t *this, child_cfg_t *child_cfg, child_init_args_t *args)`

---

## 2. 함수 내부 동작 (핵심 흐름)

### 1) IKE_SA 상태가 IKE_CREATED라면
- 로컬/원격 주소가 anyaddr(0.0.0.0 등)이면 `resolve_hosts(this)`로 주소를 해석
- 여전히 원격 주소가 anyaddr이면:
  - retry_initiate_interval이 0이면 실패 처리 및 child_cfg 파기
  - 아니면 재시도 예약(job 등록) 및 defer_initiate = TRUE

- initiator 플래그 세팅: `set_condition(this, COND_ORIGINAL_INITIATOR, TRUE)`
- IKE task 큐잉: `this->task_manager->queue_ike(this->task_manager)`

### 2) mediation(중재) 연결 처리 (특정 빌드 옵션)
- mediation 연결이면 이미 ESTABLISHED 상태면 상태만 갱신, child_cfg 파기

### 3) child_cfg가 있으면
- 일반적인 IKE_SA + CHILD_SA 생성
- `this->task_manager->queue_child(this->task_manager, child_cfg, args)`
- mediation 연결이면 mediation job 추가

### 4) 재시도 예약이 필요한 경우
- retry_initiate_job을 스케줄러에 등록

### 5) 마지막으로
- 재시도 예약이 아니면,  
  `this->task_manager->initiate(this->task_manager)` 호출  
  → 실제 IKE/CHILD 협상 task 실행

---

## 3. 내부적으로 호출되는 주요 함수

- **resolve_hosts(this)**: 주소 해석
- **set_condition(this, ...)**: initiator 플래그 세팅
- **this->task_manager->queue_ike(...)**: IKE_INIT/IKE_AUTH 등 IKE 협상 task 큐잉
- **this->task_manager->queue_child(...)**: CHILD_SA 생성 task 큐잉
- **this->task_manager->initiate(...)**: task 큐 실행(실제 패킷 송수신, 상태머신 진입)
- **retry_initiate_job_create(...)**: 재시도 job 생성 및 스케줄링

---

## 4. 요약 다이어그램

```
ike_sa->initiate(child_cfg, args)
   │
   ├─▶ (IKE_CREATED 상태면)
   │     ├─ resolve_hosts()
   │     ├─ (주소 미해결 시 재시도 예약/실패)
   │     ├─ set_condition(COND_ORIGINAL_INITIATOR)
   │     └─ task_manager->queue_ike()
   │
   ├─▶ (mediation 연결이면 mediation 처리)
   │
   ├─▶ (child_cfg 있으면)
   │     └─ task_manager->queue_child(child_cfg, args)
   │
   ├─▶ (재시도 예약 필요시 retry_initiate_job 등록)
   │
   └─▶ (정상 진행시) task_manager->initiate()
         └─ 실제 IKE/CHILD 협상 task 실행, 패킷 송수신, 상태머신 전이
```

---

## 5. 결론

- **정의 위치**: `src/libcharon/sa/ike_sa.c` 1576라인 부근
- **실제 동작**:  
  - 주소 해석 → initiator 세팅 → task 큐잉 → (필요시 재시도 예약) →  
    → task_manager->initiate()로 실제 IKE/CHILD 협상 task 실행

---

아래는 **IKEv2 기준**으로 `task_manager->initiate()`의 내부 동작과  
**task 실행 구조**를 상세하게 설명합니다.

---

## 1. 정의 위치

- **IKEv2**: `src/libcharon/sa/ikev2/task_manager_v2.c` 515라인  
  `METHOD(task_manager_t, initiate, status_t, private_task_manager_t *this)`

- **IKEv1**: `src/libcharon/sa/ikev1/task_manager_v1.c` 440라인  
  (구조는 유사, 아래 설명은 IKEv2 기준)

---

## 2. 내부 동작 (IKEv2 기준)

### 1) 이미 진행 중인 exchange가 있으면
- `this->initiating.type != EXCHANGE_TYPE_UNDEFINED`  
  → 이미 패킷 송수신 중이면 재전송/대기 처리

### 2) 활성화된 task가 없으면 (즉, 새로 시작)
- 현재 IKE_SA의 상태에 따라 필요한 task들을 **activate_task()**로 활성화
  - 예:  
    - IKE_CREATED: IKE_INIT, IKE_AUTH, CHILD_CREATE 등
    - IKE_ESTABLISHED: CHILD_CREATE, CHILD_DELETE, IKE_REKEY 등
- 각 상태별로 어떤 task를 활성화할지 결정

### 3) 이미 활성화된 task가 있으면
- derive_keys 등 필요한 준비를 하고,  
  활성화된 task의 종류에 따라 exchange type 결정

### 4) 실제 메시지 생성 및 송신
- exchange type이 결정되면:
  - message 객체 생성 (message_create)
  - source/destination, exchange type 등 세팅
  - initiating 구조체에 정보 기록

- **활성화된 task들을 순회하며 build() 호출**
  - 각 task의 build()가 message에 payload를 추가
  - build() 결과에 따라:
    - SUCCESS: task 완료, active_tasks에서 제거
    - NEED_MORE: 추가 exchange 필요, task 유지
    - FAILED: 실패 처리

- message가 완성되면 실제로 네트워크로 송신

---

## 3. task 실행 구조

### 1) task_manager의 역할
- 여러 개의 task(예: IKE_INIT, IKE_AUTH, CHILD_CREATE 등)를 관리
- 각 task는 IKE 교환의 한 단계를 담당 (payload 생성, 처리 등)

### 2) task의 생명주기
- **activate_task()**: 필요한 task를 활성화(active_tasks에 추가)
- **build()**: task가 message에 payload를 추가
- **process()**: 수신 메시지 처리 시 task가 payload를 해석/검증
- **destroy()**: 완료된 task는 제거

### 3) 주요 task 예시
- **TASK_IKE_INIT**: IKE_SA_INIT 교환
- **TASK_IKE_AUTH**: 인증 및 SA 설정
- **TASK_CHILD_CREATE**: CHILD_SA 생성
- **TASK_IKE_DELETE/CHILD_DELETE**: SA 삭제
- **TASK_IKE_REKEY/CHILD_REKEY**: SA 재키잉

---

## 4. 요약 다이어그램

```
task_manager->initiate()
   │
   ├─▶ (진행 중 exchange 있으면) → 재전송/대기
   │
   ├─▶ (활성화된 task 없으면) → 상태별로 activate_task()로 task 활성화
   │
   ├─▶ (활성화된 task 있으면) → derive_keys 등 준비
   │
   ├─▶ exchange type 결정 → message 생성
   │
   ├─▶ 활성화된 task들에 대해 build() 호출 → message에 payload 추가
   │
   ├─▶ message 완성 → 네트워크 송신
   │
   └─▶ task 완료 시 active_tasks에서 제거, NEED_MORE면 유지
```

---

## 5. 결론

- **task_manager->initiate()**는 IKE 상태에 따라 필요한 task를 활성화하고,
- 각 task의 build()를 통해 IKE 패킷을 구성, 송신합니다.
- task는 IKE 교환의 각 단계를 담당하며, 완료되면 제거되고, 추가 교환이 필요하면 유지됩니다.
- 이 구조는 IKEv2/IKEv1 모두 유사하게 동작합니다.

---
 
이제 **개별 task의 build/process 구조**와 **message 송수신 구조**를  
strongSwan의 실제 구조와 흐름에 맞춰 상세하게 설명드리겠습니다.

---

# 1. 개별 task의 구조와 동작

## 1.1 task란?
- IKE 교환의 한 단계를 담당하는 객체입니다.
- 예: IKE_INIT, IKE_AUTH, CHILD_CREATE, DELETE, REKEY 등
- 각 task는 payload 생성(build), 수신 메시지 처리(process) 등의 메서드를 가집니다.

## 1.2 task 인터페이스(핵심 메서드)
```c
struct task_t {
    task_type_t (*get_type)(task_t *this);
    status_t (*build)(task_t *this, message_t *message);
    status_t (*process)(task_t *this, message_t *message);
    void (*destroy)(task_t *this);
    // ... 기타
};
```

---

# 2. task의 생명주기

## 2.1 생성 및 활성화
- task_manager가 상태에 따라 필요한 task를 생성(activate_task)하여 active_tasks에 추가

## 2.2 build() 호출
- initiate() 시, 활성화된 task들에 대해 build()를 호출
- 각 task는 자신이 담당하는 payload를 message에 추가
- 예: IKE_INIT task는 SA, KE, Ni, N(payload) 등을 추가

## 2.3 process() 호출
- 메시지 수신 시, 해당 exchange에 맞는 task의 process()가 호출됨
- process()는 수신 payload를 해석, 검증, 상태 갱신 등을 수행

## 2.4 완료/제거
- build/process 결과가 SUCCESS면 task는 active_tasks에서 제거(destroy)
- NEED_MORE면 추가 교환 필요, task 유지
- FAILED면 실패 처리

---

# 3. message 송수신 구조

## 3.1 message 생성 및 송신
- task_manager->initiate()에서 message 객체를 생성
- 활성화된 task들의 build()를 호출해 payload를 추가
- message가 완성되면, 네트워크로 송신

## 3.2 message 수신 및 처리
- 패킷 수신 시, message 객체로 파싱
- task_manager->process_message()가 호출됨
- 해당 exchange에 맞는 task의 process()가 순차적으로 호출되어 payload 해석

---

# 4. 예시: IKE_AUTH 교환

1. **task_manager->initiate()**
   - TASK_IKE_AUTH가 활성화됨
   - build()에서 IDi, AUTH, CERT, SA, TSi, TSr 등 payload를 message에 추가
   - message 송신

2. **상대방이 응답을 보내면**
   - message 수신 → process_message() 호출
   - TASK_IKE_AUTH의 process()가 호출되어 IDr, AUTH, CERT, SA, TSi, TSr 등 해석
   - 인증 성공 시 task 완료, IKE_SA 상태 전이

---

# 5. 요약 다이어그램

```
[task_manager]
   │
   ├─▶ activate_task(TASK_IKE_AUTH)
   │
   ├─▶ task->build(message)  // payload 생성
   │
   ├─▶ message 송신
   │
   └─▶ (수신 시) task->process(message)  // payload 해석
           └─▶ SUCCESS → task 제거
           └─▶ NEED_MORE → task 유지
           └─▶ FAILED → 실패 처리
```

---

# 6. 실제 코드 예시

## build() 예시 (IKE_AUTH)
```c
status_t ike_auth_build(task_t *this, message_t *message)
{
    // IDi, AUTH, CERT, SA, TSi, TSr 등 payload 생성
    message->add_payload(...);
    // ...
    return SUCCESS;
}
```

## process() 예시 (IKE_AUTH)
```c
status_t ike_auth_process(task_t *this, message_t *message)
{
    // 수신 payload 해석, 인증 검증 등
    // ...
    return SUCCESS;
}
```

---

# 7. 결론

- **task**는 IKE 교환의 각 단계를 담당하는 객체
- **build()**로 payload 생성, **process()**로 수신 payload 해석
- **message**는 여러 task의 payload로 구성되어 송수신됨
- task는 완료되면 제거, 추가 교환 필요하면 유지

---





---
**IKE/CHILD task가 정상적으로 끝난 후, strongSwan이 extsock 플러그인으로 tunnel up(연결 성공)을 알리는 전체 call flow**를 상세하게 설명드리겠습니다.

---

# 1. 전체 흐름 요약

1. **IKE/CHILD task 완료 → CHILD_SA 생성**
2. **CHILD_SA 상태 변화 이벤트 발생**
3. **charon의 event bus에서 리스너 호출**
4. **extsock 플러그인의 child_updown 리스너 함수 호출**
5. **extsock이 외부로 tunnel up 이벤트(JSON) 전송**

---

# 2. 상세 call flow

## 1) IKE/CHILD task 정상 완료

- task_manager의 initiate/process에서 IKE_AUTH, CHILD_CREATE 등 task가 SUCCESS로 끝나면,
- 내부적으로 **CHILD_SA 객체가 생성**되고, strongSwan의 상태머신이 CHILD_SA를 "UP" 상태로 전이시킴

## 2) CHILD_SA 상태 변화 이벤트 발생

- strongSwan은 중요한 상태 변화(예: CHILD_SA UP/DOWN) 시 **event bus**를 통해 이벤트를 브로드캐스트합니다.
- 이때, `bus->child_updown()` 같은 콜백이 호출됨

## 3) event bus에서 리스너 호출

- charon->bus에는 여러 리스너(listener_t)가 등록되어 있음
- 각 리스너의 child_updown 함수가 호출됨

## 4) extsock 플러그인의 child_updown 리스너 호출

- extsock_plugin.c:
  ```c
  static bool extsock_child_updown(listener_t *this, ike_sa_t *ike_sa, child_sa_t *child_sa, bool up)
  ```
- 이 함수는 tunnel이 up(연결됨) 또는 down(끊김)일 때 호출됨

## 5) extsock이 외부로 tunnel up 이벤트 전송

- extsock_child_updown() 내부에서
  - 터널 정보(이름, SPI, 알고리즘, TS 등)를 JSON 문자열로 만듦
  - `send_event_to_external(json)` 함수로 외부(유닉스 도메인 소켓)로 전송

---

# 3. 실제 코드 흐름 예시

### (1) task_manager->initiate()/process()  
→ IKE/CHILD task 완료  
→ child_sa_create() 등으로 CHILD_SA 생성

### (2) child_sa 상태 변화  
→ charon->bus->child_updown(ike_sa, child_sa, up)

### (3) extsock 리스너 등록  
- 플러그인 로드 시:
  ```c
  charon->bus->add_listener(charon->bus, &extsock_listener);
  ```
- extsock_listener 구조체:
  ```c
  static listener_t extsock_listener = {
      .child_updown = extsock_child_updown,
  };
  ```

### (4) extsock_child_updown() 호출
- 터널 정보 JSON 생성
- send_event_to_external()로 외부 전송

---

# 4. 요약 다이어그램

```
[task_manager] (IKE/CHILD task 완료)
   │
   └─▶ [CHILD_SA 생성 및 상태 UP]
           │
           └─▶ [charon->bus->child_updown(ike_sa, child_sa, TRUE)]
                   │
                   └─▶ [extsock_child_updown() 호출]
                           │
                           └─▶ [send_event_to_external(JSON)]
                                   │
                                   └─▶ [외부 프로그램에서 tunnel up 이벤트 수신]
```

---

# 5. 실제 extsock_child_updown 코드(요약)

```c
static bool extsock_child_updown(listener_t *this, ike_sa_t *ike_sa, child_sa_t *child_sa, bool up)
{
    // 터널 정보 JSON 생성
    snprintf(buf, sizeof(buf),
        "{\"event\":\"tunnel_%s\", ... }", up ? "up" : "down", ...);
    send_event_to_external(buf);
    return TRUE;
}
```

---

# 6. 결론

- IKE/CHILD task가 정상적으로 끝나면 → CHILD_SA가 up됨
- charon의 event bus가 child_updown 이벤트를 extsock 등 리스너에 전달
- extsock_child_updown()이 외부로 tunnel up 이벤트(JSON) 전송

---

**이 흐름이 strongSwan에서 외부로 "터널이 올라갔다"(tunnel up)를 알리는 전체 구조입니다.**  

---


---


Ran tool

Read file: src/libcharon/bus/bus.c
좋은 질문입니다!  
아래는 **strongSwan event bus의 구조**와  
**tunnel up(즉, CHILD_SA가 up될 때) 리스너를 호출하는 실제 위치**에 대한 상세 설명입니다.

---

## 1. event bus 구조

- **bus_t**: strongSwan의 이벤트 브로드캐스트 시스템
- **listener_t**: 이벤트를 수신하는 리스너(플러그인, 내부 모듈 등)
- **bus_t**는 여러 listener_t를 등록/관리하며, 이벤트 발생 시 모든 리스너에 알림

### 주요 구조체
```c
struct private_bus_t {
    bus_t public;
    linked_list_t *listeners; // entry_t* 리스트
    ...
};

struct entry_t {
    listener_t *listener;
    int calling;
};
```

---

## 2. tunnel up(즉, child_updown) 이벤트 발생 시 리스너 호출

### 핵심 함수:  
**src/libcharon/bus/bus.c**
```c
METHOD(bus_t, child_updown, void,
    private_bus_t *this, child_sa_t *child_sa, bool up)
{
    ike_sa_t *ike_sa = this->thread_sa->get(this->thread_sa);

    this->mutex->lock(this->mutex);
    enumerator = this->listeners->create_enumerator(this->listeners);
    while (enumerator->enumerate(enumerator, &entry))
    {
        if (entry->calling || !entry->listener->child_updown)
        {
            continue;
        }
        entry->calling++;
        keep = entry->listener->child_updown(entry->listener,
                                             ike_sa, child_sa, up);
        entry->calling--;
        if (!keep)
        {
            unregister_listener(this, entry, enumerator);
        }
    }
    enumerator->destroy(enumerator);
    this->mutex->unlock(this->mutex);
}
```

### 동작 설명

1. **bus_t의 child_updown()이 호출**되면
2. 등록된 모든 리스너(entry->listener)의 **child_updown 함수 포인터**를 순회하며 호출
3. 각 리스너(예: extsock 플러그인)는 자신의 child_updown 구현체에서 이벤트를 처리

---

## 3. 실제 tunnel up 이벤트 발생 시점

- IKE/CHILD task가 성공적으로 끝나고, CHILD_SA가 up 상태가 되면
- 내부적으로 `charon->bus->child_updown(child_sa, TRUE)`가 호출됨
- 이 함수가 위의 bus_t의 child_updown()을 실행
- 등록된 모든 리스너(예: extsock_plugin의 extsock_child_updown)가 호출됨

---

## 4. 요약 다이어그램

```
[CHILD_SA up 발생]
   │
   └─▶ charon->bus->child_updown(child_sa, TRUE)
           │
           └─▶ (bus.c)
                   for (entry : listeners)
                       entry->listener->child_updown(entry->listener, ike_sa, child_sa, TRUE)
                           │
                           └─▶ [extsock_child_updown 등 각 리스너의 구현체 호출]
```

---

## 5. 결론

- **event bus의 child_updown()**이 tunnel up/down 이벤트를 모든 리스너에 브로드캐스트
- **실제 리스너 호출 위치**는 `src/libcharon/bus/bus.c`의 child_updown() 함수 내부
- extsock 플러그인 등은 이 리스너에 등록되어, tunnel up/down 이벤트를 받아 처리

---

