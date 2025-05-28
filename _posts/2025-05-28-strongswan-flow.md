  

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
