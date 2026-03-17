# 04. PWM — GTM/TOM 구조와 IfxGtm_Tom_Pwm

## 목차

- [1. iLLD 계층 구조 — PWM 관점](#1-illd-계층-구조--pwm-관점)
- [2. GTM/TOM 동작 원리](#2-gtmtom-동작-원리)
- [3. 핵심 레지스터 요약](#3-핵심-레지스터-요약)
- [4. iLLD API — IfxGtm_Tom_Pwm](#4-illd-api--ifxgtm_tom_pwm)
- [5. 레지스터 방식 vs iLLD 비교](#5-레지스터-방식-vs-illd-비교)
  - [5-1. GTM 클럭 및 TOM 채널 설정](#5-1-gtm-클럭-및-tom-채널-설정)
  - [5-2. Duty Cycle 변경](#5-2-duty-cycle-변경)
- [6. 예제 — 고정 주파수 PWM 출력](#6-예제--고정-주파수-pwm-출력)
- [7. 예제 — Duty Cycle 가변 (페이드 효과)](#7-예제--duty-cycle-가변-페이드-효과)
- [8. 예제 — 여러 채널 동기 출력](#8-예제--여러-채널-동기-출력)
- [정리](#정리)

---

> GPIO로 LED를 껐다 켰다 하는 것에서 한 단계 나아가,  
> **PWM(Pulse Width Modulation)** 으로 밝기나 모터 속도를 제어한다.  
> AURIX에서 PWM은 **GTM(Generic Timer Module)** 안의 **TOM(Timer Output Module)** 이 담당한다.  
> 여기서 처음으로 **Hld** 가 등장한다.

---

## 1. iLLD 계층 구조 — PWM 관점

```
Your Code
    │
    ▼
[Hld]  IfxGtm_Tom_Pwm.h / IfxGtm_Tom_Pwm.c
       iLLD/TC3xx/Tricore/Gtm/Tom/Pwm/
       (채널 init, duty 변경, start/stop 래핑)
    │
    ▼
[Lld]  IfxGtm.h / IfxGtm_Tom.h
       iLLD/TC3xx/Tricore/Gtm/Std/
       (GTM 클럭, TOM 레지스터 직접 접근)
    │
    ▼
[SFR]  GTM_TOM0_CH0_CN0, SR0, SR1, OUTEN ...
    │
    ▼
[HW]   GTM TOM 채널 → 핀 출력
```

GPIO(`IfxPort`)는 Hld가 없었지만, GTM은 초기화 순서가 복잡해서  
**Hld(`IfxGtm_Tom_Pwm`)가 Lld 호출을 묶어 `init/start/stop` 패턴으로 제공**한다.

---

## 2. GTM/TOM 동작 원리

```
GTM 클럭 분주
    │
    ▼
TOM Channel
├── CN0  : 현재 카운터 값 (0에서 SR0까지 증가 후 리셋)
├── SR0  : 주기 설정 (Period)   → PWM 주파수 결정
└── SR1  : 비교값 설정 (Compare) → Duty Cycle 결정
    │
    ▼
CN0 < SR1  → 출력 HIGH
CN0 >= SR1 → 출력 LOW
```

**주파수와 Duty Cycle 계산:**

```
PWM 주파수 = GTM 클럭 / SR0
Duty Cycle = SR1 / SR0 × 100 (%)

예) GTM 클럭 = 100MHz, SR0 = 1000
    → PWM 주파수 = 100kHz
    SR1 = 500 → Duty 50%
    SR1 = 250 → Duty 25%
```

---

## 3. 핵심 레지스터 요약

| 레지스터 | 역할 |
|---|---|
| `GTM_TOMx_CHy_CN0` | 채널 카운터 현재값 |
| `GTM_TOMx_CHy_SR0` | 주기 레지스터 (Period) |
| `GTM_TOMx_CHy_SR1` | 비교 레지스터 (Duty Cycle 기준) |
| `GTM_TOMx_CHy_CTRL` | 채널 동작 모드, 클럭 소스 선택 |
| `GTM_TOMx_TGC0_OUTEN` | 채널 출력 enable |
| `GTM_TOMx_TGC0_ENDIS` | 채널 enable / disable |
| `GTM_CMU_CLK_EN` | GTM 클럭 소스 enable |

---

## 4. iLLD API — IfxGtm_Tom_Pwm

파일 위치: `iLLD/TC3xx/Tricore/Gtm/Tom/Pwm/IfxGtm_Tom_Pwm.h`

| 함수 | 동작 |
|---|---|
| `IfxGtm_Tom_Pwm_initConfig(config, gtm)` | config 구조체를 기본값으로 초기화 |
| `IfxGtm_Tom_Pwm_init(driver, config)` | GTM 클럭, TOM 채널, 핀 라우팅 한 번에 설정 |
| `IfxGtm_Tom_Pwm_start(driver, state)` | PWM 출력 시작 / 중지 |
| `IfxGtm_Tom_Pwm_setDutyCycle(driver, duty)` | Duty Cycle 변경 (SR1 갱신) |
| `IfxGtm_Tom_Pwm_setPeriod(driver, period)` | 주기 변경 (SR0 갱신) |

Hld는 항상 **config 구조체 → init → start** 패턴을 따른다.  
이후 챕터의 ASCLIN, QSPI, EVADC도 동일한 패턴이다.

---

## 5. 레지스터 방식 vs iLLD 비교

### 5-1. GTM 클럭 및 TOM 채널 설정

레지스터로 GTM을 초기화하려면 클럭 enable → 채널 모드 → SR0/SR1 → 출력 enable 순서를  
빠짐없이 써야 한다. 하나라도 빠지면 출력이 나오지 않아서 디버깅이 까다롭다.

**레지스터 직접 접근**
```c
/* 1. GTM CMU 클럭 0 enable */
GTM_CMU_CLK_EN.B.EN_CLK0 = 0x2u;

/* 2. TOM0 CH0: SOMC 모드, CLK0 소스 */
GTM_TOM0_CH0_CTRL.B.CLK_SRC_SR = 0u;  /* CLK0 */
GTM_TOM0_CH0_CTRL.B.TRIGOUT     = 1u;  /* 출력 트리거 */

/* 3. 주기(SR0), Duty(SR1) 설정 — 100kHz, 50% */
GTM_TOM0_CH0_SR0.U = 1000u - 1u;
GTM_TOM0_CH0_SR1.U = 500u;

/* 4. 채널 출력 enable */
GTM_TOM0_TGC0_OUTEN.B.OUTEN_CTRL0 = 0x2u;
GTM_TOM0_TGC0_ENDIS.B.ENDIS_CTRL0 = 0x2u;

/* 5. 핀 라우팅 (P02.0 → TOM0 CH0) */
P2_IOCR0.B.PC0 = 0x11u;  /* GTM 출력 모드 */
```

**iLLD**
```c
#include "IfxGtm_Tom_Pwm.h"

IfxGtm_Tom_Pwm_Config pwmConfig;
IfxGtm_Tom_Pwm_Driver pwmDriver;

IfxGtm_Tom_Pwm_initConfig(&pwmConfig, &MODULE_GTM);

pwmConfig.tom          = IfxGtm_Tom_0;
pwmConfig.tomChannel   = IfxGtm_Tom_Ch_0;
pwmConfig.clock        = IfxGtm_Tom_Ch_ClkSrc_cmuFxclk0;
pwmConfig.period       = 1000u;        /* SR0: 주기 */
pwmConfig.dutyCycle    = 500u;         /* SR1: duty */
pwmConfig.pin.outputPin = &IfxGtm_TOM0_0_TOUT0_P02_0_OUT; /* 핀 자동 라우팅 */

IfxGtm_Tom_Pwm_init(&pwmDriver, &pwmConfig);
IfxGtm_Tom_Pwm_start(&pwmDriver, TRUE);
```

5단계 레지스터 시퀀스가 **config 구조체 + init 한 번**으로 줄어든다.  
핀 라우팅도 `outputPin` 멤버에 iLLD 핀 객체를 넣으면 자동으로 처리된다.

---

### 5-2. Duty Cycle 변경

**레지스터 직접 접근**
```c
/* SR1을 직접 변경 (Shadow Register → CN0 리셋 시 반영) */
GTM_TOM0_CH0_SR1.U = new_duty;
```

**iLLD**
```c
IfxGtm_Tom_Pwm_setDutyCycle(&pwmDriver, new_duty);
```

---

## 6. 예제 — 고정 주파수 PWM 출력

> PWM 주파수 1kHz, Duty 50% 고정 출력.  
> 출력 핀: P02.0 (TOM0 CH0에 연결된 핀)

```c
#include "Ifx_Types.h"
#include "IfxGtm_Tom_Pwm.h"

#define PWM_PERIOD    1000u   /* GTM 클럭 기준 주기 카운트 */
#define PWM_DUTY_50   500u    /* 50% duty */

static IfxGtm_Tom_Pwm_Driver g_pwmDriver;

static void PWM_init(void)
{
    IfxGtm_Tom_Pwm_Config cfg;
    IfxGtm_Tom_Pwm_initConfig(&cfg, &MODULE_GTM);

    cfg.tom           = IfxGtm_Tom_0;
    cfg.tomChannel    = IfxGtm_Tom_Ch_0;
    cfg.clock         = IfxGtm_Tom_Ch_ClkSrc_cmuFxclk0;
    cfg.period        = PWM_PERIOD;
    cfg.dutyCycle     = PWM_DUTY_50;
    cfg.pin.outputPin = &IfxGtm_TOM0_0_TOUT0_P02_0_OUT;

    IfxGtm_Tom_Pwm_init(&g_pwmDriver, &cfg);
    IfxGtm_Tom_Pwm_start(&g_pwmDriver, TRUE);
}

int core0_main(void)
{
    PWM_init();
    while (1) {}
    return 0;
}
```

---

## 7. 예제 — Duty Cycle 가변 (페이드 효과)

> STM 인터럽트로 10ms마다 duty를 1씩 증가/감소시켜 LED를 서서히 밝히고 어둡게 한다.  
> LED = P02.0, 주기 = 1000 카운트 (duty 0~1000)

```c
#include "IfxGtm_Tom_Pwm.h"
#include "IfxSrc.h"
#include "IfxStm.h"

#define PWM_PERIOD      1000u
#define STM_TICKS_10MS  1000000u   /* 100MHz 기준 10ms */
#define ISR_PRIO_STM    10u

static IfxGtm_Tom_Pwm_Driver g_pwmDriver;
static uint32 g_duty     = 0u;
static sint8  g_direction = 1;    /* 1: 증가, -1: 감소 */

IFX_INTERRUPT(STM0_ISR, 0, ISR_PRIO_STM)
{
    IfxSrc_clearRequest(&SRC_STM0SR0);
    IfxStm_updateCompare(&MODULE_STM0, IfxStm_Comparator_0,
                         IfxStm_getLower(&MODULE_STM0) + STM_TICKS_10MS);

    g_duty += g_direction * 10u;
    if (g_duty >= PWM_PERIOD) { g_duty = PWM_PERIOD; g_direction = -1; }
    if (g_duty == 0u)          {                       g_direction =  1; }

    IfxGtm_Tom_Pwm_setDutyCycle(&g_pwmDriver, g_duty);
}

static void PWM_init(void)
{
    IfxGtm_Tom_Pwm_Config cfg;
    IfxGtm_Tom_Pwm_initConfig(&cfg, &MODULE_GTM);
    cfg.tom           = IfxGtm_Tom_0;
    cfg.tomChannel    = IfxGtm_Tom_Ch_0;
    cfg.clock         = IfxGtm_Tom_Ch_ClkSrc_cmuFxclk0;
    cfg.period        = PWM_PERIOD;
    cfg.dutyCycle     = 0u;
    cfg.pin.outputPin = &IfxGtm_TOM0_0_TOUT0_P02_0_OUT;
    IfxGtm_Tom_Pwm_init(&g_pwmDriver, &cfg);
    IfxGtm_Tom_Pwm_start(&g_pwmDriver, TRUE);
}

static void STM_init(void)
{
    IfxStm_updateCompare(&MODULE_STM0, IfxStm_Comparator_0,
                         IfxStm_getLower(&MODULE_STM0) + STM_TICKS_10MS);
    IfxSrc_init(&SRC_STM0SR0, IfxSrc_Tos_cpu0, ISR_PRIO_STM);
    IfxSrc_enable(&SRC_STM0SR0);
}

int core0_main(void)
{
    PWM_init();
    STM_init();
    __enable();
    while (1) {}
    return 0;
}
```

---

## 8. 예제 — 여러 채널 동기 출력

> 두 개의 PWM 채널을 같은 주기로 서로 다른 duty로 동시에 출력.  
> CH0 = P02.0 (duty 25%), CH1 = P02.1 (duty 75%)

```c
#include "IfxGtm_Tom_Pwm.h"

#define PWM_PERIOD  1000u

static IfxGtm_Tom_Pwm_Driver g_pwm0;
static IfxGtm_Tom_Pwm_Driver g_pwm1;

static void PWM_initChannel(IfxGtm_Tom_Pwm_Driver *drv,
                             IfxGtm_Tom_Ch          ch,
                             const IfxGtm_Tom_ToutMap *pin,
                             uint32                 duty)
{
    IfxGtm_Tom_Pwm_Config cfg;
    IfxGtm_Tom_Pwm_initConfig(&cfg, &MODULE_GTM);
    cfg.tom           = IfxGtm_Tom_0;
    cfg.tomChannel    = ch;
    cfg.clock         = IfxGtm_Tom_Ch_ClkSrc_cmuFxclk0;
    cfg.period        = PWM_PERIOD;
    cfg.dutyCycle     = duty;
    cfg.pin.outputPin = pin;
    IfxGtm_Tom_Pwm_init(drv, &cfg);
    IfxGtm_Tom_Pwm_start(drv, TRUE);
}

int core0_main(void)
{
    PWM_initChannel(&g_pwm0, IfxGtm_Tom_Ch_0,
                    &IfxGtm_TOM0_0_TOUT0_P02_0_OUT, 250u);  /* 25% */
    PWM_initChannel(&g_pwm1, IfxGtm_Tom_Ch_1,
                    &IfxGtm_TOM0_1_TOUT1_P02_1_OUT, 750u);  /* 75% */
    while (1) {}
    return 0;
}
```

> 두 채널이 같은 TOM, 같은 클럭 소스를 쓰면 주기가 자동으로 동기화된다.  
> 채널을 다른 TOM에 배치하면 주기가 독립적으로 동작하므로 주의.

---

## 정리

| 항목 | 레지스터 | iLLD |
|---|---|---|
| 클럭 enable | `GTM_CMU_CLK_EN` 직접 | `initConfig` 내부 처리 |
| 채널 모드 설정 | `TOM_CHy_CTRL` 직접 | `cfg.clock`, `cfg.tom` |
| 주기 설정 | `TOM_CHy_SR0 = period` | `cfg.period` |
| Duty 설정 | `TOM_CHy_SR1 = duty` | `cfg.dutyCycle` |
| 핀 라우팅 | `IOCRx.PCy = 0x11` | `cfg.pin.outputPin` |
| 출력 시작 | `OUTEN/ENDIS` 직접 | `IfxGtm_Tom_Pwm_start()` |
| Duty 변경 | `SR1 = new_duty` | `IfxGtm_Tom_Pwm_setDutyCycle()` |

Hld를 쓰면 클럭 → 채널 → 핀 라우팅 순서 실수를 방지할 수 있다.  
단, `cfg.pin.outputPin`에 넣는 핀 객체 이름은 데이터시트 핀맵과  
iLLD의 `IfxGtm_PinMap.h`를 같이 보면서 확인해야 한다.

---

*다음 챕터: UART — ASCLIN 구조와 IfxAsclin_Asc*
