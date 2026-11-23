# portscope

Docker 컨테이너들이 어느 목적지로 몇 개의 연결과 포트를 쓰는지 집계하고 경보하는 Linux C 데몬이다. k8s의 DaemonSet처럼 에이전트 하나가 모든 컨테이너의 트래픽을 보고, 결과를 수집기로 보낸다. 통계 API는 mTLS로 보호한다.

## 상태

S1(저장소 골격) 진행 중이다. 아직 코드는 없다. 전체 일정은 [docs/plan.md](docs/plan.md)에 있다.

## 문서

| 문서 | 내용 |
| --- | --- |
| [docs/plan.md](docs/plan.md) | 6주 계획: 연결 감시 도구(1~4주), mTLS(5~6주) |
| [docs/process.md](docs/process.md) | 진행 방식: 단계마다의 순서, 질의답 기록, 단계 문서 구성 |
| [docs/decisions.md](docs/decisions.md) | 단계별 결정과 이유 |
| [docs/steps/](docs/steps/) | 단계 문서: 범위, 명세와 테스트 대응, 코드 → 개념 추적표 |
| [docs/qa/](docs/qa/) | 단계를 만들면서 오간 질의답 |
| [CLAUDE.md](CLAUDE.md) | Claude Code가 이 저장소에서 따르는 작업 규칙 |

## 빌드와 테스트

S1 구현 후 채운다.

## 라이선스

[MIT](LICENSE)
