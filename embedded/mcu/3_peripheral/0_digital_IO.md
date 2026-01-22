# 실제 개념
## GPIO란
GPIO(General Purpose Input Output)는 MCU 내부 디지털 로직과 외부 물리세계인 전압 신호를 연결하는 범용 디지털 입출력 주변장치입니다.

CPU는 메모리 주소 공간에 대한 LOAD, STORE 명령어만 수행할 뿐, 핀에 인가되는 전압과 전류, 신호 타이밍 같은 물리 현상을 다룰 수 없습니다. 

GPIO는 이를 해결하기 위해 다음 역할을 수행합니다. 
* **[Input path]** 외부 핀 전압 -> 디지털 비트로 변환
* **[Output path]** CPU 디지털 비트 -> 핀 전압을 생성

이를 위해 GPIO는 Input Buffer, Output Driver, 제어 레지스터, 동기화 회로 등을 가진 독립적 하드웨어 블록을 구성합니다.

### CPU 접근 방법
GPIO 레지스터는 RAM이 아니라 주변장치 내부에 있는 레지스터입니다. 
하지만 Memory-mapped 구조를 통해 CPU는 이 레지스터에 메모리 주소처럼 접근이 가능합니다. 

즉, GPIO의 IDR / ODR 레지스터도 CPU에서 `LOAD, STORE` 명령어로 제어가 가능합니다. 

### 물리적 관점
MCU의 핀은 여러 주변장치들로 연결되어 있을 수 있으며, Pin Mux를 통해 어떤 기능을 사용할 지 결정할 수 있습니다. 
```
                Physical PAD
                     |
               ┌──── PinMux ────┐
               |                |
        GPIO Logic         Peripheral Logic
       (Input/Output)    (UART/SPI/CAN/ADC)
```
위 그림처럼 GPIO Logic을 기본 설정으로 갖고 있고, Alternative Function Mode (AF)로 다른 주변장치를 사용할 수 있습니다. 

GPIO와 연결되어 있다면, 다음 GPIO관련 블록을 활성화 시켜서 동작할 수 있습니다.
```
Pin PAD
 ├─ Input Buffer (입력 경로)
 ├─ Output Driver (출력 경로)
 ├─ Pull-up / Pull-down Network
 ├─ Mode Select MUX
 └─ Peripheral Function MUX
```
### 1. Input 모드의 동작 과정
Input 모드란, 외부 전압 신호를 디지털 값으로 변환하는 경로입니다. 
```
외부 핀 전압
-> Input Buffer (Schmitt Trigger)
-> Synchronizer (Clock Domain Sync)
-> IDR (Input Data Register)
-> CPU Bus Read
```
1. 핀 전압 인가: 버튼/스위치/센서 등 외부 회로에서 핀으로 전압이 인가됩니다.
2. Input BUffer: 인가된 전압을 임계 전압을 기준으로 LOW/HIGH를 판정합니다. 
    * 핀 모드로 **Pull-up / Pull-down 구조**를 사용
        * 문제: GPIO핀이 기본적으로 고임피던스(High-Z) 상태이므로, 외부 입력이 없으면 노이즈/정전기/커패시턴스의 영향으로 임의 전압이 튈 수 있습니다.
        * 해결: 내부 Pull 저항을 활용하여 기본값을 HIGH로 두거나, LOW로 둘 수 있습니다.
    * **Schmitt Tigger** 회로로 입력 전압의 상승과 하강에 다른 임계값을 사용해서 노이즈와 떨림을 제거합니다. 
        ```
        입력
        ~~~~~~^^^~~~~~~
        High Th ------------
        Low Th  ------------

        출력
        000000111111000000  ← 안정
        ```
3. Synchronizer: 외부 입력 신호는 MCU 클록과 비동기 상태이므로, 동기화 플립플롭으로 입력 신호를 내부 클록 도메인으로 안전하게 정렬합니다.
4. IDR 레지스터에 반영: GPIO 하드웨어가 현재 핀 상태를 보여주는 역할을 담당합니다. 
5. CPU Read: 버스 시스템을 통해 IDR 주소를 읽고 값을 획득합니다. 

### 2. Output 모드의 동작 과정
Output 모드란, CPU의 디저털 값을 외부 핀 전압으로 반환하는 경로입니다.
```
CPU STORE 명령
-> Bus Write
-> ODR Register
-> Output Driver
-> Pin PAD
-> 외부 회로 동작
```
1. CPU STORE 실행: CPU가 ODR로 STORE 명령어를 실행합니다.
2. ODR 저장
3. **Output Driver** 활성화: 내부 트랜지스터를 제어하여 핀에 전압을 생성합니다.
    * Output Driver 타입으로 **Push-Pull / Open-Drain 구조** 사용
    * Push-Pull 구조: HIGH/LOW를 직접 구동합니다. LED, 디지털 출력용으로 적합합니다.
    * Open-Drain 구조: LOW만 직접 구동합니다. I2C, 다중 장치 공유선에 사용됩니다.
4. Pin PAD로 전압 출력: HIGH(VDD) 또는 LOW(GND) 전압이 외부로 출력됩니다.


## 임베디드 개발자가 하는 작업
### 0. 핀을 초기화
임베디드 개발자는 C 코드로 다음을 설정합니다. 
### 1) 핀 정의
```C
#define LED_PORT P13
#define LED_PIN  0
// OR
#define BUTTON  &MODULE_P00,7
```
다음 코드처럼 사용할 핀을 먼저 정의합니다.

### 2) 핀 모드 설정
```C
IfxPort_setPinMode()
```
해당 함수로 아래를 설정합니다.
* Input / Output : 하나의 외부 핀이 두개 역할 모두 하는 것은 불가합니다.
* Pull-up / Pull-down
* Push-pull / Open-drain
* GPIO / Alternate Function
* Drive Strength

## 예시
### 장치
* 입력 장치로는 버튼, 도어 스위치, 센서 트리거 등이 있다.
* 출력 장치로는 LED, 릴레이 제어, enable 신호, 모터 방향이 있다.
* 실무에서는 상태 감지, 장치 제어, 전원 관리, 안전 신호 등에 사용될 수 있다.


### 동작 1) 버튼 입력
1. 버튼을 누릅니다.
1. 핀에 아날로그 전압이 인가됩니다.
1. Pull-up/down 회로로 기본 상태가 안정화됩니다.
1. Input Buffer에서 디지털 `0/1`로 변환합니다.
1. Synchornizer를 통과합니다.
1. IDR에 저장합니다.
1. CPU가 polling 또는 interrupt 처리합니다.

### 동작 2) LED 출력 
1. CPU 코드가 실행됩니다.
1. ODR 레지스터에 저장됩니다.
1. Output Driver가 활성화됩니다.
1. 핀에 전압이 생성됩니다.
1. LED 회로에 전류가 흐릅니다.
1. LED가 발광합니다.


# 임베디드 개발자로서 주의할 점
## GPIO는 설정을 잘 해줘야 한다. 
LED가 안켜지면 코드 로직보다 다음같이 수많은 설정값들 중 하나가 오류일 가능성이 크다. 
* MUX 설정
* Pull 설정
* Output type
* Drive strength
* Clock enable
# 내가 이해한 바
