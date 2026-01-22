# 실제 개념
임베디드 시스템에서 CAN, LIN, Ethernet은 단순한 Peripheral 통신(UART/SPI/I2C)을 넘어, 여러 노드가 하나의 네트워크를 구성하여 통신하는 분산 시스템 통신 프로토콜입니다: `Peripheral 인터페이스 → 분산 네트워크 시스템` 

이들은 다음과 같은 공통 특징을 가집니다.
- Multi-node Bus 또는 Network 구조
- 충돌 제어 및 중재 메커니즘 포함
- 프레임 신뢰성 구조(CRC, ACK, 재전송)
- 실시간 특성 또는 고대역폭 처리 목적
- PHY와 Controller로 역할 분리: `핀 → PHY → MAC/Controller → 프레임 → 필터링/중재 → 버퍼 → CPU`
    * PHY: 전기 신호 변환, 차등 구동, 물리 계층 담당
    * Controller: 프레임 처리, 필터링, 버퍼 관리

## 1. CAN이란

CAN(Controller Area Network)은 여러 노드가 하나의 차동 버스를 공유하며, 메시지 ID 기반 중재를 통해 충돌 없이 실시간 통신을 수행하는 네트워크 프로토콜입니다.

기존 방식인 ECU마다 UART 선 연결을 하면 배선 폭증, 통신 충돌, 낮은 신뢰성 문제가 있어서, 하나의 공용 버스에 모든 ECU를 붙이는 CAN이 생겨났습니다.
CAN은 높은 신뢰성과 결정적 지연 시간을 제공합니다.

### 특징

- **Multi-node Bus 구조**
- **메시지 ID 기반 통신 (주소 없음)**
    - 특정 수신자 개념이 없는 브로드캐스트 방식으로, 모든 ECU는 메시지 ID만 필터링해서 받습니다.
    - 예: `ID 0x100 -> 엔진 RPM`, `ID 0x200 -> 브레이크 입력`, `ID 0x300 -> 차량 속도`
- **하드웨어 기반 중재(Arbitration)**
- **차동 신호 기반 물리 계층**
- **Half-Duplex**
- **CRC + ACK + 자동 재전송**
- **실시간성 보장**: 우선수위 기반으로 충돌 재전송 비용이 거의 없습니다.
    - 예: 브레이크 ECU 메시지가 항상 먼저 통과합니다. 
- 고속(최대 1 Mbps)
    - CAN FC는 최대 8 Mbps 

### 물리적 배선 구조
CAN은 MCU 내부 Controller와 외부 Transceiver를 사용합니다. 
모든 노드가 같은 선을 공유하고, Master/Slave가 없는 Multi-Master 구조입니다.

```
        MCU A                          MCU B
 ┌───────────────┐               ┌───────────────┐
 │ CAN Controller│               │ CAN Controller│
 └──────┬────────┘               └───────┬───────┘
        │ TX/RX                          │ TX/RX
        ▼                                ▼
 ┌────────────────┐               ┌────────────────┐
 │ CAN Transceiver│               │ CAN Transceiver│
 └──────┬───────┬─┘               └──────┬───────┬─┘
        │       │                        │       │
      CAN_H   CAN_L                   CAN_H   CAN_L
        │       │                        │       │
========┴=======┴========================┴=======┴========
        │       │                        │       │
      120Ω     120Ω                 (Termination Resistors)
```
- **CAN Controller** (MCU 내부): 프레임 생성, 중재 처리, CRC, 필터링, 버퍼 관리를 담당합니다.
- **CAN Transceiver** (외부 칩): 전압 변환을 담당하여, MCU TX/RX(3.3V)를 CAN_H/L 차동 신호로 변환합니다.
- **CAN_H / CAN_L** : 차동 신호선
    - CAN_H: Dominant(0)인 경우 3.5V, Recessive(1)의 경우 2.5V
    - CAN_L: Dominant(0)인 경우 1.5V, Recessive(1)의 경우 2.5V
    - 전압 차이로 외부 노이즈가 +1V 유입되었더라도 전압 차이 2V는 그대로 유지될 수 있습니다. 이는 자동차 환경에서 매우 유리합니다.
- **Termination Resistor (120Ω)** : 버스 양 끝단 배치하여, 파형 반사 오류를 막습니다. 


