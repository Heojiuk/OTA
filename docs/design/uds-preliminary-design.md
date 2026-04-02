# UDS 진단 애플리케이션 예비 시스템 설계 (Preliminary Design)

## 1. 아키텍처 개요 및 설계 철학
*   **핵심 원칙**: UI 레이어(MAUI)와 비즈니스 로직(UDS 통신, 파싱)을 완전히 분리하여 향후 순수 Web 등으로 전환하기 쉽도록 설계.
*   **기술 스택**:
    *   **UI/Platform**: .NET MAUI (.NET 8 기반)
    *   **Architecture**: MVVM 패턴 (CommunityToolkit.Mvvm)
    *   **DI**: Microsoft.Extensions.DependencyInjection (기본 내장)
    *   **Reusability**: 비즈니스 로직은 플랫폼 독립적인 **.NET Standard 2.0/2.1** 또는 **.NET 8 Class Library**로 분리.

## 2. 프로젝트 및 솔루션 구조 (모듈화)

```text
📁 UdsDiagSolution
 ┣ 📁 UdsDiag.Core (Class Library)
 ┃ ┣ 📁 Models
 ┃ ┃ ┣ 📄 DoIpPacket.cs      (DoIP 프로토콜 구조체)
 ┃ ┃ ┣ 📄 UdsMessage.cs      (UDS SID, SubFunction, Data Payload)
 ┃ ┃ ┗ 📄 ConnectionState.cs (연결 상태 Enum)
 ┃ ┣ 📁 Interfaces (DI 핵심)
 ┃ ┃ ┣ 📄 ITcpClientFactory.cs (소켓 생성 팩토리)
 ┃ ┃ ┣ 📄 IDoIpService.cs      (DoIP 패킷 조립/전송)
 ┃ ┃ ┣ 📄 IUdsDiagService.cs   (UDS 진단 로직)
 ┃ ┃ ┗ 📄 ILoggerService.cs    (로깅)
 ┃ ┗ 📁 Services (Implementation)
 ┃   ┣ 📄 DoIpServiceImpl.cs
 ┃   ┗ 📄 UdsDiagServiceImpl.cs
 ┃
 ┗ 📁 UdsDiag.App (MAUI App Project)
   ┣ 📁 Platforms (iOS, Android, Windows 특화 로직)
   ┣ 📁 ViewModels
   ┃ ┣ 📄 MainViewModel.cs
   ┃ ┣ 📄 DashboardViewModel.cs
   ┃ ┗ 📄 EngineeringViewModel.cs
   ┣ 📁 Views
   ┃ ┣ 📄 MainPage.xaml
   ┃ ┣ 📄 DashboardPage.xaml
   ┃ ┗ 📄 EngineeringPage.xaml
   ┣ 📁 PlatformServices
   ┃ ┗ 📄 MauiTcpClientFactory.cs (MAUI 기반 소켓 구현)
   ┗ 📄 MauiProgram.cs (DI 설정 진입점)
```

## 3. 주요 인터페이스 설계

### A. 통신 계층 (DoIP & TCP)
```csharp
public interface IDoIpService
{
    Task ConnectAsync(string ipAddress, int port);
    Task DisconnectAsync();
    
    // DoIP 헤더를 결합하여 Payload 전송 및 응답 대기
    Task<byte[]> SendDiagnosticMessageAsync(byte targetAddress, byte[] udsPayload);
    
    event EventHandler<ConnectionState> ConnectionStateChanged;
}
```

### B. 비즈니스 로직 계층 (UDS)
```csharp
public interface IUdsDiagService
{
    Task<UdsResponse> ReadDataByIdentifierAsync(ushort did);
    Task<UdsResponse> StartDiagnosticSessionAsync(byte sessionType);
    Task<UdsResponse> StartRoutineAsync(ushort routineId, byte[] routineControlOptionRecord);
}
```

## 4. DI (의존성 주입) 구성 예시

```csharp
public static class MauiProgram
{
    public static MauiApp CreateMauiApp()
    {
        var builder = MauiApp.CreateBuilder();
        builder.UseMauiApp<App>();

        // 1. Core Services (플랫폼 독립 로직)
        builder.Services.AddSingleton<IDoIpService, DoIpServiceImpl>();
        builder.Services.AddSingleton<IUdsDiagService, UdsDiagServiceImpl>();

        // 2. Platform Specific Services
        builder.Services.AddSingleton<ITcpClientFactory, MauiTcpClientFactory>();

        // 3. ViewModels & Views
        builder.Services.AddTransient<EngineeringViewModel>();
        builder.Services.AddTransient<EngineeringPage>();

        return builder.Build();
    }
}
```

## 5. 데이터 흐름
1. **View** → 버튼 클릭
2. **ViewModel** → `_udsDiagService.ReadDataByIdentifierAsync()` 호출
3. **UdsDiagService** → UDS 패킷 조립 후 `_doIpService.SendDiagnosticMessageAsync()` 전달
4. **DoIpService** → DoIP 헤더 결합 후 소켓 송신
5. **소켓** → RPi(Gateway)로 데이터 전송 및 응답 수신
6. **역순 파싱** → UI 데이터 바인딩 갱신
