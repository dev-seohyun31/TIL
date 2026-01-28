
# 실제 개념

## Timer란
Timer는 MCU 내부에서 시간 개념을 만들어주는 하드웨어 주변장치입니다. 
CPU는 클록 엣지, 명령 실행, 레지스터 값 변화만 인식할 뿐, "5ms 지났다", "1초 대기" 등 시간 개념을 직접 인식하지 못합니다. 
따라서 Timer는 클록 신호를 기반으로 카운트를 자동 증가시켜 시간 흐름을 숫자로 표현하는 역할을 수행합니다. 

> 타이머의 역할: 클록 -> 카운트 -> 시간 -> 이벤트 생성


Timer는 CPU가 직접 시간을 계산하는 방식이 아니라, 클록에 의해 구동되는 독립적인 하드웨어 블록으로 동작하며,
CPU는 Timer를 설정하고 시작시키는 역할만 수행합니다.

### Clock과의 관계
Timer는 내부 오실레이터에 직접 연결되지 않고, Clock Controller (특수 주변장치)를 통해 분배된 Peripheral Clock을 입력으로 받습니다.
```
Oscillator
-> PLL
-> Clock Controller
-> Peripheral Clock Line
-> Timer Core
```

Timer는 클록 경로와 버스 경로를 동시에 가지고 있어서, 
CPU로 시스템 버스를 통해 Timer 레지스터를 설정할 수 있고, 
Clock Line으로 실제 시간 동작을 수행할 수 있습니다.


### 물리적 구조
Timer는 `cpu enable` 시작과 함께 계속 실행되고 있으며, 기본 내부 구조는 다음과 같습니다. 
```
Peripheral Clock Input
-> Pasacalar
-> Counter Resgister (CNT)
-> Compare Register (CMP)
-> Event Logic
-> Interrupt / Output Toggle / PWM Trigger / DAM Trigger
```
* Presacalr: 입력 클록을 분주하여 CNT 증가 속도 조절합니다.
* Counter: 클록 기반 자동 증가 회로
* Compare: 이벤트 발생 기준 값. (CPU가 설정) 
* Event Logic: 하드웨어 이벤트 생성. (CPU가 설정)

### 동작 흐름
#### 1. Clock 입력
Timer는 Clock controller를 통해 Peripheral clock을 입력받습니다.
* 예: Peripheral Clock = 100 MHz
#### 2. Pascalar 분주
입력 클록은 너무 빠르기 때문에 Prescalar로 divide하여 사용합니다. 
* 예: 100 MHz / 100 = 1 MHz 이라면, Timer tick = 1 μs 가 됩니다.

#### 3. Counter 자동 증가
Timer 하드웨어는 클록 기반으로 Counter를 자동 증가시킵니다.
이 과정은 CPU 개입없이 하드웨어에서 독립적으로 수행합니다.
* 예: `CNT = 0 -> 1 -> 2 -> 3 -> ...`

#### 4. Compare 이벤트 발생
CPU가 설정한 CMP와 CNT가 일치하면 하드웨어 이벤트가 발생됩니다.
즉, `CNT == CMP` 조건 만족 시, 인터럽트 발생/핀 토글/PWM 상태변경/DMA 트리거 등 하드웨어 이벤트가 발생합니다. 

## Timer 활용
### 1. 정확한 Delay 생성
#### 문제점 (CPU 루프 방식)
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
Timer는 인터럽트와 결합하여 정확학 주기 실행 엔진으로 사용될 수 있습니다.
예를 들어, `1 MHz Timer, CMP = 5000`인 조건은 5ms마다 인터럽트를 발생시킬 수 있습니다.
또한, 다음과 같이 인터럽트 코드를 생성할 수 있습니다.
```C
void TIMER_ISR(void)
{
  read_sensor();
}
```
* 활용: 센서 샘플링 주기 유지, 통신 타임아웃 체크, 소프트웨어 스케줄러 tick, Watchdog 보조 타이머 등

