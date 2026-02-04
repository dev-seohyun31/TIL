# 실제 개념
## AURIX TC3xx의 Interrupt & Trap

AURIX TC3xx에서 Interrupt와 Trap은 "정상 이벤트 처리"와 "비정상 예외 처리"를 하드웨어 수준에서 분리하여 설계된 구조입니다.
이런 구조는 실시간성과 기능 안전을 동시에 만족하기 위해 설계되었습니다.

Interrupt는 외부 또는 주변장치에서 발생하는 정상 이벤트 처리 경로이며,
Trap은 CPU 내부에서 발생하는 오류 및 예외 상황 처리 경로입니다.
이 두 시스템은 서로 독립된 벡터 테이블과 하드웨어 처리 경로를 사용합니다.

### 1. 인터럽트: 정상 이벤트 처리 시스템
인터럽트는 주변장치 또는 CPU 외부에서 발생하는 정상적인 이벤트 신호이다.
대표적으로 GPIO 버튼 입력, 타이머 완료 이벤트, ADC 변환 완료, CAN 메시지 수신, 통신 송신 완료 등의 이벤트가 있습니다.

이러한 이벤트들은 CPU가 계속 polling하지 않아도, 하드웨어가 CPU에게 즉시 처리를 요청하는 구조로 설계되어 있습니다.
>즉, "이벤트 발생 -> CPU에게 자동 통보 -> 즉시 ISR 실행"인 실시간 반응 구조를 만들고 있습니다.

#### 기본 처리 흐름
AURIX에서 인터럽트는 다음과 같은 단계로 처리됩니다.
1. 주변장치에서 이벤트 발생
1. 해당 주변장치의 SRC(Service Request Control) 레지스터에서 Pending Flag 설정함
1. SRC 레지스터로 Interrupt Router가 Priority 및 Target Core 확인
1. 지정된 CPU Core가 인터럽트 감지
1. CSA를 이용하여 현재 실행 컨텍스트 저장
1. ISR(Interrupt Service Routine) 실행
1. ISR 종료 후 원래 코드로 복귀

#### 1. SRC (Service Request Control)
AURIX는 기존 MCU에서 사용하는 단일 Interrupt Controller 대신
SRC 기반 분산 인터럽트 관리 구조를 사용합니다.

이 구조는 다음 목적을 위해 설계되었습니다.
* 멀티코어 환경에서 인터럽트 코어 분산 처리: 멀티코어 MCU이기 떄문에 인터럽트별로 처리할 CPU를 지정해야합니다. 
    * Timer 인터럽트는 CPU0가 전담
    * CAN 통신 인터럽트는 CPU1이 전담
* Safety 관련 인터럽트 분리 처리
    * Safety Fault는 Lockstep Core가 관리
* 하드웨어 수준의 실시간 우선순위 관리

각 주변장치 이벤트마다 전용 SRC 레지스터가 존재하며, SRC는 다음 정보를 관리합니다: `인터럽트 활성화 여부, 인터럽트 우선순위, 대상 CPU Core, 인터럽트 대기 상태`
```
SRC_xxx {
  SRPN   // Service Request Priority Number (우선순위)
  TOS    // Target Of Service (대상 CPU Core)
  SRE    // Service Request Enable (인터럽트 Enable)
  SRR    // Service Request Flag (Pending 상태)
}
```
즉, 개발자는 "이 인터럽트는 CPU1로 보내고, 우선순위는 5번"같은 정책을 하드웨어 레벨에서 직접 설정합니다.



#### 2. Priority & Preemption (완전 선점형 구조)
AURIX 인터럽트 시스템은 완전 선점형(Preemptive) 구조를 사용합니다.
즉, 현재 ISR이 실행 중이어도 더 높은 Priority를 가진 인터럽트가 발생하면 즉시 실행 흐름이 전환됩니다.

예를 들어, Priority 5 Timer ISR 실행 중에도 Priority 20 Brake Fault 인터럽트 발생하면 
1. 현재 ISR 중단
1. CSA에 컨텍스트 저장
1. Brake Fault ISR 즉시 실행
1. 종료 후 기존 ISR 복귀

>이 구조를 통해 시간 민감한 제어 이벤트가 항상 우선 처리되도록 보장합니다.

#### 3. Context Save: CSA 사용
AURIX는 일반 MCU처럼 Stack 기반 Context Save 방식이 아니라, CSA(Context Save Area) 전용 하드웨어 관리 영역을 사용합니다.
CSA는 RAM 내에 미리 구성된 고정 크기 블록 풀 구조이며, 인터럽트 진입 시 CPU 레지스터 컨텍스트가 자동으로 저장됩니다.

stack대신 CSA를 사용하면서 다음 장점을 가질 수 있습니다.
* 인터럽트 진입 시간이 항상 일정 (Deterministic Latency)
* 중첩 인터럽트 안전성 보장
* Stack Overflow 위험 감소
* RTOS Context Switching 성능 향상
* Safety 인증에 유리

>즉, CSA 구조로 AURIX는 실시간성과 안정성을 동시에 확보할 수 있습니다.

### 트랩: 비정상 예외 처리 시스템
Trap은 CPU가 스스로 감지한 내부 오류 및 예외 상황 처리 메커니즘입니다.
예를 들어, 잘못된 opcode로 인한 `Illegal Instruction`, 보호 영역 접근으로 인한 `Memory Protection Fault`, 존재하지 않는 주소 접근으로 인한 `Bus Error`, `Divide by 0`, `Privilege Violation` 등이 있습니다. 

트랩은 Trap + SMU + Watchdog 조합을 통해 잘못되면 조용히 멈추는 것이 아니라, 반드시 감지 -> 보고 -> 안전 상태로 전환하는 안전 설계를 구현합니다.

#### 트랩 처리 흐름
트랩 발생 시, CPU 내부 동작 흐름은 다음과 같습니다. 
1. CPU 실행 중 Fault 발생
1. CPU 하드웨어가 Trap 원인 감지
1. Trap Vector Table 참조
    * Interrupt Vector가 Peripheral 이벤트를 매핑하듯, Trap Vector는 CPU 내부 예외를 매핑
1. Trap Handler 실행
1. SMU 연동, Fault Report 또는 Reset 수행


#### Trap Class 구조
AURIX는 Trap을 단순한 단일 예외가 아니라, Trap Class 단위로 분류하여 관리합니다.

Trap Class는 오류 유형에 따라 다음과 같이 구분됩니다.
* Program Error
* Instruction Error
* Context Management Error
* Bus / Memory Error
* System / Assertion Error

이를 통해 오류 종류별로 대응 방식, 로그 정책, Reset 여부, Safety Reaction을 다르게 설계할 수 있습니다.