### 프레임 구조
```
| SOF | ID | RTR | Control | Data | CRC | ACK | EOF |
```
- **ID** : 메시지 우선순위와 의미
    - ID 값이 작을수록 우선순위가 높습니다: `0x100 > 0x300이기 때문에, 0x100 먼저 전송됨`
    - 필요한 ID가 아니라면 필터링이 진행됩니다: `MASK 활용해서 필터링`
- **DLC** : 데이터 길이
- **DATA** : 최대 8바이트 (CAN FD는 64바이트)
- **CRC** : 오류 검출
- **ACK Slot** : 수신 노드 확인 응답
### 동작 원리
#### 내부 구조
```
CAN Controller Peripheral
├─ TX Mailbox
├─ RX FIFO
├─ Arbitration Logic
├─ Bit Timing Unit
├─ ID Filter
├─ CRC Generator/Checker
├─ Error Management Unit
├─ Control Register
├─ Status Register
```
#### 송신 흐름

1. CPU가 TX Mailbox에 ID + DATA 기록
2. Bus Idle 감지
3. SOF 송신
4. Arbitration 수행
5. 승리 시 프레임 송신 지속
6. CRC 송신
7. ACK Slot 확인
8. EOF 송신
9. TX Complete Interrupt 발생

#### 수신 흐름

1. 모든 노드가 Bus 감시
2. SOF 감지
3. ID 수신
4. ID Filter 검사
5. CRC 확인
6. ACK Slot Dominant 출력
7. RX FIFO 저장
8. RX Interrupt 발생

### 중재 원리

CAN은 전기적으로 다음 특성을 사용한다.

- Dominant = 0
- Recessive = 1

#### 중재 과정

1. 여러 노드가 동시에 ID 전송
2. 각 비트를 동시에 비교
3. 내가 1을 내보냈는데 Bus가 0이면 패배
4. 패배 노드는 즉시 송신 중단

→ ID 값이 가장 작은 메시지가 우선 전송됨


### 활용 분야

- 자동차 ECU 네트워크
- 산업 제어 시스템
- 로봇 제어
- 의료 장비

### 요약
> CAN은 하드웨어 기반 중재와 차동 신호를 이용해  
> 다중 노드 환경에서도 실시간성과 신뢰성을 보장하는 차량 네트워크 프로토콜입니다.

## 2. LIN이란
LIN(Local Interconnect Network)은 저속·저비용 센서 및 액추에이터 제어를 위한 Master-Slave 기반 단선 네트워크 통신 프로토콜입니다.

CAN은 트랜시버때문에 비용이 비싸고, MCU도 고성능이 필요하며, 속도는 불필요하게 빠릅니다.
이에 반해 창문 스위치, 사이드미러 모터, 실내등, 시트 조절, 선루프 등은 초당 수십바이트면 충분하기 때문에 싸고 단순한 통신인 LIN이 탄생했습니다.
- LIN은 말단, CAN은 중앙으로 하여 실제 차량 구조를 만듭니다.
### 특징
- **Master-Slave 구조**
- **Single-wire 통신**
- **UART 기반 변형 프로토콜**
- **Time-triggered 스케줄링**
- **저속 (최대 20kbps)**
- **저비용 구현**
### 물리 배선 구조
```
                Master Node (MCU)
 ┌─────────────────────────────────────┐
 │ LIN Controller (UART 기반)          │
 └───────────────┬─────────────────────┘
                 │ TX/RX
                 ▼
         ┌─────────────────┐
         │ LIN Transceiver │
         └───────┬─────────┘
                 │
               LIN BUS  (Single Wire)
                 │
   -------------------------------------------------
     │                │                │
     ▼                ▼                ▼
 Slave A           Slave B           Slave C
 ┌────────┐       ┌────────┐       ┌────────┐
 │Transcv │       │Transcv │       │Transcv │
 └───┬────┘       └───┬────┘       └───┬────┘
     │                │                │
    MCU              MCU              MCU

(공통 GND 연결)
```
- 단일 신호선 (1개) + GND
- Pull-up 저항 사용


### 프레임 구조
Master 헤더와 Slave 응답으로 이루어져 있습니다.
#### Master 헤더
```
BREAK | SYNC | ID
```
- **Break** : 버스를 강제로 LOW로 만들어서 프레임 시작을 표시합니다.
- **Sync** : Baudrate를 동기화하는 바이트를 제공하여, Slave들이 내부 오실레이터로 자동 보정합니다. (저가 MCU가 가능한 이유)
- **ID** : 응답할 Slave 식별
- **Checksum** : 오류 검출