### 3. PWM 생성
#### PWM 개념
PWM(Pulse Width Modulation)은 디지털 출력 신호의 HIGH 유지 시간 비율을 조정하여(= duty cycle), 평균 전력 전달량을 제어하는 방식입니다.
 
MCU의 핀 출력은 기본적으로 LOW (0V) / HIGH (3.3V 또는 5V) 두 상태만 출력할 수 있습니다. 
PWM은 이 두 상태를 고속으로 반복하여서 평균 전압 및 전류 효과를 만들어내어서 LED 밝기, 모터 속도와 같은 연속적인 제어를 할 수 있습니다.
```
HIGH ┌───────┐       ┌───────┐
     │       │       │       │
LOW  └───────┴───────┴───────┴───
        ←──── Period ────→
```
#### PWM 효과
* Duty 20%의 경우, `HIGH 짧게 가져감 -> 평균 전압이 낮아짐 -> LED가 어두워짐`
* Duty 80%의 경우, `HIGH 길게 가져감 -> 평균 전압이 높음 -> LED가 밝아짐`

#### Timer 기반 PWM 생성 구조
PWM은 독립된 주변장치가 아니라 Timer 주변장치 내부의 Output Compare 채널 기능으로 구현됩니다. 구조는 다음과 같습니다.
```
Clock Source
-> Prescalar
-> Timer Counter (CNT)
-> Compare Match (CMP)
-> Output Control Logic (Toggle / Set / Clear)
-> PWM Output pin
```
Timer의 Base Counter(CNT)는 모든 채널이 공유하고,
CMP는 채널별로 독립적으로 존재하여 각각의 PWM Duty를 생성합니다.

##### 1. TIMER Counter (CNT)
Timer는 클록 입력을 받아서 내부 카운터를 자동으로 증가시킵니다.
```
CNT = 0 -> 1 -> 2 -> 3 -> ... -> TOP -> 0 -> (반복)
```
이 CNT 값이 PWM의 공통 시간 기준(Time Base) 역할을 수행합니다.
CPU 개입 없이 클록 기반으로 동작합니다.

##### 2. Period Register (ARR / TOP)
PWM 한 주기의 길이를 결정하는 기준 값입니다. 
`TOP = 1000`이라면 CNT가 0부터 1000까지 증가하고 다시 0으로 리셋되며, 이는 PWM의 한 주기의 길이를 의미합니다.

PWM 주파수는 다음과 같이 정의됩니다.
```
PWM Frequency = Timer Clock / (Prescaler × TOP)
```
##### 3. Compare Register (CMP)
PWM의 Duty Cycle을 결정하는 채널별 기준값 레지스터입니다.
CMP 값은 CNT와 비교하여 출력 타이밍을 결정합니다. 
```
TOP = 1000, CMP = 3000
-> Duty = 30%
```
##### 4. Output Compare Logic
Timer 하드웨어는 매 tick마다 CNT와 CMP를 자동 비교하고,
그 결과에 따라 출력 핀 상태를 하드웨어적으로 제어합니다.
일반적인 PWM 동작 규칙은 다음과 같습니다.
```
CNT < CMP  → Output HIGH
CNT ≥ CMP  → Output LOW
```
이 과정은 CPU 개입 없이 Timer 하드웨어에서 자동 수행됩니다.

#### PWM 특징
1. CPU의 개입이 없이 Timer 하드웨어(CNT, CMP 등)이 자동으로 수행합니다.
1. 타이밍이 매우 정확합니다.
    * Timer 클록 기반으로 동작하기 때문에 인터럽트 지연에 영향도 없고, 일정한 파형을 유지합니다. 
1. 실시간으로 Duty를 변경할 수 있습니다.
    * CPU가 CMP 값만 변경하면 PWM 출력도 자동으로 갱신되고, LED 밝기나 모터 속도도 실시간으로 부드럽게 제어가 가능합니다.  


# 내가 이해한 바
# 임베디드 개발자로서 주의할 점
