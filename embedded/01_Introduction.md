# 01. 시작하기 전에 — 이 문서를 쓰게 된 배경과 iLLD 계층 구조

## 목차

- [1. 이 문서를 쓰게 된 배경](#1-이-문서를-쓰게-된-배경)
- [2. AURIX TC3xx 하드웨어 개요](#2-aurix-tc3xx-하드웨어-개요)
- [3. 레지스터 직접 접근 vs iLLD — 왜 iLLD를 쓰는가](#3-레지스터-직접-접근-vs-illd--왜-illd를-쓰는가)
- [4. iLLD 전체 계층 구조](#4-illd-전체-계층-구조)
- [5. Hld와 Lld — 언제 무엇을 쓰는가](#5-hld와-lld--언제-무엇을-쓰는가)
- [6. iLLD 폴더 구조](#6-illd-폴더-구조)
- [7. 챕터별 학습 로드맵](#7-챕터별-학습-로드맵)

---

## 1. 이 문서를 쓰게 된 배경

AURIX를 처음 접했을 때 GPIO 하나 켜는 것도 막막했다.

LED를 깜빡이는 예제 코드를 찾아 실행해봤지만,  
`IfxPort_setPinHigh()`가 내부적으로 뭘 하는지 전혀 감이 없었다.  
함수를 복붙하면 동작은 했지만, 안 되면 왜 안 되는지 알 수가 없었다.

그래서 방향을 바꿨다.  
iLLD API를 잠시 내려놓고, **레지스터를 직접 건드리는 것부터 시작했다.**

```c
/* PORT0 IOCR4 레지스터 PC5 필드 = push-pull output */
P0_IOCR4.B.PC5 = 0x10;

/* OMR 레지스터로 P0.5 토글 */
P0_OMR.B.PS5  = 1;  /* HIGH */
P0_OMR.B.PCL5 = 1;  /* LOW  */
```

이렇게 쓰고 나서야 비로소 알게 됐다.

- IOCR이 핀 방향을 결정하고
- OMR의 PS/PCL 비트가 분리되어 있어서 원자적으로 핀을 제어하고
- `IfxPort_setPinHigh()`는 결국 `OMR.PS` 비트에 1을 쓰는 것

이 경험을 바탕으로 이 문서를 쓰기 시작했다.  
**"레지스터로 이해하고, iLLD로 표현한다"** 는 흐름이 이 문서 전체의 방향이다.

레지스터를 알고 나서 iLLD를 보면,  
함수 이름만 봐도 어떤 레지스터를 건드리는지 연상이 된다.  
그 상태에서 iLLD를 쓰면, 편리함을 누리면서도 내부 동작을 이해한 채로 쓸 수 있다.

---

## 2. AURIX TC3xx 하드웨어 개요

### 코어 구조

AURIX TC3xx는 **멀티코어 32비트 마이크로컨트롤러**다.

```
AURIX TC375
├── CPU0  (TriCore, 최대 300MHz)
├── CPU1  (TriCore, 최대 300MHz)
├── CPU2  (TriCore, 최대 300MHz)
├── 공유 메모리 (DSPR, PSPR, LMU)
└── Peripheral Bus (SPI, CAN, ADC, GPIO ...)
```

각 코어는 독립적으로 동작하며, 코어간 통신은 공유 메모리나 인터럽트를 통한다.  
이 문서의 예제는 모두 **CPU0 단일 코어** 기준으로 작성됐다.

### 주요 Peripheral 목록

| Peripheral | 모듈명 | 용도 |
|---|---|---|
| GPIO | PORT | 디지털 입출력 |
| 타이머 | STM, GTM | 시간 측정, PWM 생성 |
| UART | ASCLIN | 시리얼 통신 |
| SPI | QSPI | 고속 직렬 통신 |
| I2C | I2C | 저속 직렬 통신 |
| CAN | MULTICAN | 차량 네트워크 통신 |
| ADC | EVADC | 아날로그 신호 측정 |
| 인터럽트 라우터 | SRC | 인터럽트 우선순위/코어 배정 |
| 워치독 타이머 | WDT | 시스템 감시 |

---

## 3. 레지스터 직접 접근 vs iLLD — 왜 iLLD를 쓰는가

### 레지스터 직접 접근

```c
/* ASCLIN0을 115200bps로 설정하는 레지스터 시퀀스 */
ASCLIN0_CLC.B.DISR       = 0u;
ASCLIN0_FRAMECON.B.MODE  = 1u;
ASCLIN0_FRAMECON.B.STOP  = 1u;
ASCLIN0_FRAMECON.B.PEN   = 0u;
ASCLIN0_DATCON.B.DATLEN  = 7u;
ASCLIN0_BITCON.B.OVERSAMPLING = 15u;
ASCLIN0_BRG.B.NUMERATOR  = 3u;
ASCLIN0_BRG.B.DENOMINATOR = 1u;
/* ... TX/RX 핀 라우팅, FIFO 설정 등 추가 */
```

**장점:** 하드웨어가 정확히 어떻게 동작하는지 보인다.  
**단점:** 설정 순서 실수, 분주비 계산 오류, 핀 라우팅 누락 등 실수할 곳이 많다.  
peripheral이 복잡해질수록 초기화 코드가 수십 줄로 늘어난다.

### iLLD 사용

```c
IfxAsclin_Asc_Config cfg;
IfxAsclin_Asc_initModuleConfig(&cfg, &MODULE_ASCLIN0);
cfg.baudrate.baudrate     = 115200u;
cfg.frame.dataLength      = IfxAsclin_DataLength_8;
cfg.frame.stopBit         = IfxAsclin_StopBit_1;
cfg.pins.tx               = &IfxAsclin0_TX_P14_0_OUT;
cfg.pins.rx               = &IfxAsclin0_RXA_P14_1_IN;
IfxAsclin_Asc_initModule(&drv, &cfg);
```

**장점:** 보드레이트 계산, 핀 라우팅, 초기화 순서를 iLLD가 대신 처리한다.  
**단점:** 내부 동작을 모르면 문제가 생겼을 때 원인 파악이 어렵다.

### 결론

레지스터를 먼저 이해한 다음 iLLD를 쓰면 두 가지를 모두 얻는다.

```
레지스터 이해  →  "IfxAsclin_Asc_initModule이 BRG 레지스터를 쓰는구나"
iLLD 사용     →  "보드레이트 계산은 맡기고 나는 로직에 집중"
```

이 문서의 모든 챕터는 **레지스터 방식 → iLLD 방식 비교** 구성을 따른다.

---

## 4. iLLD 전체 계층 구조

```
┌─────────────────────────────────────────┐
│           Your Application Code         │  ← main.c, task, 비즈니스 로직
└────────────────────┬────────────────────┘
                     │
┌────────────────────▼────────────────────┐
│      Hld  (High Level Driver)           │  ← IfxGtm_Tom_Pwm, IfxAsclin_Asc
│  여러 Lld 호출을 묶어 init/start 패턴 제공  │     IfxQspi_SpiMaster, IfxCan_Can
└────────────────────┬────────────────────┘
                     │
┌────────────────────▼────────────────────┐
│      Lld  (Low Level Driver)            │  ← IfxPort, IfxGtm, IfxAsclin
│  레지스터를 직접 읽고 쓰는 함수들           │     IfxSrc, IfxStm, IfxEvadc
└────────────────────┬────────────────────┘
                     │
┌────────────────────▼────────────────────┐
│  Platform / CPU Abstraction             │  ← IfxCpu, Ifx_Ssw
│  인터럽트 벡터 테이블, SFR 헤더, 부트코드   │     IfxScuWdt, IfxScuCcu
└────────────────────┬────────────────────┘
                     │
┌────────────────────▼────────────────────┐
│      Hardware Registers (SFR)           │  ← PORT_IOCR, GTM_TOM_SR0
│      실리콘 위의 레지스터 주소             │     ASCLIN_BRG, CAN_NBTP ...
└─────────────────────────────────────────┘
```

**핵심:** iLLD는 레지스터를 없애는 게 아니라 **감싸는** 것이다.  
Lld 함수를 열어보면 결국 레지스터에 값을 쓰는 코드가 나온다.

---

## 5. Hld와 Lld — 언제 무엇을 쓰는가

### Lld만 있는 peripheral

GPIO처럼 동작이 단순한 peripheral은 Hld 없이 Lld를 직접 쓴다.

| Peripheral | Lld | Hld |
|---|---|---|
| GPIO | `IfxPort` | 없음 |
| Interrupt | `IfxSrc` | 없음 |
| WDT | `IfxScuWdt` | 없음 |

### Hld가 있는 peripheral

초기화 순서가 복잡하거나, 여러 Lld 모듈을 조합해야 하는 peripheral에는 Hld가 있다.

| Peripheral | Lld | Hld |
|---|---|---|
| PWM | `IfxGtm` | `IfxGtm_Tom_Pwm` |
| UART | `IfxAsclin` | `IfxAsclin_Asc` |
| SPI | `IfxQspi` | `IfxQspi_SpiMaster` |
| ADC | `IfxEvadc` | `IfxEvadc_Adc` |
| CAN | `IfxCan` | `IfxCan_Can` |
| I2C | `IfxI2c` | `IfxI2c_I2c` |

Hld를 쓸 때도 Lld를 완전히 무시해도 되는 건 아니다.  
Hld가 지원하지 않는 세부 설정이 필요하거나,  
동작이 예상과 다를 때 Lld 코드를 직접 열어봐야 할 때가 생긴다.

### Hld의 공통 패턴

모든 Hld는 동일한 3단계 패턴을 따른다.

```c
/* 1. config 구조체를 기본값으로 초기화 */
IfxXxx_Yyy_initConfig(&config, &MODULE_XXX);

/* 2. config 구조체 멤버를 원하는 값으로 수정 */
config.baudrate = 115200u;
config.pins.tx  = &IfxXxx_TX_Pxx_x_OUT;

/* 3. 드라이버 초기화 실행 */
IfxXxx_Yyy_initModule(&driver, &config);
```

`initConfig`를 먼저 호출해 기본값을 채우고,  
필요한 항목만 수정하는 방식이라 설정 누락이 적다.

---

## 6. iLLD 폴더 구조

```
iLLD/
└── TC3xx/
    └── Tricore/
        ├── Port/
        │   └── Std/          ← IfxPort.h / IfxPort.c  (GPIO Lld)
        ├── Gtm/
        │   ├── Std/          ← IfxGtm.h  (GTM Lld)
        │   └── Tom/
        │       └── Pwm/      ← IfxGtm_Tom_Pwm.h  (PWM Hld)
        ├── Asclin/
        │   ├── Std/          ← IfxAsclin.h  (ASCLIN Lld)
        │   └── Asc/          ← IfxAsclin_Asc.h  (UART Hld)
        ├── Qspi/
        │   ├── Std/          ← IfxQspi.h  (QSPI Lld)
        │   └── SpiMaster/    ← IfxQspi_SpiMaster.h  (SPI Hld)
        ├── Evadc/
        │   ├── Std/          ← IfxEvadc.h  (EVADC Lld)
        │   └── Adc/          ← IfxEvadc_Adc.h  (ADC Hld)
        ├── Can/
        │   ├── Std/          ← IfxCan.h  (CAN Lld)
        │   └── Can/          ← IfxCan_Can.h  (CAN Hld)
        ├── I2c/
        │   ├── Std/          ← IfxI2c.h  (I2C Lld)
        │   └── I2c/          ← IfxI2c_I2c.h  (I2C Hld)
        └── Scu/
            └── Std/          ← IfxSrc.h, IfxScuWdt.h  (인터럽트, WDT Lld)
```

**패턴:**
- `Std/` 폴더 = Lld
- peripheral 이름 폴더 (예: `Asc/`, `SpiMaster/`, `Pwm/`) = Hld

새로운 peripheral을 쓰기 전에 이 구조를 먼저 확인하는 습관을 들이면  
어떤 헤더를 include해야 하는지 바로 보인다.

---

## 7. 챕터별 학습 로드맵

```
02. GPIO          → 레지스터 이해의 출발점. IfxPort (Lld만 사용)
        │
        ▼
03. Interrupt     → PWM/UART 전에 인터럽트 구조를 먼저. IfxSrc (Lld만 사용)
        │
        ▼
04. PWM (GTM)     → Hld 첫 등장. config→init→start 패턴 체감
        │
        ▼
05. UART (ASCLIN) → 통신 첫 번째. 인터럽트 + 링버퍼 연동
        │
        ▼
06. SPI (QSPI)    → UART 다음에 보면 차이가 명확
        │
        ▼
07. ADC (EVADC)   → 아날로그 입력. 모듈→그룹→채널 3단계 구조
        │
        ▼
08. CAN           → 차량 통신. 비트타이밍, 수신 필터, 루프백 테스트
        │
        ▼
09. I2C           → 2선 통신. SPI와의 차이, 오픈 드레인 특성
        │
        ▼
10. WDT           → 시스템 감시. 먼저 끄는 법부터 배우는 챕터
```

**각 챕터의 공통 구성:**

```
1. iLLD 계층 구조 (해당 peripheral 관점)
2. 동작 원리
3. 핵심 레지스터 요약
4. iLLD API 목록
5. 레지스터 방식 vs iLLD 비교
6~N. 예제 코드
정리 표
```

---

> 이 문서는 레지스터를 직접 다뤄보고 나서 iLLD로 넘어가려는 사람을 위해 썼다.  
> 완성된 참고서가 아니라, 배우면서 써 내려간 학습 노트다.  
> 잘못된 내용이나 더 나은 방법이 있다면 계속 수정해 나갈 것이다.
