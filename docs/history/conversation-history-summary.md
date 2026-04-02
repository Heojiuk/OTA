# Project Conversation History & Context Summary

> **집에서 이어서 작업할 때 지침**: 
> 새로운 Gemini CLI 세션을 시작한 후, **"이 폴더의 `conversation-history-summary.md` 파일을 읽고 지금까지의 설계 문맥을 파악해줘"**라고 요청하세요.

## 1. 프로젝트 개요 (Project Overview)
*   **목표**: CCU(라즈베리파이 4B)를 게이트웨이로 사용하여 ECU(Aurix TC275)를 진단/제어하는 멀티 플랫폼 UDS 앱 개발.
*   **통신 토폴로지**: 
    `[Client App (PC/Mobile)] <--- (DoIP/Wi-Fi) ---> [RPi 4B (Gateway)] <--- (CAN High-Speed) ---> [Aurix TC275 (ECU)]`

## 2. 주요 기술 스택 및 결정 사항 (Key Decisions)
*   **UI 프레임워크**: **.NET MAUI** (.NET 8 기반) - Windows, iOS, Android, macOS 통합 지원.
*   **아키텍처**: **MVVM Pattern** (CommunityToolkit.Mvvm 사용) + **Dependency Injection (DI)** 필수 적용.
*   **통신 프로토콜**:
    *   **PC ↔ RPi**: **DoIP (ISO 13400)** 표준 기반 TCP 소켓 통신.
    *   **RPi ↔ TC275**: **High-Speed CAN (1 Mbps)** 기반 UDS 통신.
*   **CAN Addressing**:
    *   Request ID: `0x7D3`
    *   Response ID: `0x7DB`
    *   Standard 11-bit ID 사용.

## 3. 진행된 작업 및 생성된 문서 목록
현재 워크스페이스(`D:\Source\Gemini_Test`)에 다음 문서들이 생성되어 있습니다:

1.  **`uds-app-concept.md`**: 서비스 컨셉 및 3가지 UI View(Dashboard, Engineering, Scenario) 정의.
2.  **`uds-preliminary-design-spec.md`**: 클린 아키텍처 기반의 예비 설계 사양서 (DI 인터페이스, 모듈화 전략).
3.  **`uds-can-comm-design.md`**: CCU와 ECU 간의 CAN 통신 프로토콜 변환 및 ISO-TP 설계.
4.  **`uds-icd-spec.md`**: (ICD) 바이트 단위의 UDS 메시지 포맷, SID별 페이로드 구조, NRC 정의.

## 4. 대화 히스토리 타임라인
*   **Step 1 (Concept)**: 윈도우 앱으로 시작했으나, 향후 모바일 및 클러스터 확장을 고려하여 멀티 플랫폼(MAUI)으로 방향 전환.
*   **Step 2 (Architecture)**: 100% DI 기반 모듈화 설계 확정. 비즈니스 로직(Core)과 UI(App)를 분리하여 웹 확장성 확보.
*   **Step 3 (Protocol)**: PC-RPi 구간은 DoIP로, RPi-ECU 구간은 High-speed CAN(1Mbps)으로 표준화.
*   **Step 4 (Detail Design)**: CAN ID 할당 및 주요 UDS 서비스(0x10, 0x22, 0x27)의 상세 바이트 명세(ICD) 작성 완료.

## 5. 다음 단계 제안 (Next Steps)
1.  **Project Scaffolding**: `.NET MAUI` 솔루션 생성 및 `UdsDiag.Core`, `UdsDiag.App` 프로젝트 구조화.
2.  **Base Library Implementation**: `IDoIpService` 인터페이스 및 DoIP 패킷 조립 로직 구현.
3.  **Mock Service**: 실제 하드웨어 연결 전, RPi 대용으로 동작할 Mock Server(Python 등) 구성하여 통신 테스트.
