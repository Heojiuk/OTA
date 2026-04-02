# CAN UDS Communication Design (CCU ↔ ECU)

## 1. 개요 (Overview)
본 문서는 Gateway 역할을 수행하는 라즈베리파이(CCU)와 최종 제어기인 Aurix TC275(ECU) 간의 **High-Speed CAN 기반 UDS 통신 설계**를 정의합니다.

*   **물리적 연결**: CCU (라즈베리파이 CAN Shield) ↔ CAN Bus ↔ ECU (Aurix TC275)
*   **통신 표준**:
    *   Application Layer: ISO 14229 (UDS)
    *   Network Layer: ISO 15765-2 (ISO-TP) - 8바이트 초과 데이터 분할 전송
    *   Data Link Layer: ISO 11898-1 (High-Speed CAN, 1Mbps)

## 2. 통신 아키텍처 및 프로토콜 변환 흐름
라즈베리파이(CCU)는 PC로부터 DoIP 패킷을 수신하면, 헤더를 제거하고 순수 UDS 페이로드를 ISO-TP 기반의 CAN 프레임으로 변환하여 송출합니다.

### 2.1 패킷 변환 흐름 (Protocol Translation Flow)
1.  **[PC → RPi]**: DoIP 패킷 수신 (TCP/IP)
    *   `[DoIP Header (8B)]` + `[UDS Payload (e.g., 0x22 0xF1 0x86)]`
2.  **[RPi 내부 처리]**: DoIP 헤더 제거 후 목적지 주소 확인. UDS Payload를 ISO-TP 프레임(SF, FF 등)으로 포장.
3.  **[RPi → TC275]**: CAN 프레임 송출
    *   `[CAN ID: 0x7D3] [DLC: 8] [Data: 03 22 F1 86 00 00 00 00]`
4.  **[TC275 → RPi]**: UDS 응답 CAN 프레임 수신
    *   `[CAN ID: 0x7DB] [DLC: 8] [Data: 05 62 F1 86 01 02 00 00]`
5.  **[RPi → PC]**: 수신된 CAN 페이로드를 조립하여 DoIP 헤더를 붙여 PC로 TCP 송신.

## 3. 네트워크 계층 (ISO-TP, ISO 15765-2) 설계
UDS 메시지는 길이가 다양하므로 반드시 ISO-TP 규격을 통해 프레임화(Segmentation)를 수행해야 합니다.

### 3.1 ISO-TP 프레임 타입 (Frame Types)
| PCI (Protocol Control Info) | Type | 설명 |
| :--- | :--- | :--- |
| `0x00` ~ `0x0F` | **Single Frame (SF)** | 7바이트 이하의 짧은 UDS 메시지 전송. (Byte 0의 하위 4비트가 데이터 길이) |
| `0x10` ~ `0x1F` | **First Frame (FF)** | 8바이트 이상의 긴 UDS 메시지 전송 시작. (전체 길이를 포함) |
| `0x20` ~ `0x2F` | **Consecutive Frame (CF)** | FF 이후에 이어지는 데이터 프레임 블록. |
| `0x30` ~ `0x3F` | **Flow Control (FC)** | 수신측(TC275)이 RPi에게 전송 속도 및 버퍼 상태를 알려주는 제어 프레임. |

## 4. CAN 네트워크 파라미터
*   **Baud Rate**: 1 Mbps (High-Speed CAN)
*   **Frame Format**: Standard Frame (11-bit Identifier)
*   **Padding**: `0x00` (8바이트를 채우기 위한 기본 더미 값)
*   **Timeout (N_As, N_Cr 등)**: ISO 15765-2 권장 타이머 규격 준수 (기본 150ms 내 응답 대기).