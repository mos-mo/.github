# Agent.md

## 개요

Agent는 피감시자(클라이언트) 측 응용프로그램으로서 화면 캡처(또는 인코딩된 비디오 프레임)와 각종 활동 이벤트(프린트, USB, 프로세스 실행 등)를 실시간으로 중앙 서버에 **gRPC 양방향 스트리밍**으로 전송합니다. Agent는 데스크톱 애플리케이션(C++/C#/Go/Rust 등)으로 구현됩니다.

## 주요 책임

* 화면 캡처(예: DXGI on Windows)
* 프레임 인코딩(H.264 또는 JPEG 시퀀스)
* 이벤트 수집(프린터, USB, 프로세스, 네트워크 등)
* gRPC 양방향 스트리밍 연결 수립 및 재연결 로직
* 인증(클라이언트 인증서 또는 JWT)
* 로컬 설정(프레임레이트, 해상도, 캡처 영역)
* 리소스 관리(CPU, GPU, 네트워크) 및 QoS

## 동작 플로우(요약)

1. 시작 시 설정 불러오기(서버 주소, 인증 토큰/클라이언트 cert)
2. gRPC 연결 수립: `MonitorService.StreamAgent()` 양방향 스트림 생성
3. 인증 메시지(`AuthRequest`) 전송, 서버 `AuthResponse` 수신
4. 캡처 루프: 프레임 생성 → 인코딩 → `FrameChunk` 메시지로 스트리밍
5. 이벤트 발생 시 `EventMessage`로 전송(동시 스트림 사용)
6. Heartbeat/Keepalive 처리
7. 네트워크 오류 발생 시 backoff 재시도, 로컬 큐에 최근 N초(선택적) 보관 후 폐기

## 구성/설정 항목

* `server_address` (예: monitor.example.com:50051)
* `agent_id`, `device_id`
* 인증: `use_mtls` 또는 `jwt_token`
* `capture_fps` (권장 디폴트: 15)
* `capture_resolution` (예: 1280x720)
* `bitrate_target` (H.264, kbit/s)
* `max_memory_frames` (메모리에 보관할 최신 프레임 수)

## 실패 모드/복구

* 네트워크 단절: 지수 백오프 재시도(최대 대기시간 제한)
* 인증 실패: 관리자 알림, 로컬 감사 로그, 주기적 토큰 재요청 시도
* 인코딩 실패: 저품질 JPEG로 폴백

## 보안

* **mTLS 권장**: 서버와 Agent간 상호 인증
* 메시지 레벨 암호화: gRPC/TLS
* 최소 권한의 로컬 계정으로 실행

