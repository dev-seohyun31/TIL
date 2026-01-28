# 실제 개념
다음 통신 세가지 UART, SPI, I2C는 보드 내부 통신(Chip-to-Chip) 용도로 설계된 인터페이스입니다.
```
MCU
 ├─ UART → Debug
 ├─ SPI → Flash
 └─ I2C → Sensor
```
## 1. UART란
UART(Universsal Asynchronous Reciever/Transmitter)란, 클록 공유 없이 직렬 통신 방식으로 바이트 데이터를 TX/RX 핀을 통해 송수신하는 비동기 통신 주변장치입니다. 
### 특징
* **P2P 구조**
* **비동기 통신**: 클록선을 사용하지 않고, Start bit와 Baud Rate 약속으로 타이밍을 맞춰서 통신합니다.
    * Buad Rate는 bps 단위의 전송 속도를 의미하고, 예를 들어 9600은 9600bps로 전송한다는 의미합니다.
    * 내부적으로 다음 흐름으로 비트 타이밍이 생성됩니다: `Peripheral Clock -> Buad Rate Generator -> Bit 타이밍 생성`
* **Full-Duplex**: TX, RX 선이 분리되어 있어 송신과 수신을 동시에 수행할 수 있습니다.
* **Push-Pull 출력 구조**: HIGH, LOW를 모두 적극적으로 구동하여 빠른 신호 전환이 가능합니다. (대략 1Mbps)
* **직렬 통신 방식**: 데이터를 시간 순서대로 1비트씩 전송합니다.

### 물리적 배선 연결
UART의 기본 배선 연결은 다음과 같습니다.
```
        MCU (UART)                           상대 장치 (UART)
 ┌──────────────────┐                 ┌──────────────────────┐
 │                  │  TX  ────────▶ │  RX                  │
 │                  │  RX  ◀──────── │  TX                  │
 │                  │  GND ─────────  │  GND                 │
 └──────────────────┘                 └──────────────────────┘
```
* TX (Transmit): 송신선을 GPIO 핀에 직접 연결
* RX (Receive): 수신선을 GPIO 핀에 직접 연결
* GND (Reference Ground): 기준 전압을 직접 연결 

TX와 RX는 교차 연결되며, GND는 양쪽 장치의 전압 기준을 맞추기 위해 반드시 공유되어야 합니다.

### 프레임 구조
UART는 고정된 프레임 형식을 사용합니다. 다음 형식은 일반적인 8N1 설정입니다. 
```
Idle(1)
-> Start Bit (0) + Data Bits (8bit) + Parity (Optional) + Stop Bit (1)
```
### 동작 원리
#### 내부 구조
```
UART Peripheral
 ├─ TX Shift Register (송신 시리얼화)
 ├─ RX Shift Register (수신 시리얼화)
 ├─ Baud Rate Generator
 ├─ Control Register
 ├─ Status Register
 ├─ TX Buffer / RX Buffer (FIFO 포함 가능)
```
#### 송신 흐름 (TX)
1. CPU가 UART TX 레지스터에 데이터 기록
2. UART 하드웨어가 Shift Register에 적재
3. Baud Rate Generator가 비트 타이밍 생성
4. 프레임 생성: `Start bit → Data bits → Stop bit` 
5. 생성된 비트 스트림을 TX 핀으로 출력
6. 전송 완료 시 TX Empty / TX Complete 플래그 설정 또는 인터럽트 발생
#### 수신 흐름 (RX)
1. RX 핀에서 Start bit 감지
2. Baud Rate 기준으로 샘플 타이밍 시작
3. 각 비트 중앙에서 데이터 샘플링
4. RX Shift Register에 8비트 누적
5. Stop Bit 검사 (Framing Error 체크)
6. RX Buffer에 저장
7. RX Ready 플래그 설정 또는 인터럽트 발생

### 활용 분야
UART는 구조가 단순하고 구현이 쉬워 디버깅 및 범용 통신 인터페이스로 가장 많이 사용됩니다.
* 디버깅 로그 출력: `printf("ok");`를 남기면, 문자 하나씩 프레임을 생성하여 터미널로 출력합니다..
* 펌웨어 다운로드 및 부트로더 통신
* 모듈 제어 통신: `MCU ↔ WiFi 모듈`, `MCU ↔ GPS 모듈`


