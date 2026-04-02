# Preliminary Design Specification: UDS Diagnostic Application

## 1. 개요 (Introduction)
본 문서는 CCU(Raspberry Pi) 게이트웨이를 경유하여 차량 내 ECU(Aurix TC275)와 통신하는 **멀티 플랫폼 UDS 진단 애플리케이션**의 예비 설계 사양을 정의한다. 본 시스템은 확장성, 유지보수성, 플랫폼 독립성을 최우선으로 고려하여 설계되었다.

## 2. 시스템 아키텍처 (System Architecture)

### 2.1 계층화 아키텍처 (Layered Architecture)
본 애플리케이션은 **Clean Architecture** 원칙을 준수하여 비즈니스 로직과 UI 프레임워크를 분리한다.

*   **Presentation Layer (MAUI)**: 사용자 인터페이스 및 플랫폼 특화 로직 처리.
*   **Application Layer (Core)**: 비즈니스 로직(UDS 서비스, 시나리오 엔진).
*   **Infrastructure Layer (Core/App)**: DoIP 프로토콜 스택 및 소켓 통신 구현.
*   **Domain Layer (Core)**: 프로토콜 데이터 모델(UDS/DoIP Packet) 정의.

### 2.2 모듈 구성도 (Component Diagram)
```text
[ UdsDiag.App (MAUI) ]
       |
       v (Dependency Injection)
[ UdsDiag.Core (Library) ]
       |-- [ UDS Service ] (Service ID Handling)
       |-- [ DoIP Service ] (Protocol Encapsulation)
       ┗-- [ TCP Interface ] (Abstract Socket)
```

## 3. 통신 프로토콜 설계 (Communication Protocol)

### 3.1 DoIP (Diagnostics over IP) Stack
ISO 13400 표준을 기반으로 하며, TCP/IP 환경에서 UDS 페이로드를 안정적으로 라우팅하기 위한 헤더 구조를 채택한다.

| Field | Size | Description |
| :--- | :--- | :--- |
| **Protocol Version** | 1 Byte | 0x02 (DoIP ISO 13400-2:2012) |
| **Inverse Version** | 1 Byte | 0xFD (Version XOR 0xFF) |
| **Payload Type** | 2 Bytes | 0x8001 (Diagnostic Message) |
| **Payload Length** | 4 Bytes | 뒷부분 데이터의 총 길이 (Big-Endian) |
| **UDS Payload** | N Bytes | 실제 UDS 서비스 데이터 |

### 3.2 통신 시퀀스 (Communication Sequence)
1.  **TCP Connection**: 클라이언트가 RPi 게이트웨이의 특정 포트(기본 13400)로 접속.
2.  **Routing Activation**: 클라이언트가 서버에 진단 경로 활성화 요청 전송.
3.  **UDS Request**: DoIP 헤더가 포함된 UDS 메시지 송신.
4.  **UDS Response**: 게이트웨이로부터 수신된 DoIP 응답 파싱 및 결과 처리.

## 4. 소프트웨어 상세 설계 (Software Detail Design)

### 4.1 의존성 주입 (Dependency Injection) 전략
플랫폼 독립적인 개발을 위해 모든 핵심 서비스는 인터페이스(Interface) 기반으로 주입된다.

```csharp
// 1. 통신 추상화 인터페이스
public interface ITcpClientProvider 
{
    Task ConnectAsync(string host, int port);
    Task SendAsync(byte[] data);
    IObservable<byte[]> OnDataReceived { get; }
}

// 2. DoIP 서비스 인터페이스
public interface IDoIpService 
{
    Task<byte[]> ExchangeDiagnosticMessage(byte[] udsData);
    ConnectionStatus Status { get; }
}
```

### 4.2 주요 서비스 클래스 설계
*   **`UdsDiagService`**: SID(Service ID)별 요청 조립 및 응답 분석(NRC 처리 포함).
*   **`DoIpHandler`**: DoIP 헤더 생성, 체크섬 검증, 페이로드 추출 담당.
*   **`ScenarioManager`**: 펌웨어 업데이트 등 복잡한 시나리오의 상태 머신(State Machine) 관리.

## 5. 멀티 플랫폼 및 확장성 고려사항

### 5.1 코드 재사용성 (Code Reusability)
*   **`.NET Standard 2.1` Library**: 핵심 프로토콜 스택(DoIP, UDS)을 별도 라이브러리로 빌드하여 향후 Blazor Web이나 데스크탑 전용 앱에서 100% 재사용 가능하게 한다.
*   **Platform-Specific Implementation**: 소켓의 타임아웃 처리나 네트워크 상태 감지 등 기기 종속적인 기능은 MAUI의 `Platforms` 폴더 내에서 구현하고 DI를 통해 전달한다.

### 5.2 UI/UX 디자인 원칙
*   **Responsive Layout**: `Grid` 및 `FlexLayout`을 활용하여 모바일(세로)과 태블릿/PC(가로) 화면비에 최적화된 레이아웃 제공.
*   **Asynchronous UX**: 모든 통신은 `async/await`를 사용하여 UI 프리징(Freezing) 방지 및 실시간 트레이스 로깅 보장.

## 6. 결론 및 향후 계획 (Conclusion & Roadmap)
본 설계를 기반으로 프로젝트 초기 스캐폴딩을 진행하며, 우선적으로 **DoIP 통신 모듈**의 단위 테스트를 통해 라즈베리파이와의 연결 안정성을 검증할 계획이다.
