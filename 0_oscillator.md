# 실제 개념
## 오실레이터란
오실레이터는 클록 신호를 생성하는 하드웨어 회로로, 전기 신호를 주기적으로 진동시켜 일정한 주파수의 파형을 만들어냅니다.
이 클록 신호를 기준으로 CPU, 버스, 주변장치가 동기적으로 동작할 수 있는 시간 기준(Time Reference) 이 형성됩니다.

## 1. 내부 오실레이터란
Internal Oscillator는 MCU 내부에 내장된 RC(저항-커패시터) 기반 발진 회로로, 외부 부품 없이 즉시 클록을 생성할 수 있는 기본 클록 소스입니다.
### 물리적 구성
일반적으로 다음 요소들로 구성됩니다.
* Resistor (R)
* Capacitor (C)
* Comparator
* Inverter
* Feedback Loop
RC 충·방전 주기를 이용해 사각파 형태의 클록을 생성합니다.

### 특성
RC 기반 구조 특성상 공정 편차, 온도 변화, 전압 변화에 미감하며, 일반적으로 1~5% 수준의 주파수 오차가 발생합니다.
* 예: 10 MHz 기준으로 5% 편차라고 하면, 9.5 MHz ~ 10.5 MHz 범위에 해당합니다.
* 이는 Timer 오차, Delay 오차, UART baud mismatch, 통신 프레임 에러 등 타이밍 동기화 오류를 유발할 수 있는 수준입니다.

### 장점
* Power ON 직후 즉시 동작 가능
* 외부 부품 불필요
* 기계적 진동/충격에 강함
* 외부 Crystal 실패 시 자동 fallback 클록으로 사용 가능
자동차 MCU의 경우, 전원 인가 직후 즉시 동작해야 하는 안전 요구사항 때문에 Internal Oscillator는 필수 구성 요소입니다.

### 종류
1. IMO (High Speed Internal Oscillator)
    * 수 MHz ~ 수십 MHz
    * Boot CLock (boot 과정에서 flash 접근, memory copy 등 초기 부팅 안정성과 속도가 중요하기 때문)
    * Sysstem fallback
2. ILO (Low Speed Internal Oscillator)
    * 수 kHz
    * Watchdog
    * Sleep wakeup (CPU/PLL/Crystal OFF 가능) 
3. Backup Internal Oscillator
    * 주 클록 이상시, 시스템 생존 및 안전 동작 보장을 위한 독립 백업 클록입니다. 
### 한계
1. 고속 한계: 수십 MHz 이상도 힘들고, PLL 기준 클록으로도 부적합합니다.
2. 노이즈에 취약
3. 통신 신뢰성 한계: CAN, USB, Ethernet 에서 요구하는 타이밍 허용 범위를 초과합니다.

## 2. External Crystal이란
External Crystal은 석영(Quartz)의 기계적 공진 특성을 이용해 매우 정확한 기준 주파수를 제공하는 외부 기준 발진 소자입니다.
크리스탈 자체는 발진기 아니지만, 다음을 제공하며 MCU 내부 증폭 회로와 결합되어 Oscillator를 구성합니다.
* 공진 특성 제공
* 주파수 기준 역할 수행
### 정확도
10~30 ppm의 정확도로, (1ppm = 0.0001%) 10 MHz를 기준으로 200 Hz 차이가 나, 내부 오실레이터에 비해 2500배 이상 정확도가 차이납니다.
### 역할
* PLL/FLL 기준 클록(reference clock)
* CPU 고속 시스템 클록 생성 기준
* UART/CAN/USB/Ethernet 통신 기준
* Timer 및 RTC 기준
### 단점
* 기동 시간 필요 (1ms ~ 수십 ms)
* 외부 부품 필요
* PCB 공간/EMI 고려 필요
## 3. PLL(Phase Locked Loop)이란
PLL은 기준 클록(reference clock)에 위상과 주파수를 동기화하면서 고속 클록을 생성하는 주파수 합성기입니다. 
```
Crystal 20 MHz
PLL ×10
→ CPU 200 MHz
```
PPL은 기준 클록의 정확도를 유지한 채 CPU가 요구하는 고속 클록을 생성할 수 있습니다.




### Boot 시 Oscillator 흐름
1. Power ON
2. Internal Oscillator 자동 ON  → 목표: 즉시 시스템 활성화
3. Reset release
4. Boot ROM 실행
5. CLock configuration code 실행
6. External Cryastal enable
7. Crystal stable Flag 확인
8. PLL enable
9. PPL lock 대기
10. System Clock switch
