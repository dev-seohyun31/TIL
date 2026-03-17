# 03. Interrupt / SRC — 인터럽트 라우팅 구조와 IfxSrc

## 목차

- [1. iLLD 계층 구조 — Interrupt 관점](#1-illd-계층-구조--interrupt-관점)
- [2. AURIX 인터럽트 동작 원리](#2-aurix-인터럽트-동작-원리)
- [3. 핵심 레지스터 요약](#3-핵심-레지스터-요약)
- [4. iLLD API — IfxSrc](#4-illd-api--ifxsrc)
- [5. 레지스터 방식 vs iLLD 비교](#5-레지스터-방식-vs-illd-비교)
  - [5-1. SRC 레지스터 설정](#5-1-src-레지스터-설정)
  - [5-2. 인터럽트 서비스 루틴 등록](#5-2-인터럽트-서비스-루틴-등록)
  - [5-3. 인터럽트 enable / disable](#5-3-인터럽트-enable--disable)
- [6. 예제 — STM 주기 인터럽트로 LED Blink](#6-예제--stm-주기-인터럽트로-led-blink)
- [7. 예제 — 외부 핀 인터럽트 (ERU)](#7-예제--외부-핀-인터럽트-eru)
- [8. 예제 — 인터럽트 우선순위 충돌 방지](#8-예제--인터럽트-우선순위-충돌-방지)
- [정리](#정리)

---

> GPIO 챕터까지는 폴링(polling)으로 핀 상태를 계속 읽었다.  
> 인터럽트를 쓰면 CPU가 다른 작업을 하다가 **이벤트 발생 시점에만** 호출된다.  
> AURIX의 인터럽트는 **SRC(Service Request Controller)**를 통해 라우팅된다.

---

## 1. iLLD 계층 구조 — Interrupt 관점

```
Your ISR (IFX_INTERRUPT 매크로로 등록)
    │
    ▼
[Lld]  IfxSrc.h / IfxSrc.c
       iLLD/TC3xx/Tricore/Src/Std/
    │
    ▼
[SFR]  SRC_xxx (Service Request Control Register)
       우선순위, 코어 라우팅, enable/disable
    │
    ▼
[HW]   Interrupt Router → CPU / DMA
```

GPIO와 마찬가지로 Hld 없이 **Lld(IfxSrc)를 직접 호출**한다.  
단, ISR 자체는 iLLD 매크로(`IFX_INTERRUPT`)로 등록한다.

---

## 2. AURIX 인터럽트 동작 원리

AURIX의 모든 인터럽트 소스(STM, ASCLIN, ERU 등)는 각자의 **SRC 레지스터**를 가진다.  
SRC 레지스터는 세 가지를 결정한다.

```
SRC 레지스터
├── SRPN  (Service Request Priority Number) : 0~255, 높을수록 우선
├── TOS   (Type Of Service)                 : 어떤 코어(CPU0~2)로 보낼지
└── SRE   (Service Request Enable)          : 인터럽트 활성화 여부
```

인터럽트 발생 흐름:

```
peripheral 이벤트 발생
    │
    ▼
해당 SRC 레지스터에 SRR(Service Request pending) 비트 set
    │
    ▼
SRC.SRE == 1 이면 Interrupt Router로 전달
    │
    ▼
SRPN 우선순위에 따라 TOS가 가리키는 CPU에 인터럽트 전달
    │
    ▼
CPU는 SRPN에 해당하는 벡터 테이블 엔트리 실행 → ISR 호출
```

> **벡터 번호 = SRPN**  
> `IFX_INTERRUPT(isr_func, core, priority)` 매크로가  
> 지정한 priority 번호의 벡터 슬롯에 ISR을 등록한다.  
> SRC.SRPN과 매크로의 priority가 **반드시 일치**해야 ISR이 호출된다.

---

## 3. 핵심 레지스터 요약

| 레지스터 | 역할 |
|---|---|
| `SRC_xxx` | 각 peripheral의 Service Request Control Register |
| `SRC_xxx.B.SRPN` | 인터럽트 우선순위 번호 (0~255) |
| `SRC_xxx.B.TOS` | 라우팅 대상 코어 (0=CPU0, 1=CPU1, 2=CPU2) |
| `SRC_xxx.B.SRE` | 인터럽트 enable (1) / disable (0) |
| `SRC_xxx.B.SRR` | 인터럽트 pending 여부 (read), 소프트웨어 트리거 가능 |
| `SRC_xxx.B.CLRR` | pending 비트 클리어 (ISR 안에서 명시적으로 처리할 때) |

> `SRC_STM0SR0`, `SRC_ASCLIN0TX` 처럼 peripheral마다 이름이 다르다.  
> iLLD에서는 `IfxStm_getSrcPointer()`, `IfxAsclin_getSrcPointerTx()` 등으로  
> 해당 SRC의 포인터를 가져올 수 있다.

---

## 4. iLLD API — IfxSrc

파일 위치: `iLLD/TC3xx/Tricore/Src/Std/IfxSrc.h`

| 함수 | 동작 |
|---|---|
| `IfxSrc_init(src, tos, priority)` | SRPN, TOS, SRE 한 번에 설정 |
| `IfxSrc_enable(src)` | SRE = 1 |
| `IfxSrc_disable(src)` | SRE = 0 |
| `IfxSrc_clearRequest(src)` | SRR 클리어 (CLRR) |
| `IfxSrc_isRequested(src)` | SRR 상태 읽기 |
| `IfxSrc_setRequest(src)` | 소프트웨어로 인터럽트 강제 발생 (SRR set) |

`src`는 `Ifx_SRC_SRCR*` 포인터로, peripheral마다 해당 SRC 주소를 가리킨다.  
`tos`는 `IfxSrc_Tos` enum: `IfxSrc_Tos_cpu0`, `IfxSrc_Tos_cpu1`, ...

---

## 5. 레지스터 방식 vs iLLD 비교

### 5-1. SRC 레지스터 설정

**레지스터 직접 접근**
```c
/* STM0 Compare0 인터럽트를 CPU0, 우선순위 10으로 설정 */
SRC_STM0SR0.B.SRPN = 10u;   /* 우선순위 */
SRC_STM0SR0.B.TOS  = 0u;    /* CPU0 */
SRC_STM0SR0.B.SRE  = 1u;    /* enable */
```

**iLLD**
```c
#include "IfxSrc.h"
#include "IfxStm_reg.h"

IfxSrc_init(&SRC_STM0SR0, IfxSrc_Tos_cpu0, 10u);
```

`IfxSrc_init` 내부에서 SRPN, TOS, SRE를 한 번에 쓴다.  
실수로 SRE를 빠뜨리는 것을 막아주는 것이 핵심 이점이다.

---

### 5-2. 인터럽트 서비스 루틴 등록

AURIX는 벡터 테이블 기반이다. ISR을 등록할 때 **우선순위 번호 = 벡터 번호**가 된다.

**레지스터 직접 접근** (벡터 테이블 직접 수정 — 비권장)
```c
/* 별도 어셈블리 또는 링커 스크립트로 벡터 슬롯 직접 연결 */
/* 실수 가능성이 높고 이식성이 낮아 일반적으로 사용하지 않는다 */
```

**iLLD**
```c
/* priority와 SRC.SRPN이 반드시 같아야 한다 */
IFX_INTERRUPT(STM0_ISR, 0, 10)   /* (함수명, 코어번호, 우선순위) */
{
    /* ISR 본문 */
    IfxSrc_clearRequest(&SRC_STM0SR0);
}
```

`IFX_INTERRUPT` 매크로는 컴파일러에게 해당 함수를   
지정 코어의 벡터 슬롯(우선순위 번호)에 배치하도록 지시한다.  
직접 벡터 테이블을 건드릴 필요가 없다.

---

### 5-3. 인터럽트 enable / disable

**레지스터 직접 접근**
```c
SRC_STM0SR0.B.SRE = 1u;   /* enable */
SRC_STM0SR0.B.SRE = 0u;   /* disable */
```

**iLLD**
```c
IfxSrc_enable(&SRC_STM0SR0);
IfxSrc_disable(&SRC_STM0SR0);
```

---

## 6. 예제 — STM 주기 인터럽트로 LED Blink

> STM(System Timer)의 Compare 인터럽트를 사용해 500ms마다 LED 토글.  
> LED = P0.5 (active low)

폴링 딜레이 루프 대신 타이머 인터럽트로 정확한 주기를 만드는 기본 패턴이다.

```c
#include "IfxPort.h"
#include "IfxSrc.h"
#include "IfxStm.h"

#define LED_PORT    (&MODULE_P00)
#define LED_PIN     5
#define ISR_PRIO    10u
#define ISR_CORE    0

/* STM 틱 → 시간 변환: 기본 클럭 100MHz 기준 500ms = 50,000,000 틱 */
#define STM_TICKS_500MS  50000000u

static void STM_reloadCompare(void)
{
    uint32 current = IfxStm_getLower(&MODULE_STM0);
    IfxStm_updateCompare(&MODULE_STM0, IfxStm_Comparator_0,
                         current + STM_TICKS_500MS);
}

/* ISR 등록: 함수명, 코어, 우선순위 */
IFX_INTERRUPT(STM0_ISR, ISR_CORE, ISR_PRIO)
{
    IfxSrc_clearRequest(&SRC_STM0SR0);   /* pending 클리어 */
    STM_reloadCompare();                 /* 다음 compare 값 갱신 */
    IfxPort_togglePin(LED_PORT, LED_PIN);
}

static void STM_init(void)
{
    /* Compare0 초기값 설정 */
    IfxStm_updateCompare(&MODULE_STM0, IfxStm_Comparator_0,
                         IfxStm_getLower(&MODULE_STM0) + STM_TICKS_500MS);

    /* SRC 설정: CPU0, 우선순위 10 */
    IfxSrc_init(&SRC_STM0SR0, IfxSrc_Tos_cpu0, ISR_PRIO);
    IfxSrc_enable(&SRC_STM0SR0);
}

static void GPIO_init(void)
{
    IfxPort_setPinModeOutput(LED_PORT, LED_PIN,
                             IfxPort_OutputMode_pushPull,
                             IfxPort_PadDriver_cmosAutomotiveSpeed1);
    IfxPort_setPinHigh(LED_PORT, LED_PIN); /* OFF */
}

int core0_main(void)
{
    GPIO_init();
    STM_init();

    /* 전역 인터럽트 enable */
    __enable();

    while (1)
    {
        /* 메인 루프는 다른 작업을 처리하거나 저전력 대기 */
    }
    return 0;
}
```

> **STM_TICKS_500MS 계산**  
> 실제 STM 클럭은 `IfxStm_getFrequency(&MODULE_STM0)`로 읽어야 정확하다.  
> 하드코딩 대신 `(uint32)(IfxStm_getFrequency(&MODULE_STM0) * 0.5f)` 형태를  
> 사용하면 클럭 설정이 바뀌어도 자동으로 맞춰진다.

---

## 7. 예제 — 외부 핀 인터럽트 (ERU)

> ERU(Event Request Unit)를 사용해 버튼 입력을 인터럽트로 처리.  
> 버튼 = P15.13 (falling edge 감지), LED = P0.5

폴링으로 버튼을 읽으면 CPU가 계속 IN 레지스터를 확인해야 한다.  
ERU를 쓰면 핀에 에지가 발생하는 순간에만 ISR이 호출된다.

```c
#include "IfxPort.h"
#include "IfxSrc.h"
#include "IfxScuEru.h"

#define LED_PORT    (&MODULE_P00)
#define LED_PIN     5
#define ISR_PRIO    20u
#define ISR_CORE    0

IFX_INTERRUPT(ERU_ISR, ISR_CORE, ISR_PRIO)
{
    IfxPort_togglePin(LED_PORT, LED_PIN);
    /* ERU는 자동으로 pending 클리어되므로 CLRR 불필요 */
}

static void ERU_init(void)
{
    /* P15.13 → ERU input channel 매핑 (보드 핀맵에 따라 채널 번호 달라짐) */
    IfxScuEru_InputChannel inputCh = IfxScuEru_InputChannel_5;
    IfxScuEru_InputNodePointer nodePtr = IfxScuEru_InputNodePointer_0;
    IfxScuEru_OutputChannel outputCh = IfxScuEru_OutputChannel_0;

    /* falling edge만 감지 */
    IfxScuEru_enableFallingEdgeDetection(inputCh);
    IfxScuEru_disableRisingEdgeDetection(inputCh);

    /* 이벤트 → 인터럽트 요청 연결 */
    IfxScuEru_connectTrigger(inputCh, nodePtr);
    IfxScuEru_enableTriggerPulse(inputCh);
    IfxScuEru_setInterruptGatingSel(outputCh, IfxScuEru_InterruptGatingSel_always);

    /* SRC 설정 */
    IfxSrc_init(&SRC_SCU_ERU0SR0, IfxSrc_Tos_cpu0, ISR_PRIO);
    IfxSrc_enable(&SRC_SCU_ERU0SR0);
}

static void GPIO_init(void)
{
    IfxPort_setPinModeOutput(LED_PORT, LED_PIN,
                             IfxPort_OutputMode_pushPull,
                             IfxPort_PadDriver_cmosAutomotiveSpeed1);
    IfxPort_setPinModeInput(&MODULE_P15, 13, IfxPort_InputMode_pullUp);
    IfxPort_setPinHigh(LED_PORT, LED_PIN);
}

int core0_main(void)
{
    GPIO_init();
    ERU_init();
    __enable();

    while (1) {}
    return 0;
}
```

> ERU 채널 번호와 핀 매핑은 TC3xx User Manual의  
> "Event Request Unit — Input Line Selection" 표를 반드시 확인해야 한다.  
> 핀마다 연결 가능한 ERU 채널이 고정되어 있다.

---

## 8. 예제 — 인터럽트 우선순위 충돌 방지

여러 peripheral의 인터럽트를 동시에 쓸 때 **SRPN이 겹치면 한쪽 ISR이 실행되지 않는다**.  
프로젝트 전체에서 우선순위를 한 곳에서 관리하는 방식이 권장된다.

```c
/* interrupt_config.h — 프로젝트 전체 우선순위 테이블 */
#ifndef INTERRUPT_CONFIG_H
#define INTERRUPT_CONFIG_H

/* 우선순위: 숫자가 클수록 높음 (1~255) */
#define ISR_PRIO_STM0       10u   /* 500ms LED 토글 */
#define ISR_PRIO_ERU_BTN    20u   /* 버튼 입력 */
#define ISR_PRIO_ASCLIN_RX  30u   /* UART 수신 (다음 챕터) */
#define ISR_PRIO_ASCLIN_TX  31u   /* UART 송신 (다음 챕터) */

/* 코어 배정 */
#define ISR_CORE_DEFAULT    0u

#endif /* INTERRUPT_CONFIG_H */
```

```c
/* 각 모듈에서 헤더를 include해서 사용 */
#include "interrupt_config.h"

IFX_INTERRUPT(STM0_ISR, ISR_CORE_DEFAULT, ISR_PRIO_STM0)
{
    IfxSrc_clearRequest(&SRC_STM0SR0);
    /* ... */
}
```

> 우선순위 번호가 겹치는 버그는 컴파일 오류가 나지 않아 찾기 어렵다.  
> 링커 단계에서 경고를 내주는 경우도 있지만 보장되지 않으므로,  
> `interrupt_config.h` 같은 중앙 테이블로 관리하는 습관이 중요하다.

---

## 정리

| 항목 | 레지스터 | iLLD |
|---|---|---|
| SRC 설정 (우선순위 + 코어 + enable) | `SRC_xxx.B.SRPN/TOS/SRE` | `IfxSrc_init()` |
| 인터럽트 enable | `SRC_xxx.B.SRE = 1` | `IfxSrc_enable()` |
| 인터럽트 disable | `SRC_xxx.B.SRE = 0` | `IfxSrc_disable()` |
| pending 클리어 | `SRC_xxx.B.CLRR = 1` | `IfxSrc_clearRequest()` |
| ISR 등록 | 벡터 테이블 직접 | `IFX_INTERRUPT(func, core, prio)` |
| 소프트웨어 트리거 | `SRC_xxx.B.SETR = 1` | `IfxSrc_setRequest()` |

SRC 레지스터의 SRPN과 `IFX_INTERRUPT` 매크로의 priority가 **반드시 일치**해야 한다.  
이 둘이 어긋나면 인터럽트가 발생해도 ISR이 호출되지 않는다.

---

*다음 챕터: PWM — GTM/TOM 구조와 IfxGtm_Tom_Pwm*
