
`src/libcharon/bus/bus.c`의 **bus_t 이벤트 메서드들**이 어떤 상황에서 발생하는지,  
실제 strongSwan의 동작 맥락에 맞춰 상세하게 설명드리겠습니다.

---

## 1. 주요 이벤트 메서드와 발생 상황

### 1.1. `ike_state_change(ike_sa, state)`
- **상황**: IKE_SA의 상태가 변할 때 (예: 생성, 인증 완료, 삭제 등)
- **예시**: IKE_AUTH 성공 → IKE_ESTABLISHED, 삭제 → IKE_DELETING 등

---

### 1.2. `child_state_change(child_sa, state)`
- **상황**: CHILD_SA(터널)의 상태가 변할 때 (예: 생성, 삭제, 재키잉 등)
- **예시**: CHILD_CREATE 성공 → CHILD_INSTALLED, 삭제 → CHILD_DELETED 등

---

### 1.3. `ike_updown(ike_sa, up)`
- **상황**: IKE_SA가 up(성공적으로 생성) 또는 down(삭제/종료)될 때
- **예시**: IKE_AUTH 성공 → up, delete 처리 → down

---

### 1.4. `child_updown(child_sa, up)`
- **상황**: CHILD_SA(터널)가 up(생성/설치됨) 또는 down(삭제/종료)될 때
- **예시**: CHILD_CREATE 성공 → up, delete 처리 → down  
- **extsock 플러그인에서 tunnel up/down 이벤트를 받는 핵심 메서드**

---

### 1.5. `child_rekey(old, new)`
- **상황**: CHILD_SA가 재키잉(rekey)되어 새 SA로 교체될 때
- **예시**: 터널의 키 교체가 완료되어 old/new CHILD_SA가 바뀔 때

---

### 1.6. `children_migrate(new_id, unique)`
- **상황**: CHILD_SA들이 다른 IKE_SA로 마이그레이션될 때 (MOBIKE 등)
- **예시**: 네트워크 경로 변경 등으로 CHILD_SA가 새로운 IKE_SA로 이동

---

### 1.7. `ike_rekey(old, new)`
- **상황**: IKE_SA가 재키잉(rekey)되어 새 SA로 교체될 때
- **예시**: IKE_REKEY 교환이 성공적으로 끝나서 old/new IKE_SA가 바뀔 때

---

### 1.8. `ike_update(ike_sa, local, remote)`
- **상황**: IKE_SA의 로컬/리모트 주소가 변경될 때 (MOBIKE 등)
- **예시**: IP 변경, NAT 트래버설 등

---

### 1.9. `alert(alert, ...)`
- **상황**: 인증서 만료, 인증 실패 등 중요한 이벤트 발생 시
- **예시**: 인증서 만료, 인증 실패, 경고 등

---

### 1.10. `message(message, incoming, plain)`
- **상황**: IKE 패킷이 송수신될 때
- **예시**: 모든 IKE 패킷 송수신 시

---

### 1.11. `ike_keys`, `ike_derived_keys`, `child_keys`, `child_derived_keys`
- **상황**: 키 교환/생성/파생이 완료될 때
- **예시**: IKE/CHILD 키 생성, 파생, 교체 등

---

### 1.12. `narrow(child_sa, type, local, remote)`
- **상황**: 트래픽 셀렉터 협상 결과가 좁혀질 때
- **예시**: TS narrowing

---

### 1.13. `assign_vips`, `handle_vips`
- **상황**: 가상 IP(VIP) 할당/처리 시
- **예시**: IKEv2 CFG 요청/응답 등

---

### 1.14. `authorize(final)`
- **상황**: 인증/인가 관련 이벤트 발생 시
- **예시**: 인증서 검증, 정책 위반 등

---

## 2. 이벤트 발생 예시(실제 코드 흐름)

- **IKE/CHILD task 성공 → child_updown(child_sa, TRUE)**
- **IKE_AUTH 성공 → ike_state_change(ike_sa, IKE_ESTABLISHED), ike_updown(ike_sa, TRUE)**
- **터널 삭제 → child_updown(child_sa, FALSE), child_state_change(child_sa, CHILD_DELETED)**
- **키 교체 → child_rekey(old, new), ike_rekey(old, new)**
- **MOBIKE → ike_update(ike_sa, local, remote), children_migrate(new_id, unique)**
- **IKE 패킷 송수신 → message(message, incoming, plain)**
- **인증서 만료 등 → alert(alert, ...)**

---

## 3. 요약

- **bus_t의 각 이벤트 메서드는 strongSwan 내부의 중요한 상태 변화, 패킷 송수신, 키 교환, 인증 등에서 호출됩니다.**
- **child_updown**은 터널이 up/down될 때, **ike_updown**은 IKE_SA가 up/down될 때,  
  **alert**는 인증서 등 보안 이벤트, **message**는 모든 IKE 패킷 송수신 등에서 발생합니다.

---

