# 실제 개념

## Timer란
timer란, 타이밍을 담당한는 주변장치로서, 클록(이 때, 상황에 따라 시스템 클록이 될 수도, 주변장치나 외부 핀의 클록이 될 수도 있다.)
을 인풋으로 받아 레지스터에 CNT를 저장하는 장치입니다. 
CPU는 시간을 모르고 있어서 몇 ms 지났다는 개념이 없어, 클록을 통해 카운트를 하여 레지스터에 전달합니다. 

### 물리적 구성
```
Clock source -> Prescalar -> Counter Register (CNT) -> Compare Register (CMP) -> Interrupt / Output Event
```
위와 같이 구성되어 있고, 동작 흐름은 다음과 같습니다. 
1. Clock source로부터 클록을 입력받는다.
1. Prescalar를 통해 클록(100MHz)을 적당값으로 줄인다. : 예) 100으로 나누면 1MHz가 됨
1. Counter 레지스터에 자동 증가하여 저장합니다. 예) 0 -> 1-> 2... 따라서, 1MHz 클록일 경우 1count = 1마이크로초가 됩니다. 
1. 원하는 값 (CMP 레지스터)와 CNT 레지스터를 비교하여, 같으면 이벤트(예: 인터럽트, 핀 토글, PWM 변경 등) 발생시킵니다. 

## 활용 
### 1. 정확한 delay 생성
* 예: 센서 전원 켠 뒤에 10ms 대기
* for-loop로 `for (i = 0; i <100000; i++)`이라면 클록이 바뀔 시 지연 시간이 달라집니다.

* Timer clock = 1 MHz
* CMP = 10000 이라면

다음과 같이 초를 정확히 셀 수 있음
```C
start = TIMER->CNT;
while (TIMER->CNT - start < 10000);
```

### 2. 주기적인 작업
* 예: LED 500ms마다 토글, 센서를 5ms마다 읽기

* CMP = 5000
* Interrupt enable
```C
void TIMER_ISR()
{
   read_sensor();
}
```
위와 같이 코딩하면, Timer math 시, 인터럽트가 발생하여 read_sensor() 함수가 실행됩니다.


### 3. PWM 생성

PWM은 디지털 신호로 아날로그 효과를 내기 위하여 디지털 신호 (0/1)을 매우 빠르게 반복하여, 평균 전압/전류를 제어하는 기법입니다.
아날로그 효과를통해 LED 밝기, 모터 속도, 팬 회전, 히터 출력 등 연속적인 제어가 가능해집니다. 
이는 GPIO의 출력이 0V 또는 3.3V 밖에 되지 않는 것을 보완하여, GPIO와 함께 쓰이기도 합니다. 




