# 실제 개념
## 임베디드 소프트웨어 계층 구조 (AURIX iLLD 기준)
임베디드 소프트웨어는 안정성, 유지보수성, 하드웨어 독립성을 확보하기 위해 명확한 계층 구조(Layered Architecture)를 따릅니다.
AURIX TC3xx의 iLLD 환경에서는 다음과 같이 4단계 구조로 구성됩니다.

### 1. Application Layer
Application Layer는 제품의 동작 로직과 기능 요구사항을 구현하는 영역입니다.
사용자가 작성하는 코드가 위치하며, 시스템 동작 흐름과 정책을 담당합니다.

`Cpu0_Main.c`, `Blinky_LED.c` 파일이 이 계층에 해당하고, `IfxPort_setPinHigh(PORT13, 0);`와 같은 코드를 사용합니다.

#### 특징
* 시스템 동작 시나리오를 정의하는 곳입니다.
* HAL API만 호출하여 하드웨어를 제어하고, 레지스터에 직접 접근하지 않습니다.

>"LED를 켜라"와 같은 시스템 의도만 표현하고, "어느 레지스터의 어느 비트를 조작한다"는 HAL 계층에 위임합니다.

### 2. HAL Layer
HAL(Hardware Abstration Layer)은 하드웨어 제어를 함수 인터페이스로 추상화한 계층입니다.
Application이 MCU 내부 구조를 몰라도 안전하게 하드웨어를 제어할 수 있도록 합니다.

AURIX 환경에서는 Infineon iLLD (Infineon Low Level Driver)가 HAL 역할을 수행합니다.
#### iLLD 디렉토리 구조 예시
```
Libraries/iLLD/TC3xx/
├─ Cpu/
├─ Port/
├─ Gtm/
├─ Asclin/
├─ Vadc/
├─ Stm/
├─ Src/
├─ Scu/
├─ Pms/
├─ _Impl/
├─ Std/
```
Application에서 `IfxPort_setPinHigh(PORT13, 0);`를 호출하면, HAL 내부는 `PORT13->OMR.B.PS0 = 1;`로 구현되어 있어 레지스터 접근 코드를 생성합니다. 



### 3. Register Layer
Register Layer는 MCU 내부 하드웨어 블록과 직접 연결된 메모리 매핑 영역입니다.

이 영역은 CPU의 LOAD/STORE 명령으로 접근되며, 실제 주변장치 제어 신호를 생성하는 하드웨어 제어 인터페이스입니다.
HAL 영역에서 `PORT13->OMR.B.PS0 = 1;`가 실행되면, Register 영역에서는 CPU가 다음 동작을 수행하게 됩니다.
```
STORE instruction
→ PORT13 OMR 주소에 값 write
```

#### 특징
* Memory Mapped I/O 구조로 되어있어, CPU가 LOAD/STORE 명령어로 직접 접근할 수 있습니다.
* 레지스터의 비트 조작이 하드웨어 회로 동작으로 바로 연결됩니다. 

이 계층은 반드시 다음 문서로 해석되며 비트 의미, 전기적 동작, 보호 조건 등이 정의됩니다. 
* User Manual
* Datasheet



### 4. H/W Layer (Silicon Layer)
Hardware Layer는 실제 MCU 실리콘 내부 회로 블록입니다.
즉, 레지스터 제어 결과가 물리적 전기 신호로 변환되는 영역입니다.
레지스터 계층에서 비트가 변화되면, 하드웨어 계층에서는 다음 동작이 수행됩니다.
```
Register Bit Change
→ GPIO Output Logic 활성
→ Output Driver 활성화 
→ 전압/전류 변화 발생
```

이 계층에는 다음 요소가 포함되어, 물리적인 회로를 수행합니다.
* GPIO Output Driver (PMOS / NMOS)
* Input Buffer (Schmitt Trigger)
* Timer Counter
* Bus Interconnect
* Clock Tree
* Power Domain


#### 특징
* 레지스터 제어 결과가 전기적 신호로 변환되어 실제 전압, 전류, 타이밍을 생성합니다. 
* 실리콘 구조에 의해 성능 한계가 결정됩니다.

## 전체 계층 흐름 정리
아래 흐름을 계층별로 다시 표시하면,
```
[ Application Layer ]
Application Code - iLLD를 사용할 뿐
      ↓

[ HAL Layer ]
HAL API 호출 - 레지스터 접근 코드가 실행
      ↓

[ Register Layer ]
Register Bit Write - 레지스터 값의 변화
      ↓

[ Hardware Layer ]
Internal Hardware Logic - 회로 구성
      ↓
Transistor Switching
      ↓
Pin Voltage Change - 전기/전압 생성
```

이 4가지 계층은 다음 문제를 해결합니다.
1. 안정성: 잘못된 레지스터로 접근을 방지하며, Atomic 접근 보호 및 초기화 순서를 관리합니다. 
1. 유지보수성: MCU 변경 시 HAL만 수정하면 되고, Application 코드는 재사용될 수 있습니다. 
1. 확장성: AUTOSAR MCAL 구조와 자연스럽게 연결되고 (MCAL은 추후 추가 예정), RTOS/멀티코어 구조로 확장이 용이합니다.

