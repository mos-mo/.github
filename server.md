# Server.md

## 개요

중앙 서버는 Agent로부터 들어오는 gRPC 스트림을 수신하고, 현재 그 Agent를 보고 있는 Admin(관리자)에게 해당 스트림을 **브로드캐스트(중계)** 합니다. 서버는 저장을 하지 않는 모드(실시간 전송만)와 저장 병행 모드(옵션)를 지원합니다. 서버는 여러 인스턴스로 수평 확장이 가능해야 합니다.

## 주요 책임

* 인증/인가 (mTLS / JWT 검증)
* 스트림 라우팅(Agent -> Admin)
* 세션/Presence 관리 (현재 누가 연결되어 있는지)
* 이벤트 처리(로그 저장 또는 실시간 전달)
* 로드 밸런싱, 스케일 아웃
* 모니터링/메트릭(프레임 레이트, 레이턴시, 연결 수)

## gRPC 서비스 설계(요약)

* `MonitorService`:

  * `rpc StreamAgent(stream AgentMessage) returns (stream ServerMessage)`
  * `rpc StreamAdmin(stream AdminMessage) returns (stream ServerMessage)`
  * `rpc Control(ControlRequest) returns (ControlResponse)`

## 메시지 라우팅 개념

* Agent가 `StreamAgent`로 접속하면 서버는 내부적으로 `agent_session`을 만들고, 해당 세션에 연결된 Admin 목록을 관리합니다.
* Admin이 `StreamAdmin`으로 접속하여 특정 `agent_id`를 구독하면 서버는 Agent 세션으로부터 받은 프레임/이벤트를 해당 Admin 스트림으로 전달합니다.
* 여러 Admin이 한 Agent를 구독할 수 있음(멀티캐스트)

## 상태 관리

* **In-memory**: 각 서버 인스턴스가 로컬 `agent_session` 테이블 유지 (빠름)
* **분산 Presence**: 다중 인스턴스 환경에서는 Redis Pub/Sub 또는 NATS로 세션 정보를 동기화하여 라우팅 보장

## 스케일링 아키텍처

* Fronting: TLS terminator + L4 load balancer → 여러 gRPC 서버로 라우팅
* Sticky-session 필요: gRPC 스트리밍은 상태 기반이므로 클라이언트가 같은 인스턴스로 연결되도록 LB 설정 또는 proxy에 session affinity 사용
* 혹은 Envoy + xDS로 라우팅하여 스트림 기반 라우팅 구현
* 확장성: Agent-Server 접속만 처리하는 `ingest` 레이어와 Admin 연결만 처리하는 `egress` 레이어로 분리 가능

## 저장/영구화(옵션)

* 실시간 전송만: 프레임은 메모리에서 즉시 전송, 디스크 기록 없음
* 기록 필요 시: MinIO/S3에 프레임/비디오 저장, 메타는 PostgreSQL

## 보안/권한

* Admin의 권한 체크: 어떤 Admin이 어떤 Agent를 볼 수 있는지 권한 테이블로 제어
* 감사로그: 누가 언제 어떤 화면을 조회했는지 기록

## 모니터링

* Prometheus + Grafana: 연결 수, 프레임 손실률, 평균 RTT
* 로그: structured logging (JSON)