## 2. SPI란
SPI(Serial Peripheral Interface)는 Master가 클록을 생성하여 Slave와 동기식 직렬 방식으로 고속 데이터를 송수신하는 통신 주변장치이다.
### 특징
* **Master-Slave 구조**: Master가 Clock 생성, 통신 시작/종료, Slave 선택을 담당
* **동기 통신**: SCLK(Clock) 선을 공유하여 비트 타이밍을 물리적으로 동기화
* **Full-Duplex**: MOSI/MISO 선이 분리되어 송수신 동시 가능합니다.
* **Push-Pull 출력 구조**: 빠른 신호 전환이 가능하여 고속 통신에 유리합니다. (수 MHz ~ 수십 MHz)
* **직렬 통신 방식**: 클럭마다 1비트씩 전송합니다.

### 물리적 배선 구조
SPI 기본 연결 구조는 다음과 같습니다.
```
   Master MCU                   Slave Device
 ┌────────────┐                ┌────────────┐
 │            │ MOSI ───────▶ │ MOSI       │
 │            │ MISO ◀─────── │ MISO       │
 │            │ SCLK ───────▶ │ SCLK       │
 │            │ CS   ───────▶ │ CS         │
 │            │ GND  ────────  │ GND        │
 └────────────┘                └────────────┘
```
* MOSI (Master Out Slave In): Master → Slave 데이터 전용선 (Slave 여러 개인 경우, 공유)
* MISO (Master In Slave Out): Slave → Master 데이터 전용선 (Slave 여러 개인 경우, 공유)
* SCLK (Serial Clock): Master가 생성하는 클록 (Slave 여러 개인 경우, 공유)
* CS (Chip Select): 통신할 Slave 선택하여 연결
* GND: 기준 전압

### 프레임 구조
SPI는 UART처럼 고정된 프레임 구조가 없습니다. 
단, `클록 N번 = N비트 전송` 규칙을 갖고 있고, 프레임 구조는 Slave 디바이스 datasheet에 정의됩니다. 예를 들어, Slave datasheet는 다음과 같이 정의될 수 있습니다.
```
READ:
CS LOW
0x0B (Command)
24bit Address
Dummy Byte
Data Byte
CS HIGH
```
Master에서는 코드를 다음과 같이 짤 수 있습니다.
```C
CS_LOW();

spi_send(0x90);   // READ + address
spi_send(0x00);   // dummy clock
data = spi_send(0x00);

CS_HIGH();
```
따라서 Master는 Slave의 datasheet를 보고 프레임을 작성하고, 클록 속도가 빠른만큼 데이터도 빠르게 전송될 가능성이 큽니다.

### 동작 원리
SPI는 Shift Register 기반의 동기식 전송 구조입니다.
#### 내부 구조
```
SPI Peripheral
 ├─ Shift Register (TX/RX 동시 사용)
 ├─ Clock Generator (Master 전용)
 ├─ Control Register
 ├─ Status Register
 ├─ TX Buffer / RX Buffer (FIFO 포함 가능)
 ├─ Chip Select Controller
```
#### 송신 흐름
1. CPU가 SPI TX Data Register에 데이터 기록
2. SPI 하드웨어가 Shift Register에 데이터 로드
3. Master가 CS를 LOW로 설정하여 Slave 선택
4. Clock Generator가 SCLK 생성
5. 클럭 엣지마다 MOSI 핀으로 비트 출력
6. 동시에 Slave에서 들어오는 MISO 비트 수신
7. 8클럭 후 1바이트 전송 완료
8. RX Shift Register 값이 RX Buffer로 이동
9. 필요 시 CS HIGH로 통신 종료
#### 수신 흐름
1. Master가 Dummy Data 또는 실제 데이터를 TX Register에 기록
2. Clock 생성 시작
3. 클럭 엣지마다 MISO 핀에서 비트 샘플링
4. RX Shift Register에 비트 누적
5. 8클럭 후 1바이트 수신 완료
6. RX Buffer 저장 및 RX Ready 플래그 설정
> SPI에서는 수신만 하는 개념은 없고, Clock 생성을 위해 항상 송신 동작이 동반됩니다.

### 활용 분야
SPI는 고속 전송이 필요한 보드 내부 통신에 주로 사용됩니다.
* External Flash Memory
* LCD / OLED Display
* IMU, 고속 센서
* ADC / DAC
* Ethernet PHY

