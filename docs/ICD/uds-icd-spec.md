# Interface Control Document (ICD): CCU ↔ ECU UDS Communication

## 1. 개요 (Introduction)
본 문서는 라즈베리파이(CCU)와 Aurix TC275(ECU) 간의 High-Speed CAN 기반 UDS(ISO 14229-1) 통신 메시지 규격을 상세히 정의합니다.

## 2. CAN 식별자 할당 (CAN Identifier Assignment)
모든 진단 메시지는 Standard ID (11-bit)를 사용합니다.

| Node | Direction | CAN ID (Hex) | Description |
| :--- | :--- | :--- | :--- |
| **Client (RPi)** | Tx (Request) | `0x7D3` | RPi가 TC275로 전송하는 진단 요청 |
| **Server (TC275)** | Tx (Response) | `0x7DB` | TC275가 RPi로 응답하는 진단 결과 |
| **Client (RPi)** | Tx (Func. Req) | `0x7DF` | (참고) 브로드캐스트 기능적 요청 주소 |

## 3. UDS Service Matrix (지원 서비스 목록)
TC275 제어기에서 필수적으로 지원해야 하는 주요 UDS 서비스(SID) 목록입니다.

| Service ID (Hex) | Service Name | 설명 |
| :--- | :--- | :--- |
| `0x10` | Diagnostic Session Control | 진단 세션(Default, Programming, Extended) 전환 |
| `0x11` | ECU Reset | 제어기 소프트웨어/하드웨어 리셋 |
| `0x14` | Clear Diagnostic Information | DTC(고장 코드) 삭제 |
| `0x19` | Read DTC Information | 현재 발생한 고장 코드 조회 |
| `0x22` | Read Data By Identifier | DID를 통한 특정 데이터(온도, 속도 등) 읽기 |
| `0x27` | Security Access | 보안 접근 (Seed & Key 인증 절차) |
| `0x2E` | Write Data By Identifier | 특정 DID 영역에 데이터 쓰기 |
| `0x31` | Routine Control | 특정 제어 루틴(팬 구동, 캘리브레이션 등) 실행 |

## 4. 상세 메시지 규격 (Message Specifications)

*참고: 아래 명시된 Data Payload는 ISO-TP 프레임의 PCI 바이트를 제외한 **순수 UDS 데이터**를 의미합니다.*

### 4.1. Diagnostic Session Control (SID: 0x10)
세션을 전환하여 특정 서비스(예: 보안 접근, 리프로그래밍)를 활성화합니다.

*   **Request Message (RPi → TC275, ID: 0x7D3)**
    | Byte 0 (SID) | Byte 1 (Sub-function) |
    | :--- | :--- |
    | `0x10` | `0x01` (Default) <br> `0x02` (Programming) <br> `0x03` (Extended) |

*   **Positive Response (TC275 → RPi, ID: 0x7DB)**
    | Byte 0 (Response SID) | Byte 1 (Sub-function Echo) | Byte 2~5 (Session Parameter Record) |
    | :--- | :--- | :--- |
    | `0x50` (0x10 + 0x40) | 요청한 세션 값 (예: `0x01`) | P2/P2* 타임아웃 정보 (4 Bytes) |

### 4.2. Read Data By Identifier (SID: 0x22)
특정 DID(Data Identifier, 2바이트)를 요청하여 센서 값이나 상태 정보를 읽어옵니다.

*   **Request Message (RPi → TC275, ID: 0x7D3)**
    | Byte 0 (SID) | Byte 1 (DID High) | Byte 2 (DID Low) |
    | :--- | :--- | :--- |
    | `0x22` | `0xF1` | `0x86` (예: Active Diagnostic Session) |
    | `0x22` | `0x01` | `0x0A` (예: Engine Coolant Temp) |

*   **Positive Response (TC275 → RPi, ID: 0x7DB)**
    | Byte 0 (Response SID) | Byte 1 (DID High) | Byte 2 (DID Low) | Byte 3...N (Data) |
    | :--- | :--- | :--- | :--- |
    | `0x62` (0x22 + 0x40) | 요청한 DID High | 요청한 DID Low | 실제 데이터 값 (Hex) |

### 4.3. Security Access (SID: 0x27)
데이터 쓰기나 펌웨어 업데이트 전 권한을 획득합니다. (Seed & Key 방식)

*   **Step 1: Request Seed (RPi → TC275)**
    | Byte 0 (SID) | Byte 1 (Sub-function) |
    | :--- | :--- |
    | `0x27` | `0x01` (Request Seed) |
*   **Step 1: Response (TC275 → RPi)**
    | Byte 0 (RSID) | Byte 1 | Byte 2~5 (Seed Data) |
    | :--- | :--- | :--- |
    | `0x67` | `0x01` | `0x3A 0x8F 0x11 0x2B` (난수) |

*   **Step 2: Send Key (RPi → TC275)**
    | Byte 0 (SID) | Byte 1 (Sub-function) | Byte 2~5 (Key Data) |
    | :--- | :--- | :--- |
    | `0x27` | `0x02` (Send Key) | PC에서 계산한 암호화 키 |
*   **Step 2: Response (TC275 → RPi)**
    | Byte 0 (RSID) | Byte 1 |
    | :--- | :--- |
    | `0x67` | `0x02` (인증 성공) |

## 5. Negative Response Code (NRC, 0x7F)
TC275가 요청을 처리할 수 없을 때 보내는 거절 응답 포맷입니다.

*   **Negative Response Format (ID: 0x7DB)**
    | Byte 0 (Negative Indicator) | Byte 1 (Rejected SID) | Byte 2 (NRC) |
    | :--- | :--- | :--- |
    | `0x7F` | 거절된 서비스 ID (예: `0x22`) | 에러 코드 (아래 참조) |

*   **주요 NRC 리스트**
    *   `0x11`: Service Not Supported (지원하지 않는 서비스)
    *   `0x12`: Sub-function Not Supported (지원하지 않는 서브 기능)
    *   `0x13`: Incorrect Message Length or Invalid Format (잘못된 길이/포맷)
    *   `0x22`: Conditions Not Correct (현재 조건에서 실행 불가)
    *   `0x31`: Request Out Of Range (DID가 없거나 범위 초과)
    *   `0x33`: Security Access Denied (보안 인증 필요)
    *   `0x78`: Request Correctly Received - Response Pending (처리 중이니 기다려라)
