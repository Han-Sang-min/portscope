# portscope 계획

Mac mini에서 돌아가는 Docker 컨테이너들이 어느 목적지로 몇 개의 연결과 포트를 쓰는지 집계하고 경보하는 Linux C 데몬(portscope)을 만들고, 그 통계 API를 mTLS로 보호한다. 두 프로젝트를 6주 동안 차례로 진행한다. 결과는 누구나 재현할 수 있어야 하고, Mac mini 서버에서 실제로 쓴다.

- 결정 기록: [decisions.md](decisions.md)
- 진행 방식: [process.md](process.md)
- 단계 문서: [steps/](steps/)
- 질의답 기록: [qa/](qa/)

## 실행 구조

k8s의 DaemonSet 형식을 Docker compose로 따른다. 에이전트 하나가 Docker 안의 모든 컨테이너 트래픽을 보고, 결과를 수집기로 보낸다.

```
 [컨테이너 A] [컨테이너 B] [컨테이너 C] ...      Docker Desktop의 Linux
        └──────────┴──────────┴── Docker 브리지
                         │ 캡처
              [portscope 에이전트, 1개]
                         │ 집계 결과 (mTLS)
                   [portscope 수집기]
                         │ 통계 API (mTLS)
                    폰, 노트북
```

- **에이전트**: 컨테이너가 늘어도 하나로 유지한다. 컨테이너는 IP로 구분한다.
- **수집기**: 에이전트의 집계를 모아 통계 API로 낸다. k8s의 Hubble Relay나 Prometheus에 해당한다.
- **범위**: Mac 호스트 프로세스는 감시하지 않는다.
- **개발**: 개발과 빌드는 Linux VM에서, CI는 Linux에서 한다.

## 프로젝트 1: 연결 감시 데몬 (1~4주)

다루는 영역: Linux 시스템 프로그래밍, TCP/IP, 패킷·프로토콜 분석, 동시성, 컨테이너 네트워킹.

### 구성

- **캡처**: libpcap(Linux에서는 `AF_PACKET`)으로 Docker 브리지 인터페이스에서 캡처한다. `pcap_create` → `pcap_activate` → `pcap_compile`/`pcap_setfilter` → `pcap_dispatch` 순서다.
- **링크 계층**: 브리지 인터페이스는 `DLT_EN10MB`, `any` 인터페이스는 `DLT_LINUX_SLL2`라 파싱 시작점이 다르다. 파일 입력 모드에서는 다른 OS에서 캡처한 `DLT_NULL` 파일도 읽는다.
- **헤더 파싱**: Ethernet 14바이트(VLAN이면 +4), IPv4 `IHL×4`, IPv6 40바이트 + 확장 헤더, TCP `data offset×4`. `caplen`과 `len`이 다를 수 있어 모든 접근 전에 경계를 검사하고, 비정렬 접근은 `memcpy`로 피한다.
- **연결 추적**: 5-tuple 해시 테이블, SYN/FIN/RST로 상태 관리, 타임아웃으로 항목 제거.
- **컨테이너·프로세스 매핑**: 패킷에는 컨테이너도 PID도 없다. 컨테이너는 Docker API로 IP→컨테이너 이름을 얻는다. 프로세스는 소켓 inode를 `/proc/<pid>/fd`와 맞추거나 sock_diag netlink로 찾는다. 짧게 끝나는 연결은 조회 전에 사라질 수 있다.
- **목적지 이름**: HTTPS 본문은 암호화돼 있지만 ClientHello의 SNI는 평문이다. DNS 응답을 파싱해 IP와 도메인을 짝짓는다.
- **이벤트 루프**: epoll로 캡처, 수집기 연결, 통계 API를 한 루프에서 처리한다.

### 주차별 단계

- [ ] 1주: 캡처, L2~L4 파싱, `pcap_open_offline` 파일 입력 모드
- [ ] 2주: 연결 추적, SNI·DNS 파싱
- [ ] 3주: 컨테이너·프로세스 매핑, 경보(처음 보는 외부 목적지, 임시 포트 사용량 임계치)
- [ ] 4주: epoll 이벤트 루프, 수집기와 통계 서버, Docker compose 배포

### 1주차 세부 단계

1주차를 프로토콜 계층 순서로 6단계로 나눈다. S는 Step이다.

- [ ] S1 저장소 골격: 빌드, 테스트, CI 틀. 패킷 코드는 없다
- [ ] S2 pcap 파일 입력, 링크 계층(DLT) 분기
- [ ] S3 IPv4/IPv6 파싱, 확장 헤더, 단편
- [ ] S4 TCP/UDP 파싱, JSON Lines 출력
- [ ] S5 합성 pcap 테스트 파일, 기대 출력 비교 테스트
- [ ] S6 파서 퍼징

### 증명

- 고정된 pcap 파일을 넣고 기대 출력과 비교하는 테스트. root 없이 누구나 돌릴 수 있다.
- libFuzzer로 파서 퍼징, ASan+UBSan과 MSan을 켠 CI. 비정상 패킷에 안전하다는 근거가 된다.
- 큰 pcap 파일로 처리량(pps) 측정, CI에서 epoll 테스트와 부하 수치.

### 실생활 사용

- **포트 고갈 진단**: 같은 목적지로 짧은 연결을 반복하면 TIME_WAIT가 쌓여 임시 포트가 바닥난다. 그 과정을 컨테이너별 숫자로 본다. 포트 범위는 `sysctl net.ipv4.ip_local_port_range`로 확인한다.
- **외부 연결 감시**: 금융 API를 쓰는 서비스처럼 민감한 컨테이너가 예상한 API 외의 곳으로 연결하는지 본다.