#### Slave 응답
```
DATA (1~8 bytes) | CHECKSUM
```

### 동작 원리
#### 내부 구조
```
LIN Controller
├─ UART Engine
├─ Scheduler
├─ Frame Handler
├─ TX Buffer
├─ RX Buffer
├─ Checksum Generator
```
#### 송신 흐름 (Master)

1. Scheduler에 따라 프레임 시작
2. Break 송신
3. Sync 전송
4. ID 송신
5. Slave 응답 대기
6. 데이터 수신

#### 수신 흐름 (Slave)

1. Break 감지
2. Sync 수신 및 Baud 동기화
3. ID 비교
4. 일치 시 데이터 송신 또는 수신
5. Checksum 검증


### 활용 분야
- 자동차 도어 모듈
- 시트 제어
- 미러 제어
- 실내 센서

### 요약
> LIN은 CAN을 보조하는 저속·저비용 제어 네트워크로,  
> Master 기반 스케줄 통신 구조를 사용합니다.



## 3. Ethernet이란
Ethernet은 유선 기반의 고속 패킷 기반 네트워크 통신을 위한 표준 통신 기술이며,  
자동차 및 산업 분야에서는 실시간 제어를 위한 Automotive Ethernet으로 확장되고 있습니다.

카메라, 라이다, OTA 업데이트, ADAS 등은 기존 차량 통신의 속도 (FlexRay인 경우 최대 10 Mbps)에 절대 부족하기 때문에, 100 Mbps ~ 1 Gbps 인 Ethernet을 도입했습니다.

### 특징
- **MAC + PHY 계층 구조**
- **Full-Duplex 통신**
- **고대역폭 (100Mbps ~ Gbps)**
- **패킷 기반 프레임 전송**
- **스위치 기반 네트워크 구조**
- **IP 기반 확장 가능**

### 물리적 배선 구조

```
            MCU / SoC
        ┌──────────────────┐
        │ Ethernet MAC     │   (Protocol Layer)
        └────────┬─────────┘
                 │ MII / RMII / RGMII
                 ▼
        ┌──────────────────┐
        │ Ethernet PHY     │   (Physical Layer)
        └────────┬─────────┘
                 │ Differential Signals
                 ▼
        ┌──────────────────┐
        │ Transformer      │   (Isolation + Impedance Match)
        └────────┬─────────┘
                 │
        ==========================
          Twisted Pair Cable
        ==========================
                 │
        ┌──────────────────┐
        │ Switch / ECU     │
        └──────────────────┘
```


## 프레임 구조
```
Preamble
Destination MAC
Source MAC
Type
Payload (46~1500 bytes)
CRC
```
### 동작 원리
#### 내부 구조
```
Ethernet Controller
├─ MAC Engine
├─ DMA Engine
├─ TX Descriptor Ring
├─ RX Descriptor Ring
├─ CRC Generator
├─ Interrupt Controller
```
#### 송신 흐름
1. CPU가 TX Buffer에 Frame 저장
2. DMA가 MAC으로 데이터 이동
3. MAC이 Frame 생성
4. PHY 통해 신호 송출
5. TX Complete Interrupt 발생


### 수신 흐름

1. PHY가 신호 수신
2. MAC 프레임 복원
3. CRC 검사
4. DMA로 RX Buffer 저장
5. RX Interrupt 발생

### 활용 분야

- 차량 백본 네트워크
- 카메라 영상 전송
- OTA 업데이트
- 산업 자동화
- IoT 게이트웨이

### 요약

> Ethernet은 고대역폭과 표준 네트워크 스택을 활용해  
> 차량 및 산업 시스템에서 데이터 중심 통신을 담당하는 고속 네트워크 기술이다.






# 전체 비교 요약

| 항목 | CAN | LIN | Ethernet |
------|------|------|---------
속도 | 중속 | 저속 | 초고속
구조 | Multi-node Bus | Master-Slave | Switch Network
실시간성 | 높음 | 중간 | TSN 적용 시 높음
비용 | 중간 | 매우 낮음 | 높음
주용도 | ECU 제어 | 센서/액추에이터 | 영상/백본

> CAN과 LIN은 실시간 제어 중심 네트워크이며,  
> Ethernet은 대역폭 중심 데이터 네트워크이다.  
> 이 세 가지는 현대 차량 및 산업 임베디드 시스템의 핵심 통신 인프라를 구성한다.