## 3. I2C란
I2C(Inter-Integrated Circuit)는 두 개의 선(SDA, SCL)만으로 여러 장치를 주소 기반으로 연결하는 동기식 버스 통신 인터페이스이다.
### 특징
* **Master-Multi Slave 구조**: 하나의 Master에 여러 Slave 연결 가능합니다.
* **동기 통신**: SCL 클록선을 공유하여 비트 타이밍 동기화합니다.
* **Half-Duplex**: SDA 한 선을 송수신이 공유합니다.
* **Open-Drain 출력 구조**: LOW만 직접 구동하고, HIGH는 Pull-up 저항으로 생성합니다. 속도가 느릴 수 있습니다. ()
* **주소 기반 통신**: Slave 선택을 주소로 선택합니다.
* **직렬 통신 방식**
### 물리적 배선 구조
I2C는 버스 구조를 사용합니다.
<img width="300" height="180" alt="image" src="https://github.com/user-attachments/assets/79ca221e-b612-4005-b806-2dc4ab4f1504" />

* SDA (Serial Data): 데이터 라인
* SCL (Serial Clock): 클록 라인
* Pull-up 저항 필수
* 모든 장치가 동일 선 공유
### 프레임 구조
I2C는 프로토콜 기반 프레임 구조를 가집니다.
```
START
[Slave Address + R/W bit]
ACK
[Data Byte 1]
ACK
[Data Byte 2]
ACK
...
STOP
```
* START 조건: 버스 시작 신호를 보냅니다.
    * SDA: HIGH -> LOW로 변화
    * SCL: HIGH 상태
* Slave Address + R/W bit (8bit) 
    * Master가 `7bit 주소 + 1bit R/W`로, `0b1010000 + 0 (Write)` 송신
* ACK bit: Slave가 SDA를 LOW로 당기면서 응답합니다.
* Data Byte: 8비트 단위로 전송합니다.
* STOP 조건: 통신 종료 신호를 보냅니다.
    * SDA: LOW -> HIGH로 변화
    * SDA: HIGH 상태 
### 동작 원리

#### 내부 구조
```
I2C Peripheral
 ├─ State Machine (Protocol Controller)
 ├─ Clock Generator (Master 전용)
 ├─ Address Comparator
 ├─ SDA/SCL Open-Drain Driver
 ├─ Control Register
 ├─ Status Register
 ├─ TX Buffer / RX Buffer
```
#### 송신 흐름 (Master -> Slave)
1. CPU가 START 조건 생성 요청
2. I2C 하드웨어가 START Condition 생성: `SDA: HIGH → LOW (SCL HIGH 유지)`
3. Master가 Slave Address + R/W 비트 송신
4. Slave가 ACK 응답
5. CPU가 TX Buffer에 데이터 기록
6. 8비트 단위로 SDA 출력
7. Slave ACK 수신
8. 반복 전송 또는 STOP 조건 생성: `SDA: LOW → HIGH (SCL HIGH 유지)`

#### 수신 흐름 (Slave -> Master)
1. Master가 START 생성
2. Address + Read 비트 송신
3. Slave ACK
4. Slave가 SDA에 데이터 출력
5. Master가 클럭 기반으로 비트 샘플링
6. 8비트 수신 완료 후 Master ACK 또는 NACK 전송
7. STOP 조건 생성

### 활용 분야
I2C는 다수의 저속 주변 장치 연결에 최적화되어 있습니다.
특히, PCB에 붙어있는 외부 장치(예: 온도 센서, Flash memory 칩, 이더넷 컨트롤러 등)에서 잘 쓰입니다.
* 온도 / 습도 센서
* RTC
* EEPROM
* 전원 관리 IC
* 센서 설정 레지스터 제어




## 최종 구조 차이

| 항목             | UART              | SPI                    | I2C                      |
| -------------- | ----------------- | ---------------------- | ------------------------ |
| Clock 생성       | 내부 Baud Generator | Master Clock Generator | Master Clock Generator   |
| Shift Register | TX/RX 분리          | TX/RX 공용               | 프로토콜 기반 처리               |
| 프레임 제어         | HW 자동 생성          | 외부 프로토콜 의존             | State Machine 제어         |
| 동작 트리거         | Start Bit 감지      | Clock Edge             | START/STOP 조건            |
| 버퍼 구조          | TX/RX FIFO        | TX/RX FIFO             | TX/RX Buffer + ACK Logic |



