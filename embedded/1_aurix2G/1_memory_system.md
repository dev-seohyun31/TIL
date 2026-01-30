# 실제 개념

## AURIX TC3xx의 Memory System 개요

일반 MCU는 메모리를 다음과 같이 기능별로 단순하게 분리합니다.
* Flash: 코드(`.text`), 변경하지 않는 상수(`.rodata`)
* RAM: 런타임 데이터 (stack, heap, 전역변수)
* Peripheral Register: 주변장치 제어 레지스터

이 구조는 싱글 코어 + 낮은 클록 + 단순 실시간 요구 환경에서 충분합니다. 

하지만 AURIX TC3xx는 멀티코어, 실시간 제어, 기능 안전, 고속 통신 등의 요구 조건을 만족해야 합니다. 
이로 인해 메모리 시스템이 논리적 분리를 넘어서 물리적으로까지 분리된 계층형 구조로 확장되었습니다. 
* Flash: 코드와 데이터의 논리적 분리에서, PFlash와 DFlash로 물리적 분리
* RAM: 단일 RAM에서 버스가 필요없는 CPU 전용 고속 메모리(Core Local RAM)과 멀티코어 공유 메모리(LMU RAM)으로 분리
* Peripheral Register: (큰 변화 없이 Memory Mapped I/O 기본 구조 유지)
* Cache: 고성능 CPU 대응을 위한 자동 성능 가속 계층으로 추가

## 1. Flash - PFlash와 DFlash의 물리 분리
일반 MCU에서는 Flash 내부를 링커가 `.text, .rodata, config data, EEPROM emulatioin` 영역으로 논리적으로 분리합니다.
하지만 실제 하드웨어 구조는 하나의 Flash array, 하나의 Flash controller, 동일한 erase block, 동일한 타이밍 특성으로 구성됩니다.
즉, 코드 실행과 데이터 쓰기가 같은 Flash 자원을 공유하여 둘의 성능 특성이 동일합니다.

AURIX는 Flash를 처음부터 물리 분리를 합니다. 
* PFlash (Program Flash): 코드 실행 전용으로, IF(Instruction Fetch)에 최적화된 구조를 갖고 있어 Prefetch buffer, burst fetch 경로를 제공합니다.
* DFlash (Data Flash): 데이터 저장 전용으로, EEPROM emulation 하드웨어를 지원하여 설정, 보정, 로그 저장을 할 수 있습니다. 

### 분리 효과
1. 안정적인 실시간 시스템 
    * 기존 MCU의 경우, 코드 실행 중에 Flash sector erase가 발생하면 Flash가 stall되어 CPU fetch가 지연됩니다. 
    * AURIX의 경우 코드 실행은 PFlash가, 데이터 쓰기는 DFlash가 담당하여 서로 실행 영향을 최소화 시킵니다.
1. 기능 안전 (ISO 26262) 만족
    * Code integrity와 Data integrity를 분리하여 ECC의 독립 관리가 가능해지고, 보호 영역 설정이 단순화됩니다.

## 2. RAM - Core Local RAM
### 개념
Core Local RAM은 각 CPU 코어의 바로 옆에 물리적으로 연결된 전용 메모리입니다.
* System Bus (SRI)를 거치지 않습니다.
* CPU 전용 포트로 직접 연결되어 빠릅니다.
* 고정 latency를 가지고 있어 결정적 지연을 제공합니다.
* 경합이 없습니다.

### 구성
각 CPU 코어는 다음 두 종류의 Local RAM을 가집니다.
* DSPR (Data ScratchPad RAM): 데이터 저장소입니다.
* PSPR (Program ScratchPad RAM): 코드 실행용 저장소입니다.
    * Flash 대신 시간적으로 민감한 코드들 (예: ISR, 제어 루프 등)을 RAM에서 실행하기 위한 구조입니다.

개발자는 `제어 루프 변수, 인터럽트 핫패스 데이터, 타이밍 민감 알고리즘, 실시간 task stack` 등도 명시적으로 local RAM에 배치할 수 있습니다.


## 3. RAM - LMU RAM (Shared Memory)
### 개념
LMU RAM은 모든 CPU 코어와 DMA, 주변장치가 함께 사용하는 공유 메모리입니다. 
기본 MCU에서의 RAM으로 고려하면 되며, 다음 목적에서 해당 영역을 사용할 수 있습니다.
* 멀티코어 환경에서 CPU간 데이터 교환시 사용 가능합니다.
* DMA buffer를 저장할 수 있습니다.
* 통신 Stack buffer를 저장할 수 있습니다.
* 대용량 데이터를 저장할 수 있습니다.


### 접근 구조
Local RAM과 달리 버스로 연결되어 있어, 버스 중재가 필요하고 접근 순서에 따라 지연이 변동 가능합니다.
```
CPU / DMA / Peripheral
        ↓
     System Bus (SRI)
        ↓
       LMU RAM
```

### 로컬/공유 메모리 사용 기준
따라서, Local RAM과 LMU RAM의 사용 기준은 다음과 같습니다.
* Local RAM 사용 대상: 결정적 지연이 필요한 시스템
    * ISR
    * 브레이크/모터 제어 루프
    * Hard 실시간 영역
* LMU RAM 사용 대상
    * CAN/Ethernet DMA buffer
    * Core간 공유 데이터
    * 로그/버퍼/테이블
    * RTOS IPC 영역


상황에 따라 지연이 생길 수 있어, 실시간 시스템에서 조심히 사용해야 합니다.
예를 들어, 브레이크 ECU나 ISR의 경우 지연시간이 허용되지 않아 Local RAM을 사용해야 하지만, 통신 버퍼같은 DMA Buffer는 LMU를 사용해도 괜찮습니다. 

중재 로직은 버스 중재 로직을 차용합니다???? LMU의 중재 규칙이 또 있나? 


## 4. Cach - CPU 내부 가속 계층
캐시는 기본 MCU에서는 거의 없고, Aurix같은 고급 MCU부터 본격적으로 등장하는 구조입니다. 
기존 MCU에서는 CPU 속도와 Flash 속도가 비슷하여 병목이 거의 없었지만, 현대 MCU는 CPU가 300 MHz 이상 / Flash는 수십 MHz / 공유 버스 구조를 가져 CPU 대기 시간이 급증하였습니다.
따라서, Flash와 RAM의 접근 지연을 숨기기 위해 CPU 내부에 캐시 계층이 도입되었습니다.

### Cache 구조
AURIX는 다음 두가지 캐시를 다음 목적으로 사용합니다.
* PCache (Program Cache): IF 가속
* DCache (Data Cache): RAM LOAD/STORE 가속

최초 접근 시, CPU는 Flash와 LMU RAM에서 Cache line 단위로 복사합니다. 
이후 접근부터는 CPU가 Cache hit가 발생하여 즉시 응답할 수 있습니다. 

### DMA와 Cache의 문제
안타깝게도 DMA는 캐시를 인식하지 못해 DMA가 LMU RAM에 쓰기 작업 시, 캐시와 LMU RAM의 불일치 현상이 발생할 수 있습니다.

* DMA buffer를 non-cacheable LMU 영역을 사용하도록 강제하거나,
* Cache invalidate 또는 flush를 수행하거나,
* 실시간 영역은 Local RAM을 사용하도록 명시하는 방법으로 해결할 수 있습니다.

