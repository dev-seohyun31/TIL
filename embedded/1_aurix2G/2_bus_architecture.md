# 실제 개념
## AURIX TC3xx의 버스 아키텍처

AURIX의 TC3xx은 여러 개의 TriCore CPU를 포함한 멀티코어 MCU이며, 각 코어는 32-bit CPU 아키텍처 기반으로 동작합니다.
여기서 32-bit CPU란, 기본 연산 단위, 레지스터 폭과 주소 계산 단위가 32비트라는 것을 의미하고, 이는 '버스 폭'과는 동일한 개념은 아닙니다.

실제 시스템 내부에서는 CPU 연산 폭보다 더 넓은 데이터 전송 폭을 사용하여 성능을 향상시키며, 
TC3xx 내부 인터커넥트(SRI)는 64-bit 이상의 넓은 datapath와 burst 전송 구조를 사용하여 Flash fetch, DMA 전송 등의 대역폭을 확보합니다. 
* CPU는 32-bit 단위의 화물을 처리하는 트럭
* SRI는 64-bit 이상 폭의 다차선 고속도로
* 여러 트럭이 동시에 빠르게 이동할 수 있도록 도로 폭을 넓인 구조입니다.

### SRI란
TC3xx의 버스 구조는 전통적인 MCU처럼 중앙 Bus Controller가 모든 중재를 수행하는 구조는 아닙니다. 
대신, SRI(Shared Resource Interconnect)라는 Crossbar Fabric 구조가 시스템 중앙에 존재하여, 주소 기반 스위칭과 병렬 경로 구성을 수행합니다.

중요한 특징은 중앙에서 중재를 하는 것이 아니라, Slave 인터페이스 단위로 중재가 분산되어 있다는 것입니다.

#### SRI의 Master / Slave 구조
SRI에서 데이터 전송 요청을 발생시키는 주체는 **Master**, 요청을 받아 실제 데이터를 제공하는 대상은 **Slave**라고 합니다.

**Master (요청 주체)**
* **TriCore CPU Cores**
* **DMA Controller**
* Safety Monitor
* Debug interface

**Slave (자원)**
* PFlash
* DFlash
* LMU RAM
* Peripheral Bridge
* Safety Registers

### FPI란
> FPI가 2개가 있는 것 같은데 (Peripheral, Processor) manual 보고 판단 필요

### 버스 아키텍처 전체 구조
```
CPU Core
   │
  FPI
   │
 ┌─┴─────────────────────────┐
 │           SRI             │  ← System Backbone
 └─┬─────────────┬───────────┘
   │             │
 Flash        LMU RAM
                 │
            Peripheral Bridge
                 │
            Peripheral Bus
```
---

### SRI 동작 흐름
예를 들어 CPU0가 LMU RAM을 읽는 경우, 흐름은 다음과 같습니다.
1. CPU0가 LOAD 명령 실행
1. 요청이 FPI를 통해 SRI로 전달
1. SRI Fabric이 주소 디코딩을 수행 - Slave가 LMU RAM인 것을 판단함
1. LMU RAM에 연결된 SCI에서 중재 수행
    * SCI(Slave Connection Interface)는 SRI Crossbar 내부에서 각 Slave 포트에 존재하는 중재 및 연결 인터페이스입니다. 역할은 아래와 같습니다.
    * 동일 Slave 내에서 여러 Master의 요청이 존재할 경우, 중재 역할 수행
    * 이 요청들을 High/Low 그룹으로 나누어서 우선순위를 분류
    * 각 그룹 내부에서 Round-Robin 방식으로 순차처리 수행
    * Slave 접근 경로 연결 제어
1. Crossbar Switch 내부에서 Master-Slave 경로가 하드웨어적으로 연결
1. LMU RAM에서 데이터가 반환되어, 동일 경로를 통해 CPU로 전달됩니다.

이 전체 과정은 소프트웨어의 개입은 없이 하드웨어의 파이프라인으로 동작합니다. 

### 주변장치 접근 경로
주변장치는 SRI에 직접 연결되지 않고, 반드시 Peripheral Bridge를 통해 접근합니다.
1. CPU0(주변장치의 owner)가 LOAD 명령 실행
1. 요청이 FPI를 통해 SRI로 전달
1. SRI가 주소 디코딩 수행
1. Peripheral Bridge 거침
    * SRI 고속 프로토콜과 주변장치 저속 버스 프로토콜로 상호 변환합니다.
    * 고속 Core 클록에서 주변장치 저속 클록으로 변환합니다.
    * 주소를 디코딩합니다.
    * 요청 순서대로 순차 처리합니다.
    * 이 단계에서 Peripheral Access Latency가 주로 발생됩니다.
1. Peripheral Module 레지스터로 접근

### Master 역할로서 DMA Controller
DMA는 CPU와 동일한 Master로서, SRI에 접근할 수 있습니다.
DMA는 주변장치의 이벤트를 트리거로 주변장치 레지스터와 RAM 간의 데이터 전송을 CPU 개입없이 수행합니다.

ADC 결과를 DMA로 저장하는 경로 예시를 들어보면 다음과 같습니다. 
1. EVADC 모듈로 입력 이벤트 발생 
1. 주변장치 버스에서 주변장치 Bridge로 이동
1. SRI Crossbar 통과
1. DMA 컨트롤러 동작
1. RAM에 데이터 저장

이를 통해, CPU 부하가 감소되고 대용량 데이터 처리 성능이 향상됩니다.





