# 실제 개념
## 전력 관리란
임베디드 시스템의 특성상 MCU는 항상 연산을 수행하는 상태에 있지 않습니다.
센서 입력을 대기하거나, 통신 이벤트를 기다리거나, 특정 주기까지 아무 동작도 하지 않는 시간이 전체 동작 시간의 대부분을 차지하는 경우가 많습니다.
그래서 MCU는 “항상 실행되는 컴퓨터”가 아니라, 이벤트가 발생할 때만 활성화되는 구조를 기본 동작 모델로 갖습니다.

이처럼 MCU가 필요한 순간에만 하드웨어 자원을 활성화하고, 나머지 시간에는 클록, 전원, 주변장치 일부 또는 전체를 비활성화하여 전력 소비를 최소화하는 제어 기법을 전력 관리(Power Management) 라고 합니다.

즉 전력 관리는 단순한 절전 기능이 아니라,
MCU가 언제 깨어 있고, 언제 잠들지, 어떤 하드웨어 블록을 유지하고 어떤 블록을 끌지를 결정하는 시스템 동작 전략입니다.

이 구조는 자연스럽게 이벤트 기반(Event-driven) 아키텍처로 이어집니다.
인터럽트, 타이머 만료, 외부 입력, 통신 수신과 같은 이벤트가 발생할 때만 CPU를 깨워 작업을 수행하고, 작업이 끝나면 다시 저전력 상태로 진입하는 방식이 기본 패턴이 됩니다.

## Power Management Unit (PMU)
PMU(Power Management Unit)은 MCU 내부 전체의 전력 흐름을 통제하는 시스템 주변장치입니다.
PMU는 단순히 레지스터 값을 바꾸는 수준이 아니라, 실제 전원 회로를 제어하여 전력을 통제합니다.

PMU는 다음과 같은 핵심 구성요소로 이루어져 있습니다.
### 1. Power Switch Control
Power Switch Control은 MCU 내부의 각 전원 도메인(Power Domain)에 전원을 공급할지 여부를 물리적으로 제어합니다.

#### Power Domain이란
MCU는 하나의 칩이지만 내부 전원 공급선이 여러 영역으로 분리되어 있습니다.
각 도메인은 독립적인 전원 스위치를 통해 선택적으로 전원 공급이 가능합니다.

대표적인 도메인 구성은 다음과 같습니다.
* **Core domain**: CPU, SRAM, Interrupt Controller, 레지스터
* **Peripheral domain**: UART, ADC, Timer 등 주변장치 블록
* **Backup domain**: RTC, Wake logic, Backup RAM 등 저전력 유지 영역

도메인 제어는 두 가지 수준이 존재합니다.
* **Clock OFF**: (Sleep 계열) 전원은 유지하되, 클록 공급만 차단합니다. 회로 상태를 유지하고 빠른 복귀가 가능합니다.
* **Power OFF**: (Standby/Hibernate 계열) 전원 자체를 차단합니다. 회로 상태가 소멸되고 재초기화가 필요합니다.
### 2. Voltage Regulator
MCU 내부 회로는 서로 다른 동작 전압을 필요로 합니다.
예를 들어 CPU Core는 1.xV, IO 영역은 3.3V와 같은 전압을 사용합니다.
PMU 내부에는 이러한 전압을 생성하고 분배하는 내부 전압 레귤레이터(LDO 또는 DC-DC) 가 포함되어 있습니다. 
또한, 단순 ON/OFF뿐 아니라 전압 레벨 자체를 조절하여 추가적인 전력을 절감합니다.

### 3. Wake Logic
Wake Logic은 Sleep, Standby 상태에서도 동작하는 초저전력 감시 회로입니다.
MCU 대부분이 정지되어 있는 상태에서도 외부 이벤트를 감시하고, 조건이 만족되면 PMU에 전원 복구 요청을 전달합니다.
주로 External Wake Pin, RTC Alarm, CAN Bus Activity, Watchdog 등을 감시합니다. 

## 모드 별 전력 상태 개요
MCU 데이터시트에는 Power Mode Table 형태로 각 모드의 전원 상태가 정의되어 있습니다.
### RUN 모드
정상 동작 모드로, 최대 성능, 최대 전력 소비 상태입니다.
* CPU: ON
* RAM: ON
* Flash: ON
* Peripheral: ON

### Sleep 모드
클록기반 저전력 모드로, 전원은 유지되며 빠른 복귀가 가능합니다.
* CPU: Clock OFF
* RAM: ON
* Peripheral: 선택적으로 Clock Gate

### Deep Sleep 모드
부분 전원을 차단하지만 메모리는 유지하는 상태입니다.
* CPU: Power OFF
* RAM: Retention
    * Retention 모드는 SRAM에 최소 전압만 공급하여 데이터가 유지될 수 있는 상태입니다.
* Peripheral: Power OFF

### Standby 모드
초저전력 대기상태로, Backup domain만 유지되며, 복귀 시 재초기화 과정이 필요합니다.
* CPU: OFF
* Main RAM: OFF
* Peripheral: OFF
* Backup Domain: ON

### Hibernate 모드
최소 전력 소비 상태로, 거의 전원 차단에 가까운 동작입니다.
* 대부분의 도메인: OFF
* Wake Logic: ON

```
전력 소비 많음
    │
    │  Normal Run
    │
    │  Sleep
    │
    │  Deep Sleep
    │
    │  Standby
    │
    │  Hibernate
    │
전력 소비 최소
```
# 임베디드 개발자가 신경써야 할 점
## sleep은 기본으로 사용한다. 
polling 기반으로 코드를 작성하면, 전력이 최악인 시스템이 나옵니다.
```C
while(1) {
   if(flag) do_something();
}
```
따라서, 인터럽트 + Sleep 구조를 사용하여 전력이 관리되는 시스템을 만들어야 합니다.
```C
sleep();
ISR() {
   do_something();
}
```
