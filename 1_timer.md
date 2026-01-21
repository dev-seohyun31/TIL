# 실제 개념

## Timer란
timer는 MCU 내부에서 시간 개념을 만들어주는 하드웨어 주변장치입니다. 
CPU는 클록 엣지, 명령 실행, 레지스터 값 변화만 인식할 뿐, "5ms 지났다", "1초 대기" 등 시간 개념을 직접 인식하지 못합니다. 
따라서 Timer는 클록 신호를 기반으로 카운트를 누적하여, 시간 흐름을 숫자로 변환하는 역할을 담당합니다. 

> 타이머의 역할: 클록 -> 카운트 -> 시간 -> 이벤트 생성

### 물리적 구조
Timer의 기본 내부 구조는 다음과 같습니다. 
```
Clock Source
-> Pasacalar
-> Counter Resgister (CNT)
-> Compare Register (CMP)
-> Interrupt / Output Event / PWM Trigger
```

### 동작 흐름
#### 1. Clock 입력
Timer는 선택된 클록 소스로부터 주기적인 펄스를 입력받는다. 
* 예: System Clock이 클록 소스의 경우, 100 MHz의 펄스를 받는다.
* 그 외의 클록 소스로는 Peripheral Clock, Internal Oscillator, External Clock Pin 등이 될 수 있습니다.

#### 2. Pascalar의 분주
입력 클록은 너무 빠르기 때문에 Prescalar로 divide하여 사용합니다. 
* 예: 100 MHz / 100 = 1 MHz 이라면, Timer tick = 1 μs 가 됩니다.

#### 3. Counter 증가
Timer 하드웨어는 자동으로 Counter 레지스터를 증가시킵니다.
* 예: `CNT = 0 -> 1 -> 2 -> 3 -> ...`

#### 4. Compare 비교
CPU가 설정한 CMP와 CNT를 하드웨어가 자동으로 비교합니다. 
`CNT == CMP` 조건 만족 시, 인터럽트 발생/핀 토글/PWM 상태변경/DMA 트리거 등 하드웨어 이벤트가 발생합니다. 

## Timer 활용
### 1. 정확한 Delay 생성
#### 문제
단순 루프방식인 `for(i=0;i<100000;i++);`은 CPU 클록이 변경되거나, 컴파일러 최적화가 발생했을 때 시간이 변동될 수 있어, 정확한 시간 보장이 불가능합니다.
#### Timer 기반 방식으로 해결
다음과 같은 하드웨어 조건이라면,
* Timer Clock = 1 MHz
* 1 tick = 1 μs

10 ms 대기인 경우 `10,000 tick`을 거쳐야합니다. 
이를 코드로 구현하게 되면 다음과 같습니다. 
```C
uint32 start = TIMER->CNT;
while ((TIMER->CNT) - start < 10000);
```



### 2. 주기적 작업 실행
Timer는 인터럽트와 결합하면 정확학 주기 실행 엔진으로 사용될 수 있습니다.
예를 들어, `1 MHz Timer, CMP = 5000`인 조건에서 5ms마다 인터럽트를 발생시키고 싶다면 다음과 같이 인터럽트 코드를 생성할 수 있습니다.
```C
void TIMER_ISR(void)
{
  read_sensor();
}
```
* 활용: 센서 샘플링 주기 유지, 통신 타임아웃 체크, 소프트웨어 스케줄러 tick, Watchdog 보조 타이머 등

### 3. PWM 생성
#### PWM 개념
PWM(Pulse Width Modulation)은 
> 디지털 신호의 ON/OFF 비율을 조정하여(= duty cycle), 평균 전압/전류 효과를 만들어내는 방식입니다.
 
MCU의 핀 출력은 기본적으로 LOW (0V) / HIGH (3.3V 또는 5V) 두 상태만 가능합니다. 
PWM은 이 두상태를 고속으로 반복하여서 아날로그 신호처럼 동작하게 만들 수 있습니다.
```
HIGH ┌────┐        ┌────┐
     │    │        │    │
LOW  └────┴────────┴────┴─────
```
#### PWM 효과
* Duty 20%의 경우, `HIGH 짧게 가져감 -> 평균 전압이 낮아짐 -> LED가 어두워짐`
* Duty 80%의 경우, `HIGH 길게 가져감 -> 평균 전압이 높음 -> LED가 밝아짐`

#### Timer 기반 PWM 생성 구조
```
Clock Source
-> Prescalar
-> Timer Counter (CNT)
-> Compare Match (CMP)
-> Output Control Logic (Toggle / Set / Clear)
-> PWM Output pin
```
##### 1. TIMER Counter (CNT)
Timer는 클록 입력을 받아서 내부 카운터를 자동으로 증가시킵니다.
```
CNT = 0 -> 1 -> 2 -> 3 -> ... -> PERIOD (TOP)
```
이 카운터 값은 PWM 시간 축의 역할을 담당합니다.
##### 2. Period Register (TOP)
PWM 한 주기의 길이를 결정하는 기준 값입니다. 
`TOP = 1000`이라면 CNT가 0부터 1000까지 증가하고 다시 0으로 리셋되며, 이는 한 사이클을 의미합니다.
##### 3. Compare Register (CMP)
PWM duty cycle을 결정하는 기준 값입니다.: `CMP < TOP`
##### 4. Output Control Logic
하드웨어 비교 결과에 따라 출력 핀 상태를 자동으로 제어합니다.
일반적인 PWM 동작 규칙은 다음과 같습니다.
```
CNT < CMP  → Output HIGH
CNT ≥ CMP  → Output LOW
```
#### PWM 특징
1. CPU의 개입이 없이 Timer 하드웨어(CNT, CMP 등)이 자동으로 수행합니다.
1. 타이밍이 매우 정확합니다.
    * Timer 클록 기반으로 동작하기 때문에 인터럽트 지연에 영향도 없고, 일정한 파형을 유지합니다. 
1. 실시간으로 Duty를 변경할 수 있습니다.
    * CMP 값만 변경하면 PWM 출력도 자동으로 갱신되고, LED 밝기나 모터 속도도 실시간으로 부드럽게 제어가 가능합니다.  


# 내가 이해한 바
# 임베디드 개발자로서 주의할 점


