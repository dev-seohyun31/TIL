# 11. TIM — GTM/TIM 구조와 IfxGtm_Tim

## 목차

- [1. iLLD 계층 구조 — TIM 관점](#1-illd-계층-구조--tim-관점)
- [2. GTM 타이머 모듈 전체 구조](#2-gtm-타이머-모듈-전체-구조)
- [3. TIM 동작 원리](#3-tim-동작-원리)
- [4. 핵심 레지스터 요약](#4-핵심-레지스터-요약)
- [5. iLLD API — IfxGtm_Tim](#5-illd-api--ifxgtm_tim)
- [6. STM vs TIM — 언제 무엇을 쓰는가](#6-stm-vs-tim--언제-무엇을-쓰는가)
- [7. 레지스터 방식 vs iLLD 비교](#7-레지스터-방식-vs-illd-비교)
  - [7-1. TIM 채널 입력 캡처 설정](#7-1-tim-채널-입력-캡처-설정)
  - [7-2. 캡처 값 읽기](#7-2-캡처-값-읽기)
- [8. 예제 — 입력 신호 주파수 측정](#8-예제--입력-신호-주파수-측정)
- [9. 예제 — PWM 입력 Duty Cycle 측정](#9-예제--pwm-입력-duty-cycle-측정)
- [10. 예제 — STM으로 주기 타이머 (ms 단위)](#10-예제--stm으로-주기-타이머-ms-단위)
- [11. 예제 — STM으로 경과 시간 측정](#11-예제--stm으로-경과-시간-측정)
- [정리](#정리)

---

> AURIX에는 용도가 다른 두 종류의 타이머가 있다.  
> **STM(System Timer)** 은 단순 카운터로 시간 측정과 주기 인터럽트에 쓰이고,  
> **GTM TIM(Timer Input Module)** 은 외부 핀 신호를 캡처해 주파수/duty를 측정한다.  
> 이 챕터에서는 두 가지를 모두 다룬다.

---

## 1. iLLD 계층 구조 — TIM 관점

```
Your Code
    │
    ▼
[Hld]  IfxGtm_Tim_In.h / IfxGtm_Tim_In.c
       iLLD/TC3xx/Tricore/Gtm/Tim/In/
       (입력 캡처 init, 결과 읽기 래핑)
    │
    ▼
[Lld]  IfxGtm_Tim.h / IfxGtm.h
       iLLD/TC3xx/Tricore/Gtm/Std/
       (TIM 채널 설정, 캡처 레지스터 접근)
    │
    ▼
[SFR]  GTM_TIMx_CHy_GPR0/GPR1, CNT, CTRL ...
    │
    ▼
[HW]   외부 입력 핀 (TIN)
```

STM은 Hld 없이 `IfxStm.h` (Lld)를 직접 쓴다.

---

## 2. GTM 타이머 모듈 전체 구조

GTM은 출력(TOM)과 입력(TIM) 두 방향 모두를 담당한다.

```
GTM (Generic Timer Module)
├── CMU  (Clock Management Unit) — 클럭 분주 공급
│
├── TOM  (Timer Output Module)   — PWM 출력 (04 챕터)
│   ├── TOM0 CH0~CH15
│   └── TOM1 CH0~CH15
│
└── TIM  (Timer Input Module)    — 입력 캡처 (이 챕터)
    ├── TIM0 CH0~CH7
    └── TIM1 CH0~CH7
```

TOM이 PWM을 **출력**하는 모듈이라면,  
TIM은 외부 핀 신호를 **캡처**해서 주파수, 주기, Duty Cycle을 측정하는 모듈이다.

---

## 3. TIM 동작 원리

```
외부 핀(TIN) 입력
    │
    ▼
에지 감지 (rising / falling / both)
    │
    ├── GPR0 : 에지 발생 시 CNT 값 캡처 (주기 측정용)
    └── GPR1 : 에지 발생 시 CNT 값 캡처 (Duty 측정용)
```

**주파수/주기 측정 (TPWM 모드):**
```
rising edge      rising edge
    │                │
    │←── GPR0 ──────→│   : 한 주기의 카운트 수
    │←─ GPR1 ──→│        : HIGH 구간의 카운트 수

주기   = GPR0 / GTM_CLK
주파수 = GTM_CLK / GPR0
Duty   = GPR1 / GPR0 × 100 (%)
```

---

## 4. 핵심 레지스터 요약

**GTM TIM 레지스터:**

| 레지스터 | 역할 |
|---|---|
| `GTM_TIMx_CHy_CTRL` | 채널 동작 모드, 에지 선택, 클럭 소스 |
| `GTM_TIMx_CHy_GPR0` | 캡처값 0 (주기 측정) |
| `GTM_TIMx_CHy_GPR1` | 캡처값 1 (HIGH 구간 측정) |
| `GTM_TIMx_CHy_CNT` | 현재 카운터 값 |
| `GTM_TIMx_CHy_EIRQ_EN` | 인터럽트 enable |
| `GTM_TIMx_CHy_IRQ_NOTIFY` | 인터럽트 플래그 클리어 |

**STM 레지스터:**

| 레지스터 | 역할 |
|---|---|
| `STMx_TIM0` | 64비트 카운터 하위 32비트 |
| `STMx_TIM1` | 64비트 카운터 상위 32비트 |
| `STMx_CMP0` | Compare 0 레지스터 (인터럽트 발생 기준) |
| `STMx_CMCON` | Compare 비교 비트 범위 설정 |
| `STMx_ICR` | 인터럽트 제어 |

---

## 5. iLLD API — IfxGtm_Tim

**TIM (입력 캡처):**

파일 위치: `iLLD/TC3xx/Tricore/Gtm/Tim/In/IfxGtm_Tim_In.h`

| 함수 | 동작 |
|---|---|
| `IfxGtm_Tim_In_initConfig(config, gtm)` | config 기본값 초기화 |
| `IfxGtm_Tim_In_init(driver, config)` | TIM 채널 초기화 |
| `IfxGtm_Tim_In_update(driver)` | 캡처 결과 갱신 |
| `driver->periodUs` | 측정된 주기 (마이크로초) |
| `driver->dutyPercent` | 측정된 Duty Cycle (%) |

**STM (시스템 타이머):**

파일 위치: `iLLD/TC3xx/Tricore/Stm/Std/IfxStm.h`

| 함수 | 동작 |
|---|---|
| `IfxStm_getLower(stm)` | 카운터 하위 32비트 읽기 |
| `IfxStm_get(stm)` | 64비트 카운터 전체 읽기 |
| `IfxStm_updateCompare(stm, cmp, val)` | Compare 값 갱신 |
| `IfxStm_getFrequency(stm)` | STM 클럭 주파수 읽기 |
| `IfxStm_getTicksFromMilliseconds(stm, ms)` | ms → 틱 변환 |

---

## 6. STM vs TIM — 언제 무엇을 쓰는가

| 항목 | STM | GTM TIM |
|---|---|---|
| 방향 | 내부 카운터 | 외부 핀 입력 캡처 |
| 주 용도 | 주기 인터럽트, 경과 시간 측정 | 외부 신호 주파수/Duty 측정 |
| 핀 필요 | 없음 | 입력 핀 필요 |
| 정밀도 | 클럭 주기 단위 (ns) | GTM 클럭 주기 단위 |
| iLLD | `IfxStm.h` (Lld 직접) | `IfxGtm_Tim_In` (Hld) |

---

## 7. 레지스터 방식 vs iLLD 비교

### 7-1. TIM 채널 입력 캡처 설정

**레지스터 직접 접근**
```c
/* GTM CMU 클럭 enable */
GTM_CMU_CLK_EN.B.EN_CLK0 = 0x2u;

/* TIM0 CH0: TPWM 모드, 상승 에지 캡처, CLK0 */
GTM_TIM0_CH0_CTRL.B.TIM_MODE = 0x4u;  /* TPWM */
GTM_TIM0_CH0_CTRL.B.CLK_SEL  = 0x0u;  /* CLK0 */
GTM_TIM0_CH0_CTRL.B.ISL      = 0x0u;  /* 상승 에지 */

/* 핀 입력 라우팅 */
/* ... IOCR 설정 ... */

/* 채널 enable */
GTM_TIM0_CH0_CTRL.B.TIM_EN = 1u;
```

**iLLD**
```c
#include "IfxGtm_Tim_In.h"

IfxGtm_Tim_In_Config cfg;
IfxGtm_Tim_In_Driver drv;

IfxGtm_Tim_In_initConfig(&cfg, &MODULE_GTM);
cfg.filter.inputPin    = &IfxGtm_TIM0_0_TIN0_P02_0_IN;
cfg.tim                = IfxGtm_Tim_0;
cfg.timChannel         = IfxGtm_Tim_Ch_0;
cfg.clock              = IfxGtm_Tim_Ch_ClkSrc_cmuFxclk0;

IfxGtm_Tim_In_init(&drv, &cfg);
```

---

### 7-2. 캡처 값 읽기

**레지스터 직접 접근**
```c
/* GPR0: 주기 카운트, GPR1: HIGH 구간 카운트 */
uint32 period_cnt = GTM_TIM0_CH0_GPR0.B.GPR0;
uint32 high_cnt   = GTM_TIM0_CH0_GPR1.B.GPR1;

float32 gtm_clk   = 100000000.0f;  /* 100MHz */
float32 period_us = (float32)period_cnt / gtm_clk * 1000000.0f;
float32 duty_pct  = (float32)high_cnt / (float32)period_cnt * 100.0f;
```

**iLLD**
```c
IfxGtm_Tim_In_update(&drv);
float32 period_us = drv.periodUs;
float32 duty_pct  = drv.dutyPercent;
```

---

## 8. 예제 — 입력 신호 주파수 측정

> TIM0 CH0, 입력 핀 P02.0. 외부 신호의 주파수를 측정해 UART로 출력.

```c
#include "IfxGtm_Tim_In.h"
#include "IfxAsclin_Asc.h"
#include <stdio.h>

static IfxGtm_Tim_In_Driver g_timDrv;
static IfxAsclin_Asc_Driver g_asc;

static void TIM_init(void)
{
    IfxGtm_Tim_In_Config cfg;
    IfxGtm_Tim_In_initConfig(&cfg, &MODULE_GTM);
    cfg.filter.inputPin = &IfxGtm_TIM0_0_TIN0_P02_0_IN;
    cfg.tim             = IfxGtm_Tim_0;
    cfg.timChannel      = IfxGtm_Tim_Ch_0;
    cfg.clock           = IfxGtm_Tim_Ch_ClkSrc_cmuFxclk0;
    IfxGtm_Tim_In_init(&g_timDrv, &cfg);
}

static void UART_init(void)
{
    IfxAsclin_Asc_Config cfg;
    IfxAsclin_Asc_initModuleConfig(&cfg, &MODULE_ASCLIN0);
    cfg.baudrate.baudrate     = 115200u;
    cfg.baudrate.oversampling = IfxAsclin_OversamplingFactor_16;
    cfg.frame.dataLength      = IfxAsclin_DataLength_8;
    cfg.frame.stopBit         = IfxAsclin_StopBit_1;
    cfg.frame.parityBit       = FALSE;
    cfg.pins.tx               = &IfxAsclin0_TX_P14_0_OUT;
    cfg.pins.rx               = &IfxAsclin0_RXA_P14_1_IN;
    IfxAsclin_Asc_initModule(&g_asc, &cfg);
}

int _write(int fd, const char *buf, int count)
{
    (void)fd;
    Ifx_SizeT len = (Ifx_SizeT)count;
    IfxAsclin_Asc_write(&g_asc, (const uint8 *)buf, &len, TIME_INFINITE);
    return count;
}

int core0_main(void)
{
    TIM_init();
    UART_init();

    while (1)
    {
        IfxGtm_Tim_In_update(&g_timDrv);

        float32 freq_hz = (g_timDrv.periodUs > 0.0f)
                        ? (1000000.0f / g_timDrv.periodUs)
                        : 0.0f;

        printf("Freq: %.1f Hz  Duty: %.1f %%\r\n",
               (double)freq_hz, (double)g_timDrv.dutyPercent);

        volatile uint32 i; for (i = 0; i < 2000000u; i++);
    }
    return 0;
}
```

---

## 9. 예제 — PWM 입력 Duty Cycle 측정

> 04 챕터의 PWM 출력(P02.0)을 TIM 입력(P02.1)으로 루프백해서 측정.  
> 실제 출력 Duty와 측정값을 비교해 캡처 정확도를 확인한다.

```c
#include "IfxGtm_Tom_Pwm.h"
#include "IfxGtm_Tim_In.h"

static IfxGtm_Tom_Pwm_Driver g_pwmDrv;
static IfxGtm_Tim_In_Driver  g_timDrv;

static void PWM_init(void)
{
    IfxGtm_Tom_Pwm_Config cfg;
    IfxGtm_Tom_Pwm_initConfig(&cfg, &MODULE_GTM);
    cfg.tom           = IfxGtm_Tom_0;
    cfg.tomChannel    = IfxGtm_Tom_Ch_0;
    cfg.clock         = IfxGtm_Tom_Ch_ClkSrc_cmuFxclk0;
    cfg.period        = 1000u;
    cfg.dutyCycle     = 300u;   /* 30% */
    cfg.pin.outputPin = &IfxGtm_TOM0_0_TOUT0_P02_0_OUT;
    IfxGtm_Tom_Pwm_init(&g_pwmDrv, &cfg);
    IfxGtm_Tom_Pwm_start(&g_pwmDrv, TRUE);
}

static void TIM_init(void)
{
    IfxGtm_Tim_In_Config cfg;
    IfxGtm_Tim_In_initConfig(&cfg, &MODULE_GTM);
    cfg.filter.inputPin = &IfxGtm_TIM0_0_TIN1_P02_1_IN; /* 별도 핀으로 수신 */
    cfg.tim             = IfxGtm_Tim_0;
    cfg.timChannel      = IfxGtm_Tim_Ch_1;
    cfg.clock           = IfxGtm_Tim_Ch_ClkSrc_cmuFxclk0;
    IfxGtm_Tim_In_init(&g_timDrv, &cfg);
}

int core0_main(void)
{
    PWM_init();
    TIM_init();

    while (1)
    {
        IfxGtm_Tim_In_update(&g_timDrv);
        /* g_timDrv.dutyPercent ≈ 30.0% 이면 정상 */
        volatile uint32 i; for (i = 0; i < 500000u; i++);
    }
    return 0;
}
```

> P02.0(출력)과 P02.1(입력)을 점퍼선으로 연결하면 루프백 테스트가 된다.

---

## 10. 예제 — STM으로 주기 타이머 (ms 단위)

> STM0 Compare 인터럽트를 이용해 1ms 주기 태스크를 구현한다.  
> 실제 RTOS 없이 태스크 스케줄러를 흉내 낼 때 기본이 되는 패턴이다.

```c
#include "IfxStm.h"
#include "IfxSrc.h"

#define ISR_PRIO_STM  10u

static volatile uint32 g_tick_1ms = 0u;   /* 1ms마다 증가 */

static uint32 STM_getTicksPerMs(void)
{
    /* 실제 STM 클럭 기반으로 1ms에 해당하는 틱 수 계산 */
    return (uint32)(IfxStm_getFrequency(&MODULE_STM0) / 1000u);
}

IFX_INTERRUPT(STM0_ISR, 0, ISR_PRIO_STM)
{
    IfxSrc_clearRequest(&SRC_STM0SR0);
    IfxStm_updateCompare(&MODULE_STM0, IfxStm_Comparator_0,
                         IfxStm_getLower(&MODULE_STM0) + STM_getTicksPerMs());
    g_tick_1ms++;
}

static void STM_init(void)
{
    IfxStm_updateCompare(&MODULE_STM0, IfxStm_Comparator_0,
                         IfxStm_getLower(&MODULE_STM0) + STM_getTicksPerMs());
    IfxSrc_init(&SRC_STM0SR0, IfxSrc_Tos_cpu0, ISR_PRIO_STM);
    IfxSrc_enable(&SRC_STM0SR0);
}

/* 호출 시점으로부터 ms 밀리초 대기 */
void delay_ms(uint32 ms)
{
    uint32 start = g_tick_1ms;
    while ((g_tick_1ms - start) < ms);
}

int core0_main(void)
{
    STM_init();
    __enable();

    while (1)
    {
        /* 10ms마다 실행 */
        if (g_tick_1ms % 10u == 0u)
        {
            /* 10ms 주기 태스크 */
        }

        /* 100ms마다 실행 */
        if (g_tick_1ms % 100u == 0u)
        {
            /* 100ms 주기 태스크 */
        }
    }
    return 0;
}
```

> `g_tick_1ms`가 32비트라서 약 49일 후 오버플로가 발생한다.  
> `(g_tick_1ms - start) < ms` 방식의 비교는 오버플로가 나도 정상 동작한다  
> (부호 없는 정수 뺄셈 wrapping 특성 이용).

---

## 11. 예제 — STM으로 경과 시간 측정

> 코드 구간의 실행 시간을 마이크로초 단위로 측정한다.  
> 함수 처리 시간, 통신 응답 시간 측정에 유용하다.

```c
#include "IfxStm.h"

static uint32 g_stm_freq = 0u;

static void STM_initMeasure(void)
{
    g_stm_freq = (uint32)IfxStm_getFrequency(&MODULE_STM0);
}

static uint32 STM_getTimestamp(void)
{
    return IfxStm_getLower(&MODULE_STM0);
}

/* 두 타임스탬프 사이의 경과 시간 (마이크로초) */
static uint32 STM_elapsedUs(uint32 start, uint32 end)
{
    /* 오버플로 대응: 부호 없는 뺄셈으로 처리 */
    uint32 delta_ticks = end - start;
    return (uint32)((uint64)delta_ticks * 1000000u / g_stm_freq);
}

int core0_main(void)
{
    STM_initMeasure();

    while (1)
    {
        uint32 t_start = STM_getTimestamp();

        /* 측정할 코드 구간 */
        doSomeWork();

        uint32 t_end  = STM_getTimestamp();
        uint32 elapsed = STM_elapsedUs(t_start, t_end);
        /* elapsed: 실행 시간 (μs) */
    }
    return 0;
}
```

---

## 정리

### STM

| 항목 | 레지스터 | iLLD |
|---|---|---|
| 카운터 읽기 | `STMx_TIM0.U` | `IfxStm_getLower()` |
| 64비트 읽기 | `TIM0` + `TIM1` 조합 | `IfxStm_get()` |
| Compare 설정 | `STMx_CMP0` 직접 | `IfxStm_updateCompare()` |
| 클럭 주파수 | SCU 클럭 계산 | `IfxStm_getFrequency()` |
| ms → 틱 | 직접 계산 | `IfxStm_getTicksFromMilliseconds()` |

### GTM TIM

| 항목 | 레지스터 | iLLD |
|---|---|---|
| 채널 초기화 | `TIMx_CHy_CTRL` 직접 | `IfxGtm_Tim_In_init()` |
| 핀 라우팅 | IOCR 직접 | `cfg.filter.inputPin` |
| 캡처값 읽기 | `GPR0`, `GPR1` 직접 | `IfxGtm_Tim_In_update()` |
| 주기 계산 | `GPR0 / GTM_CLK` 직접 | `driver->periodUs` |
| Duty 계산 | `GPR1 / GPR0 × 100` 직접 | `driver->dutyPercent` |

STM은 **시간의 흐름**을 재는 자(ruler)고,  
GTM TIM은 **외부 신호의 파형**을 분석하는 도구다.  
두 가지를 함께 쓰면 "내 코드가 신호에 얼마나 빠르게 반응하는가"도 측정할 수 있다.

---

*다음 챕터: LIN — ASCLIN LIN 모드와 IfxAsclin_Lin*
