# 실제 개념
## 인터럽트 시스템
인터럽트 시스템은 CPU가 순차 실행 중에도 외부/내부 이벤트에 즉시 반응하기 위해 만들어진 하드웨어 동작입니다.
CPU 내부 예외(Fault, Reset) 또는 주변 장치 이벤트(GPIO, Timer, UART 등)이 발생하면 현재 실행 흐름을 중단하고 사전에 등록된 ISR(Interrupt Service Routine)로 실행 흐름을 전환하여 이벤트를 처리합니다. 
이를 통해 MCU는 실시간성을 확보할 수 있습니다.



## Interrupt Controller의 구성 요소
CPU와 인터럽트 처리 역할을 분리하기 위해, MCU 내부에는 Interrupt Controller라는 전용 하드웨어 블록이 존재합니다. 
Interrupt Controller는 여러 주변장치에서 발생하는 인터럽트 요청들을 수집해서 정리하고, CPU로 전달할 인터럽트를 결정합니다.

### 1. Interrupt Request Line (IRQ Line)
IRQ Line은 주변장치에서 인터럽트 컨트롤러로 연결된 하드웨어 신호선입니다.

UART 수신 완료 / Timer Compare Match / GPIO Edge Detect 같은 이벤트가 발생하면, 해당 주변장치는 자신의 IRQ Line을 Assert(HIGH) 합니다. 

### 2. Pending Register
Pending Register는 인터럽트 발생 상태(flag)를 저장하는 IRQ별 레지스터이며, 
처리 대기 중인 인터럽트 요청 목록 (Queue) 역할을 수행합니다.

각 인터럽트마다 1비트씩 존재하여, 이벤트가 발생하면 해당 비트가 1로 저장되어 CPU가 처리할 때까지 유지합니다.


### 3. Priority Register
Priority Register는 각 인터럽트의 CPU 선점 우선순위를 저장하는 레지스터로, CPU가 가장 중요한 이벤트부터 처리하게 만듭니다. 

IRQ별로 숫자 값 형태로 저장되어 중재 로직에서 사용합니다.

### 4. Enable Register (Interrupt Mask Register)
Enable Register는 인터럽트 허용 여부를 결정하는 IRQ별 활성화 비트입니다. 
* Enable = 0 → Pending 발생해도 CPU로 전달이 되지 않습니다.
* Enable = 1 → 정상적으로 CPU로 전달됩니다.

## 선점
Preemption은 현재 ISR이 실행 중일 때, 더 높은 우선순위 인터럽트가 발생하면 실행 중인 ISR을 일시 중단하고 새로운 ISR로 즉시 전환하는 기능입니다.

Interrupt Controller는 현재 Active 인터럽트의 Priority와 새로 Pending 된 인터럽트의 Priority를 비교하여, 새 인터럽트의 Priority가 더 높으면 CPU에 즉시 전달합니다.

이 과정에서 CPU는
* 현재 ISR의 context를 스택에 저장하고,
* PC를 새 ISR 주소로 변경하며,
* 우선순위가 높은 ISR을 먼저 처리합니다.

이로 인해 Nested Interrupt(중첩 인터럽트) 구조가 형성될 수 있으며,
실시간성이 중요한 이벤트가 지연되지 않도록 보장할 수 있습니다..

## 동작 흐름
Timer Compare Match 인터럽트로 동작 예시를 보입니다.
1. 주변장치 이벤트 발생
    * Timer 내부에서 `Compare Match` 조건이 만족됩니다: `CNT == CMP`
1. 주변장치에서 IRQ Line로 신호 인가
1. 인터럽트 컨트롤러의 수신
    * 인터럽트 컨트롤러는 IRQ 입력을 감지한 후, 다음을 수행합니다.
    1. `Pending Register`에 해당 IRQ 비트를 1로 저장
    1. `Enable = 1`인 IRQ만 선별
    1. `(Pending == 1 && Enable == 1)` 조건을 만족하는 인터럽트 중 `Priority Register` 기준으로 중재 수행
1. CPU로 인터럽트 요청 전달
    * 인터럽트 번호 (Vector Index), IRQ 요청 신호를 함께 전달합니다. 
1. CPU의 인터럽트 수신
    * 인터럽트 진입되면 현재 실행 context 저장, 파이프라인 flush를 처리하여 인터럽트 실행할 준비를 합니다. 
1. CPU가 Vector Table 참조
    * CPU는 Flash에 있는 Interrupt Vector Table에서 인터럽트 번호에 해당하는 ISR 시작 주소를 Fetch
1. ISR 실행
    * CPU가 PC를 ISR 주소로 변경하여 Interrupt Service Routine(ISR)을 실행합니다. 
1. ISR 종료 및 복귀
    *  CPU는 저장된 context를 복구하고, 원래 실행 흐름으로 복귀합니다.



