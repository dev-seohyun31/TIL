# 실제 개념
## GPIO란
GPIO(General Purpose Input Output)는 디지털 I/O를 담당한는 Peripheral로,
MCU 내부 디지털 로직과 외부 물리 세계(전압)를 연결하는 범용 입출력 인터페이스입니다.

CPU는 메모리 주소 공간을 통해 연산을 할 뿐, 물리적인 전압을 직접 처리하지 않습니다. 
GPIO는 input buffer와 output driver를 통해 전압과 디지털 비트 사이의 변환을 담당하는 하드웨어 인터페이스입니다. 

## 물리적 관점 
GPIO는 MCU의 외부 핀 중 일부를 담당하며, 핀 MUX를 통해 다음 기능 중 하나로 선택될 수 있습니다: `GPIO / UART / SPI / CAN / ADC / PWM / ...`
핀은 여러 모드 중 선택하여 사용합니다. 
### 1. Input 모드
핀으로부터 외부 전압을 읽어서 디지털 값으로 변환하는 모드입니다.
1. 핀에 전압이 인가됨
2. Input buffer 가 Schmitt Trigger 로직으로 아날로그 신호를 디지털 신호로 변환함
3. Synchronizer가 외부 비동기 입력 신호를 MCU 내부 시스템 클록에 맞춰서 동기화시킴
4. IDR (Input Data Register)에 비트 저장

* 단, GPIO 핀은 Input 모드에서 기본적으로 떠 있는 상태 (즉, 고임피던스) 입니다.
    * 외부 입력이 없는 경우, 노이즈 / 정전기 / 커패시턴스 에 의해 값이 전압 값이 확 튀는 등 불안정해질 수 있으므로, Pull-up / Pull-down 저항으로 기본 전압 상태를 만들어줍니다.
* Input Pull-up: VDD와 약한 저항을 연결시켜, 기본값은 HIGH입니다.
* Input Pull-down: GND와 약한 저항을 연결시켜, 기본값은 LOW입니다.
### 2. Output 모드
CPU가 계산한 결과 값을 기반으로 핀에 전압을 생성하는 모드입니다.
1. CPU가 GPIO Output Register 주소로 STORE 수행
2. ODR (Output Data Register)에 저장
3. Output Driver가 활성화됨
4. 핀에 전압을 생성



## 기계적 동작 순서
### 0. 초기화: 핀 설정하기
임베디드 개발자는 C 코드로 다음을 설정합니다. 
### 1) 핀을 정의
```C
#define LED_PORT P13
#define LED_PIN  0
// OR
#define BUTTON  &MODULE_P00,7
```
다음 코드처럼 사용할 핀을 먼저 정의합니다.

### 2) 핀 모드를 설정
```C
IfxPort_setPinMode()
```
해당 함수로 아래를 설정합니다.
* Input / Output : 하나의 외부 핀이 두개 역할 모두 하는 것은 불가합니다.
* Push-pull / Open-drain
* GPIO / Alternate Function
#### 1-1. Input 동작 순서
1. 외부 장치(버튼, 센서)에 전압 변화 발생
1. 핀 PAD에 아날로그 전압 인가
1. Pull 저항이 기본 전압 안정화시킴 
1. Input Buffer가 전압을 디지털 값으로 변환
1. Synchronizer 장치가 MCU 클록 도메인으로 동기화
1. IDR 레지스터 비트에 저장
1. CPU가 Polling 또는 Interrupt로 처리

#### 1-2. Output 동작 순서
1. CPU 작성 코드에서 GPIO 출력 명령 실행
1. ODR 레지스터에 출력값 저장
1. output driver가 활성화
1. 핀 PAD에 전압 생성
1. 외부 장치 동작: LED, 릴레이, IC enable 등


## 예시
### 장치
* 입력 장치로는 보튼, 도어 스위치, 센서 트리거 등이 있다.
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