### 주의

- **개발 커널과 실행 커널이 다르다**: 개발은 Ubuntu VM에서, 실행은 Docker Desktop의 Linux에서 한다. 커널 기능에 기대는 부분(sock_diag, 캡처 옵션)은 실행 환경에서 다시 확인한다.
- **Docker Desktop 확인**: 에이전트가 Docker Desktop 안에서 브리지 캡처와 다른 컨테이너 프로세스 조회를 할 수 있는지 3주차 전에 확인한다.
- **kind 클러스터**: kind 노드도 컨테이너라 노드 단위로만 보이고, 그 안의 파드는 구분되지 않는다.
- **최소 권한**: `privileged` 대신 필요한 capability만 준다(`NET_RAW` 등). `snaplen`은 헤더만 담는 크기(예: 128바이트)로 둬서 페이로드를 저장하지 않는다.

### 검토할 사항

계획 안에서 서로 맞지 않거나 빠진 부분이다. 해당 주차의 결정 단계에서 정한다.

- **snaplen과 SNI·DNS (2주차)**: 128바이트로는 ClientHello의 SNI와 DNS 응답을 읽을 수 없다. X25519MLKEM768 key share(1216바이트) 때문에 ClientHello가 두 세그먼트로 나뉘는 경우도 흔하다. 핸들 두 개(전체는 128바이트, DNS와 443 앞쪽 세그먼트는 크게)와 앞쪽 세그먼트 재조립을 검토한다. 원칙을 "페이로드를 저장하거나 로그로 남기지 않는다"로 바꿀지도 함께 정한다.
- **프로세스 매핑 권한 (3주차)**: 다른 컨테이너 프로세스의 `/proc/<pid>/fd`를 읽으려면 호스트 PID 네임스페이스와 `CAP_SYS_PTRACE`가 필요하다. 최소 권한 원칙과의 균형을 정한다.
- **`/proc`와 sock_diag (3주차)**: `/proc` 순회는 단순하지만 느리고, sock_diag는 빠르지만 netlink를 다뤄야 한다.
- **MSan과 libpcap (S2)**: MSan 빌드에서는 링크하는 라이브러리도 MSan으로 빌드해야 오탐이 없다. libpcap을 소스에서 빌드한다.
- **범위**: QUIC(UDP 443)과 암호화 DNS(DoH) 트래픽은 SNI·DNS 매핑이 되지 않는다. 대상을 TCP로 명시한다.

## 프로젝트 2: mTLS 적용 (5~6주)

에이전트→수집기 구간과 수집기의 통계 API를 TLS로 감싸서, 인증서가 있는 쪽만 통신하게 한다. 다루는 영역: SSL/TLS, PKI, 보안 프로토콜 연동.

### 구성

- **개발 환경**: Linux의 OpenSSL 3(`libssl-dev`)에 링크한다.
- **서버 설정**: `SSL_CTX_new(TLS_server_method())`, `SSL_CTX_set_min_proto_version`으로 TLS 1.2 미만 차단, `SSL_CTX_use_certificate_chain_file`과 `SSL_CTX_use_PrivateKey_file`로 인증서와 키를 읽는다.
- **mTLS**: `SSL_CTX_load_verify_locations`(사설 CA)와 `SSL_CTX_set_verify(SSL_VERIFY_PEER | SSL_VERIFY_FAIL_IF_NO_PEER_CERT)`.
- **클라이언트**: 호스트명 검증(`SSL_set1_host`)과 SNI(`SSL_set_tlsext_host_name`). 빠뜨리면 MITM에 뚫린다.
- **사설 CA**: `openssl`로 루트 CA를 만들고 수집기·에이전트·클라이언트 인증서를 발급한다. 루트 키는 서버에 두지 않는다.
- **non-blocking 통합**: 가장 어려운 부분이다. `SSL_read`/`SSL_write`/`SSL_do_handshake`가 `SSL_ERROR_WANT_READ`나 `WANT_WRITE`를 돌려주면 epoll 관심 이벤트를 바꾼다. 읽기 중에 쓰기를 기다려야 하는 경우도 있어서 연결마다 핸드셰이크 상태 머신을 둔다.

### 주차별 단계

- [ ] 5주: blocking 에코 서버/클라이언트, 사설 CA, mTLS, 실패 케이스 테스트
- [ ] 6주: 이벤트 루프에 non-blocking 통합, testssl.sh 점검, 인증서 만료 감시

### 증명

- `openssl s_client` 스크립트로 아래 경우가 전부 거부되는지 CI에서 확인한다: 만료된 인증서, 다른 CA가 발급한 인증서, 호스트명 불일치, 클라이언트 인증서 없음, TLS 1.0/1.1.
- testssl.sh로 프로토콜, 암호군, 알려진 취약점을 점검한 결과.

### 실생활 사용

- 밖에서 폰으로 컨테이너 연결 현황과 경보를 확인한다.
- 인증서 발급, 만료, 교체를 직접 운영한다. 실제 서비스에서 장애가 자주 나는 지점이다.

### 주의

- **외부 노출**: 공유기 포트포워딩으로 인터넷에 직접 열면 공격 면적이 늘어난다. 기존 원격 접근 수단(VPN) 안쪽에 두고, mTLS는 두 번째 방어층으로 쓴다.
- **적용 범위**: 직접 짠 TLS 코드는 이 도구에만 쓴다. 다른 서비스 보호는 검증된 도구에 맡긴다.
