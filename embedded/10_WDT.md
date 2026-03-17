# 10. WDT — Watchdog Timer 구조와 IfxScuWdt

## 목차

- [1. iLLD 계층 구조 — WDT 관점](#1-illd-계층-구조--wdt-관점)
- [2. WDT 동작 원리](#2-wdt-동작-원리)
- [3. AURIX WDT 종류](#3-aurix-wdt-종류)
- [4. 핵심 레지스터 요약](#4-핵심-레지스터-요약)
- [5. iLLD API — IfxScuWdt](#5-illd-api--ifxscuwdt)
- [6. 레지스터 방식 vs iLLD 비교](#6-레지스터-방식-vs-illd-비교)
  - [6-1. WDT 서비스 (Refresh)](#6-1-wdt-서비스-refresh)
  - [6-2. WDT 비활성화](#6-2-wdt-비활성화)
- [7. 예제 — 정상 동작 중 WDT 서비스](#7-예제--정상-동작-중-wdt-서비스)
- [8. 예제 — WDT 타임아웃 주기 설정](#8-예제--wdt-타임아웃-주기-설정)
- [9. 예제 — WDT와 STM 인터럽트 연동](#9-예제--wdt와-stm-인터럽트-연동)
- [10. WDT 사용 시 주의사항](#10-wdt-사용-시-주의사항)
- [정리](#정리)

---

> WDT(Watchdog Timer)는 소프트웨어가 정상적으로 실행되고 있는지 감시하는 하드웨어다.  
> 정해진 시간 안에 소프트웨어가 WDT를 서비스(리프레시)하지 않으면  
> 시스템을 자동으로 리셋한다.  
> 다른 peripheral과 달리, **개발 초기에 WDT를 비활성화하는 방법을 먼저 알아야**  
> 다른 코드를 작성하는 동안 의도치 않은 리셋을 피할 수 있다.

---

## 1. iLLD 계층 구조 — WDT 관점

```
Your Code
    │
    ▼
[Lld]  IfxScuWdt.h / IfxScuWdt.c
       iLLD/TC3xx/Tricore/Scu/Std/
       (WDT는 단순해서 Hld 없이 Lld 직접 사용)
    │
    ▼
[SFR]  SCU_WDTCPU0CON0, SCU_WDTSCON0 (Safety WDT)
    │
    ▼
[HW]   WDT 카운터 → 타임아웃 시 CPU 리셋
```

GPIO처럼 Hld 없이 **Lld(IfxScuWdt)를 직접 호출**한다.

---

## 2. WDT 동작 원리

```
WDT 카운터
    │ 클럭으로 카운트 업
    ▼
타임아웃 전에 서비스(리프레시) → 카운터 리셋 → 정상
    │
    ▼ (서비스 안 함)
카운터 오버플로 → 경고(1차) → 리셋(2차)
```

**서비스 절차 (Password Protection):**

AURIX WDT는 패스워드 보호 메커니즘이 있다.  
서비스할 때 **정해진 순서**로 써야 하며, 틀리면 즉시 리셋된다.

```
1. 패스워드 접근: CON0에 현재 패스워드 XOR 값 쓰기
2. 서비스: CON0에 카운터 리셋 값 쓰기
```

이 두 단계 사이에 인터럽트가 끼어들어 순서가 깨지면 리셋이 발생한다.  
그래서 서비스 시퀀스는 **인터럽트 비활성화 상태에서** 실행해야 한다.

---

## 3. AURIX WDT 종류

| 종류 | 이름 | 대상 |
|---|---|---|
| CPU WDT | `SCU_WDTCPUx` | 각 CPU 코어별 (CPU0, CPU1, CPU2) |
| Safety WDT | `SCU_WDTS` | 전체 시스템 (기능 안전용) |

일반 개발에서는 **CPU0 WDT**를 주로 다룬다.  
Safety WDT는 ISO 26262 기능 안전 요구사항에 따라 별도 관리한다.

---

## 4. 핵심 레지스터 요약

| 레지스터 | 역할 |
|---|---|
| `SCU_WDTCPU0CON0` | CPU0 WDT 제어 (서비스, 패스워드, enable) |
| `SCU_WDTCPU0CON1` | CPU0 WDT 타임아웃 주기 설정 |
| `SCU_WDTCPU0SR` | CPU0 WDT 상태 (타임아웃 발생 여부) |
| `SCU_WDTSCON0` | Safety WDT 제어 |
| `SCU_WDTSCON1` | Safety WDT 타임아웃 주기 설정 |

**CON0 패스워드 보호 동작:**  
CON0를 읽으면 현재 패스워드(PW 필드)가 반전된 값으로 나온다.  
서비스할 때 이 값을 그대로 쓰면 패스워드 검증이 통과한다.

---

## 5. iLLD API — IfxScuWdt

파일 위치: `iLLD/TC3xx/Tricore/Scu/Std/IfxScuWdt.h`

| 함수 | 동작 |
|---|---|
| `IfxScuWdt_getCpuWatchdogPassword(wdt)` | 현재 CPU WDT 패스워드 읽기 |
| `IfxScuWdt_serviceCpuWatchdog(wdt, password)` | CPU WDT 서비스 (리프레시) |
| `IfxScuWdt_disableCpuWatchdog(wdt, password)` | CPU WDT 비활성화 |
| `IfxScuWdt_enableCpuWatchdog(wdt, password)` | CPU WDT 활성화 |
| `IfxScuWdt_changeCpuWatchdogReload(wdt, password, reload)` | 타임아웃 주기 변경 |
| `IfxScuWdt_getSafetyWatchdogPassword()` | Safety WDT 패스워드 읽기 |
| `IfxScuWdt_serviceSafetyWatchdog(password)` | Safety WDT 서비스 |
| `IfxScuWdt_disableSafetyWatchdog(password)` | Safety WDT 비활성화 |

모든 함수는 패스워드를 인자로 받는다.  
**패스워드는 매 서비스 호출 직전에 새로 읽어야 한다.**

---

## 6. 레지스터 방식 vs iLLD 비교

### 6-1. WDT 서비스 (Refresh)

**레지스터 직접 접근**
```c
/* 인터럽트 비활성화 */
uint32 saved_ie = __disable_and_save();

/* 1단계: 패스워드 접근 */
SCU_WDTCPU0CON0.U = ((SCU_WDTCPU0CON0.U ^ 0x000000FCu) & ~0x00000003u)
                   | 0x00000002u;

/* 2단계: 서비스 (카운터 리셋) */
SCU_WDTCPU0CON0.U = ((SCU_WDTCPU0CON0.U ^ 0x000000FCu) & ~0x00000003u)
                   | 0x00000003u;

/* 인터럽트 복원 */
__restore(saved_ie);
```

**iLLD**
```c
#include "IfxScuWdt.h"

uint16 password = IfxScuWdt_getCpuWatchdogPassword(&MODULE_SCU.WDTCPU[0]);
IfxScuWdt_serviceCpuWatchdog(&MODULE_SCU.WDTCPU[0], password);
```

패스워드 XOR 계산과 인터럽트 보호가 내부에서 처리된다.

---

### 6-2. WDT 비활성화

개발 초기, 또는 WDT가 필요 없는 구간에서 비활성화한다.

**레지스터 직접 접근**
```c
uint32 saved_ie = __disable_and_save();

/* 패스워드 접근 후 WDTDR(Disable Request) 비트 설정 */
SCU_WDTCPU0CON0.U = ((SCU_WDTCPU0CON0.U ^ 0x000000FCu) & ~0x00000003u)
                   | 0x00000002u;
SCU_WDTCPU0CON0.U = ((SCU_WDTCPU0CON0.U ^ 0x000000FCu) & ~0x00000001u)
                   | (1u << 3u);  /* WDTDR = 1 */

__restore(saved_ie);
```

**iLLD**
```c
uint16 password = IfxScuWdt_getCpuWatchdogPassword(&MODULE_SCU.WDTCPU[0]);
IfxScuWdt_disableCpuWatchdog(&MODULE_SCU.WDTCPU[0], password);
```

---

## 7. 예제 — 정상 동작 중 WDT 서비스

> 개발 초기 코드. CPU WDT + Safety WDT를 비활성화하거나,  
> 메인 루프에서 주기적으로 서비스하는 두 가지 패턴.

**패턴 A — 개발 중 WDT 비활성화 (권장하지 않음, 디버깅 목적)**
```c
#include "IfxScuWdt.h"

void WDT_disableAll(void)
{
    /* CPU0 WDT 비활성화 */
    uint16 cpuPw = IfxScuWdt_getCpuWatchdogPassword(&MODULE_SCU.WDTCPU[0]);
    IfxScuWdt_disableCpuWatchdog(&MODULE_SCU.WDTCPU[0], cpuPw);

    /* Safety WDT 비활성화 */
    uint16 safePw = IfxScuWdt_getSafetyWatchdogPassword();
    IfxScuWdt_disableSafetyWatchdog(safePw);
}

int core0_main(void)
{
    WDT_disableAll();
    /* 이후 WDT 걱정 없이 개발 */
    while (1) {}
    return 0;
}
```

**패턴 B — 메인 루프에서 주기적으로 서비스 (실제 운용)**
```c
#include "IfxScuWdt.h"

int core0_main(void)
{
    /* WDT는 기본 활성화 상태 유지 */

    while (1)
    {
        /* 작업 수행 */
        doMainTask();

        /* 루프마다 WDT 서비스 — 타임아웃 주기보다 짧게 호출해야 함 */
        uint16 pw = IfxScuWdt_getCpuWatchdogPassword(&MODULE_SCU.WDTCPU[0]);
        IfxScuWdt_serviceCpuWatchdog(&MODULE_SCU.WDTCPU[0], pw);
    }
    return 0;
}
```

---

## 8. 예제 — WDT 타임아웃 주기 설정

WDT 타임아웃은 `reload` 값으로 설정한다.  
카운터가 0xFFFF까지 올라가는 데 걸리는 시간이 타임아웃이다.

```c
#include "IfxScuWdt.h"

/*
 * WDT 클럭 = SPB 클럭 / 16384 (기본 분주비)
 * SPB 클럭 = 100MHz → WDT 클럭 ≈ 6104 Hz
 * 타임아웃 = (0xFFFF - reload) / WDT_CLK
 *
 * reload = 0x0000 → 타임아웃 ≈ 10.7초 (최대)
 * reload = 0xFF00 → 타임아웃 ≈ 42ms
 */
#define WDT_RELOAD_1S   0xFFFF - 6104u   /* 약 1초 */

void WDT_setTimeout1s(void)
{
    uint16 pw = IfxScuWdt_getCpuWatchdogPassword(&MODULE_SCU.WDTCPU[0]);
    IfxScuWdt_changeCpuWatchdogReload(&MODULE_SCU.WDTCPU[0], pw, WDT_RELOAD_1S);
}

int core0_main(void)
{
    WDT_setTimeout1s();

    while (1)
    {
        doMainTask();  /* 1초 안에 반드시 완료되어야 함 */

        uint16 pw = IfxScuWdt_getCpuWatchdogPassword(&MODULE_SCU.WDTCPU[0]);
        IfxScuWdt_serviceCpuWatchdog(&MODULE_SCU.WDTCPU[0], pw);
    }
    return 0;
}
```

> **reload 계산은 실제 SPB 클럭을 측정해서 계산하는 것이 정확하다.**  
> `IfxScuCcu_getSpbFrequency()`로 실제 클럭을 읽어 동적으로 계산하면  
> 클럭 설정이 바뀌어도 타임아웃이 유지된다.

---

## 9. 예제 — WDT와 STM 인터럽트 연동

STM 인터럽트로 정확한 주기마다 WDT를 서비스하는 패턴.  
메인 루프 실행 시간이 가변적일 때 유용하다.

```c
#include "IfxScuWdt.h"
#include "IfxSrc.h"
#include "IfxStm.h"

#define ISR_PRIO_STM    10u
#define STM_TICKS_500MS 50000000u  /* 100MHz 기준 500ms */

IFX_INTERRUPT(STM0_WDT_ISR, 0, ISR_PRIO_STM)
{
    IfxSrc_clearRequest(&SRC_STM0SR0);
    IfxStm_updateCompare(&MODULE_STM0, IfxStm_Comparator_0,
                         IfxStm_getLower(&MODULE_STM0) + STM_TICKS_500MS);

    /* 500ms마다 WDT 서비스 (타임아웃 > 500ms로 설정된 경우) */
    uint16 pw = IfxScuWdt_getCpuWatchdogPassword(&MODULE_SCU.WDTCPU[0]);
    IfxScuWdt_serviceCpuWatchdog(&MODULE_SCU.WDTCPU[0], pw);
}

static void STM_init(void)
{
    IfxStm_updateCompare(&MODULE_STM0, IfxStm_Comparator_0,
                         IfxStm_getLower(&MODULE_STM0) + STM_TICKS_500MS);
    IfxSrc_init(&SRC_STM0SR0, IfxSrc_Tos_cpu0, ISR_PRIO_STM);
    IfxSrc_enable(&SRC_STM0SR0);
}

int core0_main(void)
{
    STM_init();
    __enable();

    while (1)
    {
        /* 메인 루프 작업 — WDT 서비스는 STM ISR에서 처리 */
        doMainTask();
    }
    return 0;
}
```

> ISR에서 WDT를 서비스할 때도 **패스워드를 매번 새로 읽어야 한다.**  
> 패스워드는 서비스할 때마다 하드웨어가 변경하므로,  
> 이전에 읽어둔 값을 재사용하면 리셋이 발생한다.

---

## 10. WDT 사용 시 주의사항

**1. 패스워드는 매번 새로 읽을 것**
```c
/* 틀린 방법 — password를 한 번 읽어서 재사용 */
uint16 pw = IfxScuWdt_getCpuWatchdogPassword(&MODULE_SCU.WDTCPU[0]);
while (1) {
    IfxScuWdt_serviceCpuWatchdog(&MODULE_SCU.WDTCPU[0], pw);  /* 2회차부터 리셋 */
}

/* 올바른 방법 — 서비스할 때마다 패스워드 새로 읽기 */
while (1) {
    uint16 pw = IfxScuWdt_getCpuWatchdogPassword(&MODULE_SCU.WDTCPU[0]);
    IfxScuWdt_serviceCpuWatchdog(&MODULE_SCU.WDTCPU[0], pw);
}
```

**2. Safety WDT도 잊지 말 것**
```c
/* CPU WDT만 서비스하고 Safety WDT를 놓치는 경우가 많다 */
void WDT_serviceAll(void)
{
    uint16 cpuPw  = IfxScuWdt_getCpuWatchdogPassword(&MODULE_SCU.WDTCPU[0]);
    IfxScuWdt_serviceCpuWatchdog(&MODULE_SCU.WDTCPU[0], cpuPw);

    uint16 safePw = IfxScuWdt_getSafetyWatchdogPassword();
    IfxScuWdt_serviceSafetyWatchdog(safePw);
}
```

**3. 서비스 시퀀스 사이에 인터럽트 주의**

iLLD `serviceCpuWatchdog` 내부에서 인터럽트를 잠시 차단한다.  
인터럽트 latency에 민감한 시스템이라면 WDT 서비스 호출 위치를 신중히 선택할 것.

**4. 멀티코어 환경**
```c
/* 각 코어는 자신의 WDT를 서비스해야 한다 */
/* CPU1에서 실행되는 코드 */
uint16 pw = IfxScuWdt_getCpuWatchdogPassword(&MODULE_SCU.WDTCPU[1]);
IfxScuWdt_serviceCpuWatchdog(&MODULE_SCU.WDTCPU[1], pw);
```

---

## 정리

| 항목 | 레지스터 | iLLD |
|---|---|---|
| 패스워드 읽기 | `CON0.B.PW` 직접 XOR 계산 | `IfxScuWdt_getCpuWatchdogPassword()` |
| WDT 서비스 | 2단계 패스워드 시퀀스 직접 | `IfxScuWdt_serviceCpuWatchdog()` |
| WDT 비활성화 | WDTDR 비트 패스워드 시퀀스 | `IfxScuWdt_disableCpuWatchdog()` |
| 타임아웃 변경 | CON1.B.IR 직접 | `IfxScuWdt_changeCpuWatchdogReload()` |
| Safety WDT 서비스 | WDTSCON0 직접 | `IfxScuWdt_serviceSafetyWatchdog()` |

WDT는 다른 peripheral과 반대로 **먼저 끄는 방법을 알고 시작**하는 것이 좋다.  
개발 완료 후 WDT를 다시 활성화하고 서비스 주기를 맞춰가는 것이  
예상치 못한 리셋으로 시간을 낭비하지 않는 방법이다.
