# 07. ADC — EVADC 구조와 IfxEvadc_Adc

## 목차

- [1. iLLD 계층 구조 — ADC 관점](#1-illd-계층-구조--adc-관점)
- [2. EVADC 동작 원리](#2-evadc-동작-원리)
- [3. 핵심 레지스터 요약](#3-핵심-레지스터-요약)
- [4. iLLD API — IfxEvadc_Adc](#4-illd-api--ifxevadc_adc)
- [5. 레지스터 방식 vs iLLD 비교](#5-레지스터-방식-vs-illd-비교)
  - [5-1. ADC 그룹 및 채널 설정](#5-1-adc-그룹-및-채널-설정)
  - [5-2. 변환 결과 읽기](#5-2-변환-결과-읽기)
- [6. 예제 — 단일 채널 폴링 변환](#6-예제--단일-채널-폴링-변환)
- [7. 예제 — 여러 채널 순차 변환](#7-예제--여러-채널-순차-변환)
- [8. 예제 — ADC 결과를 전압으로 변환](#8-예제--adc-결과를-전압으로-변환)
- [정리](#정리)

---

> EVADC(Enhanced Versatile ADC)는 AURIX의 ADC 모듈이다.  
> 최대 12비트 해상도, 복수의 ADC 그룹(Group)으로 구성되며  
> 각 그룹은 독립적으로 채널을 변환한다.

---

## 1. iLLD 계층 구조 — ADC 관점

```
Your Code
    │
    ▼
[Hld]  IfxEvadc_Adc.h / IfxEvadc_Adc.c
       iLLD/TC3xx/Tricore/Evadc/Adc/
       (그룹/채널 init, 큐 설정, 결과 읽기 래핑)
    │
    ▼
[Lld]  IfxEvadc.h / IfxEvadc.c
       iLLD/TC3xx/Tricore/Evadc/Std/
       (그룹 클럭, 채널 설정, 레지스터 접근)
    │
    ▼
[SFR]  EVADC_Gx_ARBCFG, CHCTR, RESD, QMR0 ...
    │
    ▼
[HW]   아날로그 입력 핀 (ANx)
```

---

## 2. EVADC 동작 원리

```
ADC 구조
EVADC
├── Group 0 (독립 ADC 코어)
│   ├── Channel 0 ~ 11  (아날로그 입력)
│   ├── Queue 0         (변환 요청 큐)
│   └── Result Register (변환 결과 저장)
├── Group 1
│   └── ...
└── Group N
```

**변환 흐름:**
```
채널 번호를 Queue에 추가
    │
    ▼
Arbiter가 Queue 순서대로 채널 선택
    │
    ▼
SAR(Successive Approximation) 변환 수행
    │
    ▼
결과 → Channel Result Register (RESD)
    │
    ▼
VF(Valid Flag) = 1 → 결과 준비 완료
```

**해상도와 기준 전압:**
```
결과값 범위: 0 ~ 4095 (12비트)
전압 = 결과값 × VREF / 4095
예) 결과 = 2048, VREF = 3.3V → 전압 ≈ 1.65V
```

---

## 3. 핵심 레지스터 요약

| 레지스터 | 역할 |
|---|---|
| `EVADC_Gx_ARBCFG` | 그룹 Arbiter 설정 (변환 모드) |
| `EVADC_Gx_ARBPR` | Queue 우선순위 설정 |
| `EVADC_Gx_CHCTR_y` | 채널 y 설정 (기준 전압, 변환 모드) |
| `EVADC_Gx_QMR0` | Queue 동작 모드 (enable, refill) |
| `EVADC_Gx_QINR0` | Queue에 변환 요청 추가 |
| `EVADC_Gx_RES_y` | 채널 y 변환 결과 (12비트) |
| `EVADC_Gx_RESD_y` | 결과 + Valid Flag (읽으면 VF 클리어) |

---

## 4. iLLD API — IfxEvadc_Adc

파일 위치: `iLLD/TC3xx/Tricore/Evadc/Adc/IfxEvadc_Adc.h`

| 함수 | 동작 |
|---|---|
| `IfxEvadc_Adc_initModuleConfig(config, evadc)` | 모듈 config 기본값 초기화 |
| `IfxEvadc_Adc_initModule(driver, config)` | EVADC 모듈 초기화 |
| `IfxEvadc_Adc_initGroupConfig(config, driver)` | 그룹 config 기본값 초기화 |
| `IfxEvadc_Adc_initGroup(group, config)` | ADC 그룹 초기화 |
| `IfxEvadc_Adc_initChannelConfig(config, group)` | 채널 config 기본값 초기화 |
| `IfxEvadc_Adc_initChannel(channel, config)` | ADC 채널 초기화 |
| `IfxEvadc_Adc_addToQueue(channel, refill)` | 채널을 변환 Queue에 추가 |
| `IfxEvadc_Adc_startQueue(group)` | Queue 변환 시작 |
| `IfxEvadc_Adc_getResult(channel)` | 변환 결과 읽기 |

EVADC는 **모듈 → 그룹 → 채널** 3단계 계층 초기화가 필요하다.  
다른 peripheral보다 init 단계가 하나 더 많다.

---

## 5. 레지스터 방식 vs iLLD 비교

### 5-1. ADC 그룹 및 채널 설정

**레지스터 직접 접근**
```c
/* EVADC Group 0, Channel 0 설정 */
/* 1. 모듈 클럭 enable */
EVADC_CLC.B.DISR = 0u;

/* 2. 그룹 0 Arbiter 설정 */
EVADC_G0_ARBCFG.B.ASEN0 = 1u;  /* Queue 0 enable */
EVADC_G0_ARBPR.B.PRIO0  = 3u;  /* Queue 0 최고 우선순위 */

/* 3. 채널 0: 내부 기준 전압, 12비트 */
EVADC_G0_CHCTR0.B.ICLSEL = 0u;   /* 내부 기준 */
EVADC_G0_CHCTR0.B.RESREG = 0u;   /* 결과 → Result Register 0 */

/* 4. Queue에 채널 추가 */
EVADC_G0_QINR0.B.REQCHNR = 0u;   /* 채널 0 요청 */
EVADC_G0_QINR0.B.RF       = 1u;   /* refill: 변환 후 자동 재등록 */

/* 5. Queue 시작 */
EVADC_G0_QMR0.B.ENGT = 1u;
```

**iLLD**
```c
#include "IfxEvadc_Adc.h"

IfxEvadc_Adc        g_evadc;
IfxEvadc_Adc_Group  g_group;
IfxEvadc_Adc_Channel g_channel;

/* 모듈 초기화 */
IfxEvadc_Adc_Config moduleCfg;
IfxEvadc_Adc_initModuleConfig(&moduleCfg, &MODULE_EVADC);
IfxEvadc_Adc_initModule(&g_evadc, &moduleCfg);

/* 그룹 초기화 */
IfxEvadc_Adc_GroupConfig groupCfg;
IfxEvadc_Adc_initGroupConfig(&groupCfg, &g_evadc);
groupCfg.groupId = IfxEvadc_GroupId_0;
groupCfg.master  = groupCfg.groupId;
IfxEvadc_Adc_initGroup(&g_group, &groupCfg);

/* 채널 초기화 */
IfxEvadc_Adc_ChannelConfig chCfg;
IfxEvadc_Adc_initChannelConfig(&chCfg, &g_group);
chCfg.channelId       = IfxEvadc_ChannelId_0;
chCfg.resultRegister  = IfxEvadc_ChannelResult_0;
IfxEvadc_Adc_initChannel(&g_channel, &chCfg);

/* Queue에 추가 후 시작 */
IfxEvadc_Adc_addToQueue(&g_channel, IfxEvadc_RequestSource_queue0,
                        IFXEVADC_QUEUE_REFILL);
IfxEvadc_Adc_startQueue(&g_group, IfxEvadc_RequestSource_queue0);
```

---

### 5-2. 변환 결과 읽기

**레지스터 직접 접근**
```c
/* Valid Flag가 1이 될 때까지 대기 */
Ifx_EVADC_G_RESD result;
do {
    result = EVADC_G0_RESD0;
} while (result.B.VF == 0u);

uint16 adc_val = (uint16)result.B.RESULT;
```

**iLLD**
```c
Ifx_EVADC_G_RES result;
do {
    result = IfxEvadc_Adc_getResult(&g_channel);
} while (!result.B.VF);

uint16 adc_val = (uint16)result.B.RESULT;
```

---

## 6. 예제 — 단일 채널 폴링 변환

> EVADC Group 0, Channel 0, 폴링 방식.  
> 아날로그 입력 핀: AN0 (그룹/핀 매핑은 데이터시트 확인)

```c
#include "IfxEvadc_Adc.h"

static IfxEvadc_Adc         g_evadc;
static IfxEvadc_Adc_Group   g_group;
static IfxEvadc_Adc_Channel g_channel;

static void ADC_init(void)
{
    IfxEvadc_Adc_Config moduleCfg;
    IfxEvadc_Adc_initModuleConfig(&moduleCfg, &MODULE_EVADC);
    IfxEvadc_Adc_initModule(&g_evadc, &moduleCfg);

    IfxEvadc_Adc_GroupConfig groupCfg;
    IfxEvadc_Adc_initGroupConfig(&groupCfg, &g_evadc);
    groupCfg.groupId = IfxEvadc_GroupId_0;
    groupCfg.master  = groupCfg.groupId;
    IfxEvadc_Adc_initGroup(&g_group, &groupCfg);

    IfxEvadc_Adc_ChannelConfig chCfg;
    IfxEvadc_Adc_initChannelConfig(&chCfg, &g_group);
    chCfg.channelId      = IfxEvadc_ChannelId_0;
    chCfg.resultRegister = IfxEvadc_ChannelResult_0;
    IfxEvadc_Adc_initChannel(&g_channel, &chCfg);

    IfxEvadc_Adc_addToQueue(&g_channel, IfxEvadc_RequestSource_queue0,
                            IFXEVADC_QUEUE_REFILL);
    IfxEvadc_Adc_startQueue(&g_group, IfxEvadc_RequestSource_queue0);
}

static uint16 ADC_read(void)
{
    Ifx_EVADC_G_RES result;
    do {
        result = IfxEvadc_Adc_getResult(&g_channel);
    } while (!result.B.VF);
    return (uint16)result.B.RESULT;
}

int core0_main(void)
{
    ADC_init();

    while (1)
    {
        uint16 adc_val = ADC_read();
        /* adc_val: 0~4095 */
        volatile uint32 i; for (i = 0; i < 100000u; i++);
    }
    return 0;
}
```

---

## 7. 예제 — 여러 채널 순차 변환

> Group 0에서 채널 0, 1, 2를 Queue에 등록해 순차 변환.

```c
#include "IfxEvadc_Adc.h"

#define ADC_CH_COUNT  3u

static IfxEvadc_Adc         g_evadc;
static IfxEvadc_Adc_Group   g_group;
static IfxEvadc_Adc_Channel g_channel[ADC_CH_COUNT];

static const IfxEvadc_ChannelId ch_ids[ADC_CH_COUNT] = {
    IfxEvadc_ChannelId_0,
    IfxEvadc_ChannelId_1,
    IfxEvadc_ChannelId_2,
};

static void ADC_init(void)
{
    IfxEvadc_Adc_Config moduleCfg;
    IfxEvadc_Adc_initModuleConfig(&moduleCfg, &MODULE_EVADC);
    IfxEvadc_Adc_initModule(&g_evadc, &moduleCfg);

    IfxEvadc_Adc_GroupConfig groupCfg;
    IfxEvadc_Adc_initGroupConfig(&groupCfg, &g_evadc);
    groupCfg.groupId = IfxEvadc_GroupId_0;
    groupCfg.master  = groupCfg.groupId;
    IfxEvadc_Adc_initGroup(&g_group, &groupCfg);

    for (uint8 i = 0u; i < ADC_CH_COUNT; i++)
    {
        IfxEvadc_Adc_ChannelConfig chCfg;
        IfxEvadc_Adc_initChannelConfig(&chCfg, &g_group);
        chCfg.channelId      = ch_ids[i];
        chCfg.resultRegister = (IfxEvadc_ChannelResult)i; /* 결과 레지스터 분리 */
        IfxEvadc_Adc_initChannel(&g_channel[i], &chCfg);

        IfxEvadc_Adc_addToQueue(&g_channel[i], IfxEvadc_RequestSource_queue0,
                                IFXEVADC_QUEUE_REFILL);
    }

    IfxEvadc_Adc_startQueue(&g_group, IfxEvadc_RequestSource_queue0);
}

static uint16 ADC_readChannel(uint8 idx)
{
    Ifx_EVADC_G_RES result;
    do {
        result = IfxEvadc_Adc_getResult(&g_channel[idx]);
    } while (!result.B.VF);
    return (uint16)result.B.RESULT;
}

int core0_main(void)
{
    ADC_init();

    uint16 adc_vals[ADC_CH_COUNT];
    while (1)
    {
        for (uint8 i = 0u; i < ADC_CH_COUNT; i++)
            adc_vals[i] = ADC_readChannel(i);

        volatile uint32 d; for (d = 0; d < 100000u; d++);
    }
    return 0;
}
```

> 채널마다 **결과 레지스터를 분리**(`chCfg.resultRegister`)하지 않으면  
> 마지막 채널 결과가 이전 결과를 덮어써서 값을 놓친다.

---

## 8. 예제 — ADC 결과를 전압으로 변환

```c
#include "IfxEvadc_Adc.h"

#define ADC_VREF_MV   3300u   /* 기준 전압 3.3V = 3300mV */
#define ADC_RESOLUTION 4096u  /* 12비트: 0~4095 */

/* ADC 결과 → 전압(mV) 변환 */
static uint32 ADC_toMillivolts(uint16 adc_raw)
{
    return ((uint32)adc_raw * ADC_VREF_MV) / ADC_RESOLUTION;
}

int core0_main(void)
{
    ADC_init();  /* 예제 6의 init 함수 */

    while (1)
    {
        uint16 raw   = ADC_read();
        uint32 mv    = ADC_toMillivolts(raw);
        /* mv: 0~3300 mV */

        volatile uint32 i; for (i = 0; i < 100000u; i++);
    }
    return 0;
}
```

> 정수 연산에서 곱셈을 먼저 해야 나눗셈 오차를 줄일 수 있다.  
> `(adc_raw / 4096) * 3300` 순서로 계산하면 정수 잘림으로 결과가 0이 된다.

---

## 정리

| 항목 | 레지스터 | iLLD |
|---|---|---|
| 모듈 클럭 enable | `EVADC_CLC.B.DISR = 0` | `initModule` 내부 처리 |
| 그룹 Arbiter 설정 | `Gx_ARBCFG`, `ARBPR` 직접 | `IfxEvadc_Adc_initGroup()` |
| 채널 설정 | `Gx_CHCTR_y` 직접 | `IfxEvadc_Adc_initChannel()` |
| 변환 요청 등록 | `Gx_QINR0.B.REQCHNR` 직접 | `IfxEvadc_Adc_addToQueue()` |
| 변환 시작 | `Gx_QMR0.B.ENGT = 1` | `IfxEvadc_Adc_startQueue()` |
| 결과 읽기 | `Gx_RESD_y.B.RESULT` + VF 확인 | `IfxEvadc_Adc_getResult()` |

EVADC는 **모듈 → 그룹 → 채널** 3단계 init이 필요하다.  
채널을 여러 개 쓸 때는 결과 레지스터를 반드시 분리해서 할당해야 값이 섞이지 않는다.

---

*다음 챕터: CAN — MULTICAN 구조와 IfxCan_Can*
