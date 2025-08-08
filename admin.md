# Admin.md

## 개요

관리자(뷰어) 애플리케이션은 서버에 gRPC 스트림으로 접속해 특정 Agent의 화면 스트림과 이벤트를 수신하여 실시간으로 표시합니다. 데스크톱 앱(예: WPF, Electron, Qt) 또는 네이티브 앱으로 구현 가능합니다.

## 주요 책임

* 관리자 인증/권한 확보
* Agent 목록/Presence 보기
* 선택한 Agent의 실시간 스트림 구독
* 이벤트 로그 실시간 표시
* UI: 비디오 패널, 이벤트 패널, 제어(요청 스크린샷, 강제 캡처 등)

## 동작 플로우

1. 로그인(관리자 자격증명)
2. `StreamAdmin` gRPC 스트림 개설
3. `Subscribe` 메시지로 특정 `agent_id` 요청
4. 서버로부터 `FrameChunk` 및 `EventMessage` 수신하여 디코딩/렌더링
5. 사용자가 Disconnect하면 `Unsubscribe`

## UX/성능 고려사항

* 지터/패킷 손실이 있을 때 프레임 드랍 처리(프레임 스킵, 보간 X)
* 네트워크 상황에 따라 해상도/프레임레이트 동적 조정(Adaptive)
* 다수 Agent 동시 모니터링: 각 Agent의 프레임을 별도 스레드로 디코딩

## 권한/감사

* 관리자 행동 로깅(누구를 언제 봤는지)
* 민감정보 표시 경고(예: 비밀번호/주민등록번호 노출 시 알림)

---

# gRPC Proto 예시

```proto
syntax = "proto3";
package monitor;

message AuthRequest {
  string agent_id = 1;
  string token = 2; // JWT or client cert thumbprint
}

message AuthResponse {
  bool ok = 1;
  string message = 2;
}

message FrameMeta {
  string agent_id = 1;
  int64 timestamp = 2;
  int32 width = 3;
  int32 height = 4;
  int32 seq = 5;
  bool is_keyframe = 6;
}

message FrameChunk {
  FrameMeta meta = 1;
  bytes data = 2; // H.264 annex B or JPEG bytes
}

message EventMessage {
  string agent_id = 1;
  int64 timestamp = 2;
  string type = 3; // "print", "usb", "process"
  string payload_json = 4; // 세부내용
}

// Agent -> Server 스트림 메시지 (oneof)
message AgentMessage {
  oneof payload {
    AuthRequest auth = 1;
    FrameChunk frame = 2;
    EventMessage event = 3;
    // heartbeat, control responses 등
  }
}

// Server -> Admin/Agent 메시지
message ServerMessage {
  oneof payload {
    AuthResponse auth_resp = 1;
    FrameChunk frame = 2;
    EventMessage event = 3;
    // control commands
  }
}

service MonitorService {
  // Agent connects here
  rpc StreamAgent(stream AgentMessage) returns (stream ServerMessage);

  // Admin connects here
  rpc StreamAdmin(stream AgentMessage) returns (stream ServerMessage);

  // Optional control RPCs
  rpc Control(ControlRequest) returns (ControlResponse);
}
```

---

# 운영/배포 고려사항

* **TLS/mTLS 배포**: Istio/Envoy 또는 직접 nginx/Envoy로 TLS terminator 구성
* **스케일 아웃**: Redis/NATS로 presence sync, sticky-session LB 또는 Envoy 라우팅
* **리소스 제어**: Agent가 인코딩 시 CPU/GPU 사용량을 제한
* **모니터링**: Prometheus metrics (connections, frames/sec, avg\_latency)
* **로그/감사**: Elastic Stack 또는 Loki + Grafana

---

# 성능/네트워크 참고치 (예시)

* 1280x720 @ 15fps, H.264(2000 kbps) → 약 2 Mbps per client
* 서버는 중계만 한다면 네트워크는 `ingest_bandwidth ≈ sum(all agent bitrates)` 및 `egress_bandwidth ≈ sum(admin subscriptions * agent bitrate)`
* 멀티 Admin이 같은 Agent를 보는 경우, 서버가 단일 수신을 받고 멀티캐스트(메모리 복제)로 전송하므로 서버측 egress가 커짐

---

# 보안/법적 권고

* Agent는 배포 전 직원 동의를 반드시 받도록 UI/로그 및 동의서 기능 제공
* 감사 로그는 변경 불가능하게(append-only) 저장
* 권한 분리(누가 어떤 Agent를 볼 수 있는지 RBAC로 명확히)

---

# 추가 자료/다음 단계 제안

1. 이 설계를 바탕으로 **proto 파일**과 **간단한 reference 구현(Agent C#, Server Go, Admin Electron)** 스캐폴드 생성
2. POC: 한 Agent ↔ 한 Admin 연결로 end-to-end 레이턴시/대역폭 측정
3. 스케일 테스트: 100/500/1000 에이전트 시나리오 시뮬레이션

---

*끝*
