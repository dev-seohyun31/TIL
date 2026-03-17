# 02. GPIO — iLLD 계층 구조로 다시 보기

## 목차

- [1. iLLD 계층 구조 — GPIO 관점](#1-illd-계층-구조--gpio-관점)
- [2. 핵심 레지스터 요약](#2-핵심-레지스터-요약)
- [3. iLLD API — IfxPort](#3-illd-api--ifxport)
- [4. 레지스터 방식 vs iLLD 비교](#4-레지스터-방식-vs-illd-비교)
  - [4-1. 핀 출력 설정 (IOCR)](#4-1-핀-출력-설정-iocr)
  - [4-2. 핀 HIGH / LOW / Toggle (OMR)](#4-2-핀-high--low--toggle-omr)
  - [4-3. 핀 입력 읽기 (IN)](#4-3-핀-입력-읽기-in)
  - [4-4. 여러 핀 한 번에 쓰기](#4-4-여러-핀-한-번에-쓰기-omr-직접-vs-setgroupstate)
- [5. 예제 — LED Blink](#5-예제--led-blink)
- [6. 예제 — 버튼으로 LED 제어](#6-예제--버튼으로-led-제어)
- [7. 예제 — 버튼 디바운싱](#7-예제--버튼-디바운싱)
- [8. 예제 — LED 여러 개 동시 제어](#8-예제--led-여러-개-동시-제어-setgroupstate)
- [9. 예제 — 포트 전체 비트 한 번에 쓰기](#9-예제--포트-전체-비트-한-번에-쓰기-out-직접)
- [10. 예제 — Active High / Active Low 래퍼](#10-예제--active-high--active-low-래퍼)
- [정리](#정리)

---

> 앞 챕터에서 PORT_IOCR, PORT_OMR 레지스터를 직접 다뤄봤다면,  
> 이번 챕터에서는 **같은 동작을 iLLD로 표현**하고 내부 구조가 어떻게 연결되는지 확인한다.

---

## 1. iLLD 계층 구조 — GPIO 관점

```
Your Code
    │
    ▼
[Hld] — GPIO는 단순해서 Hld 없음
    │
    ▼
[Lld]  IfxPort.h / IfxPort.c
       iLLD/TC3xx/Tricore/Port/Std/
    │
    ▼
[SFR]  PORT_xx_IOCR, PORT_xx_OMR, PORT_xx_IN
       (Ifx_reg.h 에 구조체로 정의됨)
    │
    ▼
[HW]   실제 GPIO 핀
```

GTM, ASCLIN 같은 복잡한 peripheral은 Hld가 Lld 호출을 묶어주지만,  
GPIO는 동작이 단순하기 때문에 **Lld(IfxPort)를 직접 호출**하는 것으로 끝난다.

---

## 2. 핵심 레지스터 요약

| 레지스터 | 역할 |
|---|---|
| `PORT_xx_IOCR0~IOCR12` | 핀 방향 및 드라이브 모드 설정 (input / push-pull output 등) |
| `PORT_xx_OMR` | 출력 값 제어 (set / clear / toggle, write-only) |
| `PORT_xx_OUT` | 현재 출력 래치 값 확인 (read) |
| `PORT_xx_IN` | 핀 입력 값 읽기 |

> `OMR`(Output Modification Register)은 set/clear 비트가 분리되어 있어서  
> read-modify-write 없이 원자적으로 핀을 조작할 수 있다.

---

## 3. iLLD API — IfxPort

파일 위치: `iLLD/TC3xx/Tricore/Port/Std/IfxPort.h`

| 함수 | 동작 |
|---|---|
| `IfxPort_setPinModeOutput(port, pin, mode, padDriver)` | IOCR 설정 — 핀을 출력으로 구성 |
| `IfxPort_setPinModeInput(port, pin, mode)` | IOCR 설정 — 핀을 입력으로 구성 |
| `IfxPort_setPinHigh(port, pin)` | OMR.PS 비트 set → 핀 HIGH |
| `IfxPort_setPinLow(port, pin)` | OMR.PCL 비트 set → 핀 LOW |
| `IfxPort_togglePin(port, pin)` | OUT 읽고 OMR로 반전 |
| `IfxPort_getPinState(port, pin)` | IN 레지스터 읽기 → 0 or 1 |
| `IfxPort_setGroupState(port, mask, data)` | OMR로 여러 핀을 한 번에 제어 |

`mode` 인자는 `IfxPort_OutputMode` 또는 `IfxPort_InputMode` enum으로 지정한다.  
자주 쓰는 값: `IfxPort_OutputMode_pushPull`, `IfxPort_InputMode_pullUp`.

---

## 4. 레지스터 방식 vs iLLD 비교

### 4-1. 핀 출력 설정 (IOCR)

**레지스터 직접 접근**
```c
/* P0.5를 push-pull output으로 설정 */
/* IOCR4의 PC5 필드 (bit[4:0]) = 0x10 (push-pull output) */
P0_IOCR4.B.PC5 = 0x10;
```

**iLLD**
```c
#include "IfxPort.h"

IfxPort_setPinModeOutput(&MODULE_P00, 5,
                         IfxPort_OutputMode_pushPull,
                         IfxPort_PadDriver_cmosAutomotiveSpeed1);
```

내부적으로 `IfxPort.c`는 동일하게 `IOCR4.B.PC5`에 값을 쓴다.  
드라이브 강도(pad driver) 설정까지 한 번에 처리해주는 점이 다르다.

> **IOCR 레지스터 번호 계산**  
> 핀 번호 ÷ 4 = IOCR 번호, 핀 번호 % 4 = 필드 인덱스  
> P0.5 → IOCR**4**, PC**5** (5 / 4 = 1 → IOCR4, 5 % 4 = 1 → PC5)  
> — iLLD를 쓰면 이 계산을 신경 쓸 필요가 없다.

---

### 4-2. 핀 HIGH / LOW / Toggle (OMR)

**레지스터 직접 접근**
```c
/* P0.5 HIGH */
P0_OMR.B.PS5  = 1;   /* Set bit */

/* P0.5 LOW */
P0_OMR.B.PCL5 = 1;   /* Clear bit */

/* P0.5 Toggle */
if (P0_OUT.B.P5)
    P0_OMR.B.PCL5 = 1;
else
    P0_OMR.B.PS5  = 1;
```

**iLLD**
```c
IfxPort_setPinHigh(&MODULE_P00, 5);
IfxPort_setPinLow(&MODULE_P00, 5);
IfxPort_togglePin(&MODULE_P00, 5);
```

---

### 4-3. 핀 입력 읽기 (IN)

**레지스터 직접 접근**
```c
uint8 pin_state = P0_IN.B.P5;
```

**iLLD**
```c
uint8 pin_state = IfxPort_getPinState(&MODULE_P00, 5);
```

---

### 4-4. 여러 핀 한 번에 쓰기 (OMR 직접 vs setGroupState)

포트의 여러 핀을 동시에 바꿔야 할 때, 핀을 하나씩 호출하면  
함수 호출 사이에 핀 상태가 어긋나는 순간이 생긴다.  
OMR에 마스크를 한 번에 쓰면 원자적으로 처리된다.

**레지스터 직접 접근**
```c
/* P0.5, P0.6 동시에 HIGH / P0.7 LOW */
/* OMR: 상위 16비트 = clear(PCL), 하위 16비트 = set(PS) */
P0_OMR.U = (1u << 23)   /* PCL7 */
          | (1u << 5)    /* PS5  */
          | (1u << 6);   /* PS6  */
```

**iLLD**
```c
/* mask: 건드릴 핀, data: 1=HIGH 0=LOW */
/* P0.5=HIGH, P0.6=HIGH, P0.7=LOW */
IfxPort_setGroupState(&MODULE_P00,
                      (1u << 5) | (1u << 6) | (1u << 7),  /* mask */
                      (1u << 5) | (1u << 6));              /* data */
```

내부적으로 `setGroupState`는 mask/data로 OMR의 PS/PCL 비트를 계산해서  
레지스터에 한 번만 쓴다.

---

## 5. 예제 — LED Blink

> LED = P0.5 (active low), 딜레이는 단순 루프

```c
#include "IfxPort.h"

#define LED_PORT  (&MODULE_P00)
#define LED_PIN   5

void GPIO_init(void)
{
    IfxPort_setPinModeOutput(LED_PORT, LED_PIN,
                             IfxPort_OutputMode_pushPull,
                             IfxPort_PadDriver_cmosAutomotiveSpeed1);
    IfxPort_setPinHigh(LED_PORT, LED_PIN); /* 초기 상태: OFF (active low) */
}

int core0_main(void)
{
    GPIO_init();

    while (1)
    {
        IfxPort_togglePin(LED_PORT, LED_PIN);
        volatile uint32 i;
        for (i = 0; i < 1000000; i++);
    }
    return 0;
}
```

---

## 6. 예제 — 버튼으로 LED 제어

> LED = P0.5 (active low), 버튼 = P15.13 (active low, pull-up)

```c
#include "IfxPort.h"

#define LED_PORT  (&MODULE_P00)
#define LED_PIN   5
#define BTN_PORT  (&MODULE_P15)
#define BTN_PIN   13

void GPIO_init(void)
{
    IfxPort_setPinModeOutput(LED_PORT, LED_PIN,
                             IfxPort_OutputMode_pushPull,
                             IfxPort_PadDriver_cmosAutomotiveSpeed1);
    IfxPort_setPinModeInput(BTN_PORT, BTN_PIN, IfxPort_InputMode_pullUp);
    IfxPort_setPinHigh(LED_PORT, LED_PIN); /* 초기: OFF */
}

int core0_main(void)
{
    GPIO_init();

    while (1)
    {
        /* 버튼 눌림(LOW) → LED ON(LOW) */
        if (IfxPort_getPinState(BTN_PORT, BTN_PIN) == 0)
            IfxPort_setPinLow(LED_PORT, LED_PIN);
        else
            IfxPort_setPinHigh(LED_PORT, LED_PIN);
    }
    return 0;
}
```

---

## 7. 예제 — 버튼 디바운싱

기계식 버튼은 누르는 순간 수십 ms 동안 HIGH/LOW가 반복된다(채터링).  
소프트웨어 디바운싱으로 안정된 상태를 판별한다.

> LED = P0.5, 버튼 = P15.13

```c
#include "IfxPort.h"

#define LED_PORT      (&MODULE_P00)
#define LED_PIN       5
#define BTN_PORT      (&MODULE_P15)
#define BTN_PIN       13
#define DEBOUNCE_CNT  50000u   /* 루프 카운트 기준 약 5ms (코어 클럭에 따라 조정) */

typedef enum {
    BTN_IDLE,
    BTN_PRESSED,
    BTN_RELEASED
} BtnState;

BtnState GPIO_getButtonEvent(void)
{
    static uint8  last_stable  = 1;   /* pull-up → 기본 HIGH */
    static uint32 debounce_cnt = 0;
    uint8         current      = IfxPort_getPinState(BTN_PORT, BTN_PIN);

    if (current != last_stable)
    {
        debounce_cnt++;
        if (debounce_cnt >= DEBOUNCE_CNT)
        {
            debounce_cnt = 0;
            last_stable  = current;
            return (current == 0) ? BTN_PRESSED : BTN_RELEASED;
        }
    }
    else
    {
        debounce_cnt = 0;
    }
    return BTN_IDLE;
}

int core0_main(void)
{
    IfxPort_setPinModeOutput(LED_PORT, LED_PIN,
                             IfxPort_OutputMode_pushPull,
                             IfxPort_PadDriver_cmosAutomotiveSpeed1);
    IfxPort_setPinModeInput(BTN_PORT, BTN_PIN, IfxPort_InputMode_pullUp);
    IfxPort_setPinHigh(LED_PORT, LED_PIN);

    while (1)
    {
        if (GPIO_getButtonEvent() == BTN_PRESSED)
            IfxPort_togglePin(LED_PORT, LED_PIN); /* 눌릴 때만 토글 */
    }
    return 0;
}
```

> **디바운싱 카운트 조정**  
> `DEBOUNCE_CNT`는 루프 1회 실행 시간에 따라 다르다.  
> STM 타이머를 쓸 수 있다면 루프 카운트 대신 ms 단위로 측정하는 방식이 더 정확하다.  
> (Interrupt + STM 챕터에서 다룬다.)

---

## 8. 예제 — LED 여러 개 동시 제어 (setGroupState)

같은 포트의 LED 3개를 독립적으로 켜고 끄되,  
핀 상태가 어긋나는 순간이 없어야 하는 경우.

> LED1 = P0.5, LED2 = P0.6, LED3 = P0.7 (모두 active low)

```c
#include "IfxPort.h"

#define LED_PORT  (&MODULE_P00)
#define LED1_PIN  5
#define LED2_PIN  6
#define LED3_PIN  7
#define LED_MASK  ((1u << LED1_PIN) | (1u << LED2_PIN) | (1u << LED3_PIN))

void GPIO_init(void)
{
    const IfxPort_OutputMode mode    = IfxPort_OutputMode_pushPull;
    const IfxPort_PadDriver  driver  = IfxPort_PadDriver_cmosAutomotiveSpeed1;

    IfxPort_setPinModeOutput(LED_PORT, LED1_PIN, mode, driver);
    IfxPort_setPinModeOutput(LED_PORT, LED2_PIN, mode, driver);
    IfxPort_setPinModeOutput(LED_PORT, LED3_PIN, mode, driver);

    IfxPort_setGroupState(LED_PORT, LED_MASK, LED_MASK); /* 전부 OFF (active low) */
}

/* pattern: 비트 위치가 1이면 해당 LED ON */
void GPIO_setLEDs(uint32 pattern)
{
    /* active low이므로 ON하고 싶은 핀은 LOW = data에서 0 */
    IfxPort_setGroupState(LED_PORT, LED_MASK, LED_MASK & ~pattern);
}

int core0_main(void)
{
    GPIO_init();

    volatile uint32 i;
    while (1)
    {
        GPIO_setLEDs(1u << LED1_PIN);                              /* LED1만 ON */
        for (i = 0; i < 500000; i++);

        GPIO_setLEDs((1u << LED1_PIN) | (1u << LED2_PIN));        /* LED1, 2 ON */
        for (i = 0; i < 500000; i++);

        GPIO_setLEDs(LED_MASK);                                    /* 전부 ON */
        for (i = 0; i < 500000; i++);

        GPIO_setLEDs(0);                                           /* 전부 OFF */
        for (i = 0; i < 500000; i++);
    }
    return 0;
}
```

---

## 9. 예제 — 포트 전체 비트 한 번에 쓰기 (OUT 직접)

`IfxPort_setGroupState`는 OMR을 통해 변경할 핀만 골라서 쓰는 방식이다.  
반면 `OUT` 레지스터에 직접 쓰면 포트 전체 핀 상태를 한 번에 덮어쓸 수 있다.  
초기화 시점이나, 포트 전체를 알려진 상태로 리셋해야 할 때 유용하다.

> **주의**: OUT 직접 쓰기는 해당 포트의 **모든 출력 핀**을 동시에 덮어쓴다.  
> 일부 핀만 건드리고 싶다면 `setGroupState`(OMR 경유)를 써야 한다.

**레지스터 직접 접근**
```c
/* P0 포트 전체를 한 번에 0으로 클리어 */
P0_OUT.U = 0x00000000u;

/* P0.5, P0.6만 HIGH, 나머지 전부 LOW */
P0_OUT.U = (1u << 5) | (1u << 6);
```

**iLLD**

iLLD에는 OUT을 직접 덮어쓰는 전용 함수가 없다.  
포트 구조체 포인터로 SFR에 직접 접근하는 방식을 쓴다.

```c
#include "IfxPort.h"

/* IfxPort 구조체는 SFR 레이아웃과 동일하게 매핑된다 */
/* MODULE_P00.OUT.U 로 OUT 레지스터에 직접 접근 가능 */

/* P0 포트 전체 클리어 */
MODULE_P00.OUT.U = 0x00000000u;

/* P0.5, P0.6만 HIGH */
MODULE_P00.OUT.U = (1u << 5) | (1u << 6);
```

> `MODULE_P00`은 `Ifx_P` 구조체 타입으로,  
> `OUT`, `OMR`, `IOCR0` 등 PORT 레지스터 전체가 멤버로 들어있다.  
> iLLD 함수가 없어도 이 구조체를 통하면 레지스터를 직접 건드릴 수 있다.

**실제 사용 예 — 초기화 시 포트 전체 상태 설정**
```c
#include "IfxPort.h"

#define LED1_PIN  5
#define LED2_PIN  6
#define LED3_PIN  7

void GPIO_init(void)
{
    const IfxPort_OutputMode mode   = IfxPort_OutputMode_pushPull;
    const IfxPort_PadDriver  driver = IfxPort_PadDriver_cmosAutomotiveSpeed1;

    IfxPort_setPinModeOutput(&MODULE_P00, LED1_PIN, mode, driver);
    IfxPort_setPinModeOutput(&MODULE_P00, LED2_PIN, mode, driver);
    IfxPort_setPinModeOutput(&MODULE_P00, LED3_PIN, mode, driver);

    /* 방향 설정 후 OUT으로 초기 상태를 한 번에 확정 (active low → 전부 HIGH = OFF) */
    MODULE_P00.OUT.U |= (1u << LED1_PIN) | (1u << LED2_PIN) | (1u << LED3_PIN);
}
```

> `OUT.U =` (전체 덮어쓰기)와 `OUT.U |=` (해당 핀만 set)의 차이에 주의.  
> 포트에 출력 핀이 여러 종류 섞여 있다면 `|=` 또는 `setGroupState`가 안전하다.

---

## 10. 예제 — Active High / Active Low 래퍼



회로 구성에 따라 LED가 HIGH일 때 켜지는지, LOW일 때 켜지는지 달라진다.  
극성을 매 호출마다 주석으로 남기는 대신, 래퍼 함수 한 곳에서만 관리한다.

```
Active Low  : VCC → LED → 저항 → GPIO 핀  (핀 LOW = 전류 흐름 = ON)
Active High : GPIO 핀 → 저항 → LED → GND  (핀 HIGH = 전류 흐름 = ON)
```

```c
#include "IfxPort.h"

#define LED_PORT       (&MODULE_P00)
#define LED_PIN        5
#define LED_ACTIVE_LOW 1   /* 회로에 따라 0 또는 1 */

static inline void LED_on(void)
{
#if LED_ACTIVE_LOW
    IfxPort_setPinLow(LED_PORT, LED_PIN);
#else
    IfxPort_setPinHigh(LED_PORT, LED_PIN);
#endif
}

static inline void LED_off(void)
{
#if LED_ACTIVE_LOW
    IfxPort_setPinHigh(LED_PORT, LED_PIN);
#else
    IfxPort_setPinLow(LED_PORT, LED_PIN);
#endif
}

int core0_main(void)
{
    IfxPort_setPinModeOutput(LED_PORT, LED_PIN,
                             IfxPort_OutputMode_pushPull,
                             IfxPort_PadDriver_cmosAutomotiveSpeed1);
    LED_off();

    volatile uint32 i;
    while (1)
    {
        LED_on();
        for (i = 0; i < 1000000; i++);
        LED_off();
        for (i = 0; i < 1000000; i++);
    }
    return 0;
}
```

> 회로가 바뀌어 active high로 전환되더라도 `LED_ACTIVE_LOW` 한 줄만 고치면 된다.  
> `setPinHigh/Low`를 직접 호출하는 코드가 여기저기 퍼져 있으면  
> 수정해야 할 곳을 놓치기 쉽다.

---

## 정리

| 항목 | 레지스터 | iLLD |
|---|---|---|
| 출력 설정 | `IOCRx.PCy = 0x10` | `IfxPort_setPinModeOutput()` |
| 입력 설정 | `IOCRx.PCy = 0x00` | `IfxPort_setPinModeInput()` |
| 핀 HIGH | `OMR.PSx = 1` | `IfxPort_setPinHigh()` |
| 핀 LOW | `OMR.PCLx = 1` | `IfxPort_setPinLow()` |
| Toggle | `OUT` 읽고 `OMR` 쓰기 | `IfxPort_togglePin()` |
| 입력 읽기 | `IN.Px` | `IfxPort_getPinState()` |
| 여러 핀 동시 제어 | `OMR.U = (mask)` | `IfxPort_setGroupState()` |

iLLD를 쓴다고 레지스터 지식이 필요 없어지는 게 아니다.  
함수 이름만 봐도 어떤 레지스터를 건드리는지 연상할 수 있어야  
드라이버가 예상대로 동작하지 않을 때 빠르게 원인을 찾을 수 있다.

---

*다음 챕터: Interrupt / SRC — 인터럽트 라우팅 구조와 IfxSrc*
