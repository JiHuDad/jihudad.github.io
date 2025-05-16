
SNMP example
```c
#include <net-snmp/net-snmp-config.h>
#include <net-snmp/net-snmp-includes.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define BGP_OID_PREFIX "1.3.6.1.2.1.15"      // 예시: BGP4-MIB
#define BFD_OID_PREFIX "1.3.6.1.4.1.3317.1"  // 예시: BFD-MIB (실제 OID는 환경에 맞게 수정)

void process_trap(netsnmp_pdu *pdu) {
    netsnmp_variable_list *vars;
    printf("=== SNMP Trap received ===\n");
    for (vars = pdu->variables; vars; vars = vars->next_variable) {
        char buf[1024];
        snprint_variable(buf, sizeof(buf), vars->name, vars->name_length, vars);
        printf("%s\n", buf);

        // BGP/BFD OID 필터 예시
        char oidbuf[256];
        snprint_objid(oidbuf, sizeof(oidbuf), vars->name, vars->name_length);
        if (strncmp(oidbuf, BGP_OID_PREFIX, strlen(BGP_OID_PREFIX)) == 0 ||
            strncmp(oidbuf, BFD_OID_PREFIX, strlen(BFD_OID_PREFIX)) == 0) {
            // 여기에 외부 시스템 연동 코드 추가
            // 예: REST API 호출, 파일 기록 등
            printf(">>> [ALERT] BGP/BFD 관련 알람 감지: %s\n", oidbuf);
        }
    }
    printf("\n");
}

int main(int argc, char **argv) {
    struct snmp_session session, *ss;
    int fds, block;
    fd_set fdset;
    int numfds;
    struct timeval timeout, *tvp;

    // SNMP 라이브러리 초기화
    init_snmp("snmptrapd");
    snmp_sess_init(&session);
    session.version = SNMP_VERSION_2c;
    session.community = (u_char *)"public";
    session.community_len = strlen((const char *)session.community);
    session.peername = strdup("udp:0.0.0.0:162"); // 162 포트에서 수신

    SOCK_STARTUP;
    ss = snmp_open(&session);
    if (!ss) {
        snmp_perror("snmp_open");
        SOCK_CLEANUP;
        exit(1);
    }

    printf("SNMP Trap receiver started on UDP/162 ...\n");

    while (1) {
        FD_ZERO(&fdset);
        numfds = 0;
        block = 1;
        snmp_select_info(&numfds, &fdset, &timeout, &block);
        tvp = block ? NULL : &timeout;
        fds = select(numfds, &fdset, NULL, NULL, tvp);
        if (fds > 0) {
            snmp_read(&fdset);
        } else if (fds == 0) {
            snmp_timeout();
        } else {
            if (errno == EINTR)
                continue;
            perror("select failed");
            break;
        }

        // Trap 처리
        while (1) {
            netsnmp_pdu *pdu = NULL;
            int rc = snmp_sess_read(ss, &pdu);
            if (rc == 0 || !pdu)
                break;
            if (pdu->command == SNMP_MSG_TRAP2 || pdu->command == SNMP_MSG_INFORM) {
                process_trap(pdu);
            }
            snmp_free_pdu(pdu);
        }
    }

    snmp_close(ss);
    SOCK_CLEANUP;
    return 0;
}
```

build
```
sudo apt-get install libsnmp-dev
gcc -o snmptrap_listener snmptrap_listener.c -lnetsnmp
sudo ./snmptrap_listener
```
참고: 162 포트는 root 권한 필요. 개발/테스트용으로 10162 등 높은 포트로 변경 가능.

OID 확인
1) MIB 파일 확인
FRR의 경우, 설치 경로(예: /usr/share/snmp/mibs/ 또는 /usr/local/share/snmp/mibs/)에
BGP4-MIB.txt, BFD-MIB.txt, FRR-BGP-MIB.txt 등 MIB 파일이 있을 수 있음
MIB 파일에서 OBJECT IDENTIFIER 또는 TRAP-TYPE/NOTIFICATION-TYPE을 검색

3) snmptranslate 명령어 사용
OID와 이름을 상호 변환할 수 있음

예시:
```bash
  snmptranslate -On BGP4-MIB::bgpEstablished
  # 결과: .1.3.6.1.2.1.15.3.1.2
  snmptranslate -On BFD-MIB::bfdSessState
  # 결과: .1.3.6.1.4.1.3317.1.2.2.1.8 (예시)
```

3) Trap 수신 후 OID 직접 확인
SNMP Trap 리스너(예: 위 Python/C 코드)로 Trap을 받아
실제로 오는 OID를 콘솔에 출력해서 확인
예시 출력:
```bash
  1.3.6.1.2.1.15.3.1.2 = 6
  1.3.6.1.4.1.3317.1.2.2.1.8 = 3
```

4) FRR/장비의 SNMP 설정/문서 확인
FRR 공식 문서, 장비 벤더 문서에서
"BGP/BFD Trap OID" 또는 "MIB 지원 목록"을 확인
