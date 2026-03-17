# 05. UART — ASCLIN 구조와 IfxAsclin_Asc

## 목차

- [1. iLLD 계층 구조 — UART 관점](#1-illd-계층-구조--uart-관점)
- [2. ASCLIN 동작 원리](#2-asclin-동작-원리)
- [3. 핵심 레지스터 요약](#3-핵심-레지스터-요약)
- [4. iLLD API — IfxAsclin_Asc](#4-illd-api--ifxasclin_asc)
- [5. 레지스터 방식 vs iLLD 비교](#5-레지스터-방식-vs-illd-비교)
  - [5-1. 보드레이트 및 채널 설정](#5-1-보드레이트-및-채널-설정)
  - [5-2. 송신 / 수신](#5-2-송신--수신)
- [6. 예제 — 폴링 방식 송수신](#6-예제--폴링-방식-송수신)
- [7. 예제 — 인터럽트 + 링 버퍼 수신](#7-예제--인터럽트--링-버퍼-수신)
- [8. 예제 — printf 리다이렉트](#8-예제--printf-리다이렉트)
- [정리](#정리)

---

> ASCLIN(Asynchronous/Synchronous Interface)은 UART, SPI, LIN을 하나의 모듈로 지원한다.  
> 이 챕터에서는 **UART 모드(Asc)**만 다룬다.  
> SPI 모드는 다음 챕터(QSPI), LIN 모드는 별도 챕터에서 다룬다.

---

## 1. iLLD 계층 구조 — UART 관점

```
Your Code
    │
    ▼
[Hld]  IfxAsclin_Asc.h / IfxAsclin_Asc.c
       iLLD/TC3xx/Tricore/Asclin/Asc/
       (init, read, write, 인터럽트 핸들러 래핑)
    │
    ▼
[Lld]  IfxAsclin.h / IfxAsclin.c
       iLLD/TC3xx/Tricore/Asclin/Std/
       (보드레이트 계산, FIFO 제어, 레지스터 접근)
    │
    ▼
[SFR]  ASCLIN0_FRAMECON, BITCON, BRG, TXDATA, RXDATA ...
    │
    ▼
[HW]   TX/RX 핀
```

---

## 2. ASCLIN 동작 원리

```
송신(TX)
Your Code → write() → TX FIFO → Shift Register → TX 핀

수신(RX)
RX 핀 → Shift Register → RX FIFO → read() → Your Code
```

FIFO가 있기 때문에 CPU가 바이트마다 직접 대기하지 않아도 된다.  
FIFO가 차거나 비면 인터럽트로 알려주는 방식이 일반적이다.

**보드레이트 계산:**
```
보드레이트 = f_ASC / (prescaler × (oversampling + 1))
```
레지스터로 직접 계산하면 복잡하지만, iLLD는 목표 보드레이트를 넣으면  
BRG(Baudrate Generator) 레지스터 값을 자동으로 계산해준다.

---

## 3. 핵심 레지스터 요약

| 레지스터 | 역할 |
|---|---|
| `ASCLINx_FRAMECON` | 프레임 형식 (데이터 비트, 스톱 비트, 패리티) |
| `ASCLINx_BITCON` | 오버샘플링 설정 |
| `ASCLINx_BRG` | 보드레이트 분주비 |
| `ASCLINx_TXDATA` | 송신 데이터 쓰기 (→ TX FIFO) |
| `ASCLINx_RXDATA` | 수신 데이터 읽기 (← RX FIFO) |
| `ASCLINx_TXFIFOCON` | TX FIFO 레벨, flush 설정 |
| `ASCLINx_RXFIFOCON` | RX FIFO 레벨, flush 설정 |
| `ASCLINx_FLAGS` | 송수신 상태 플래그 (TX ready, RX available 등) |

---

## 4. iLLD API — IfxAsclin_Asc

파일 위치: `iLLD/TC3xx/Tricore/Asclin/Asc/IfxAsclin_Asc.h`

| 함수 | 동작 |
|---|---|
| `IfxAsclin_Asc_initModuleConfig(config, asclin)` | config 기본값 초기화 |
| `IfxAsclin_Asc_initModule(driver, config)` | ASCLIN 모듈 초기화 (보드레이트, 핀, FIFO) |
| `IfxAsclin_Asc_write(driver, data, count, timeout)` | TX FIFO에 데이터 쓰기 |
| `IfxAsclin_Asc_read(driver, data, count, timeout)` | RX FIFO에서 데이터 읽기 |
| `IfxAsclin_Asc_getWriteCount(driver)` | TX FIFO 여유 공간 |
| `IfxAsclin_Asc_getReadCount(driver)` | RX FIFO 수신 바이트 수 |
| `IfxAsclin_Asc_isrTransmit(driver)` | TX 인터럽트 핸들러 (ISR 안에서 호출) |
| `IfxAsclin_Asc_isrReceive(driver)` | RX 인터럽트 핸들러 (ISR 안에서 호출) |
| `IfxAsclin_Asc_isrError(driver)` | 에러 인터럽트 핸들러 |

---

## 5. 레지스터 방식 vs iLLD 비교

### 5-1. 보드레이트 및 채널 설정

**레지스터 직접 접근**
```c
/* ASCLIN0을 115200 bps, 8N1로 설정 (100MHz 기준) */
ASCLIN0_CLC.B.DISR   = 0u;     /* 모듈 클럭 enable */
ASCLIN0_FRAMECON.B.MODE = 1u;  /* ASC(UART) 모드 */
ASCLIN0_FRAMECON.B.STOP = 1u;  /* 스톱 비트 1 */
ASCLIN0_FRAMECON.B.PEN  = 0u;  /* 패리티 없음 */
ASCLIN0_DATCON.B.DATLEN = 7u;  /* 8비트 (7+1) */
ASCLIN0_BITCON.B.OVERSAMPLING = 15u;  /* 16× 오버샘플링 */
/* BRG 계산: 100MHz / (prescaler × 16) = 115200 → prescaler ≈ 54 */
ASCLIN0_BRG.B.NUMERATOR   = 3u;
ASCLIN0_BRG.B.DENOMINATOR = 1u;
ASCLIN0_IOCR.B.ALTI = ...; /* RX 핀 선택 */
/* TX 핀 라우팅 별도 처리 */
```

**iLLD**
```c
#include "IfxAsclin_Asc.h"

IfxAsclin_Asc_Config cfg;
IfxAsclin_Asc_Driver drv;

IfxAsclin_Asc_initModuleConfig(&cfg, &MODULE_ASCLIN0);

cfg.baudrate.baudrate    = 115200u;
cfg.baudrate.oversampling = IfxAsclin_OversamplingFactor_16;
cfg.frame.dataLength     = IfxAsclin_DataLength_8;
cfg.frame.stopBit        = IfxAsclin_StopBit_1;
cfg.frame.parityBit      = FALSE;
cfg.pins.tx = &IfxAsclin0_TX_P14_0_OUT;  /* TX 핀 자동 라우팅 */
cfg.pins.rx = &IfxAsclin0_RXA_P14_1_IN;  /* RX 핀 자동 라우팅 */

IfxAsclin_Asc_initModule(&drv, &cfg);
```

보드레이트 분주비 계산과 핀 라우팅이 모두 자동으로 처리된다.

---

### 5-2. 송신 / 수신

**레지스터 직접 접근**
```c
/* 1바이트 송신 */
while (ASCLIN0_TXFIFOCON.B.FILL >= 16u);  /* FIFO 공간 대기 */
ASCLIN0_TXDATA.U = 'A';

/* 1바이트 수신 */
while (ASCLIN0_RXFIFOCON.B.FILL == 0u);   /* 수신 대기 */
uint8 ch = (uint8)ASCLIN0_RXDATA.U;
```

**iLLD**
```c
uint8 tx_data = 'A';
IfxAsclin_Asc_write(&drv, &tx_data, 1, TIME_INFINITE);

uint8 rx_data;
IfxAsclin_Asc_read(&drv, &rx_data, 1, TIME_INFINITE);
```

---

## 6. 예제 — 폴링 방식 송수신

> ASCLIN0, 115200 bps, 8N1.  
> 수신한 바이트를 그대로 에코 송신한다.

```c
#include "IfxAsclin_Asc.h"

static IfxAsclin_Asc_Driver g_asc;

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

static void UART_send(const uint8 *data, uint32 len)
{
    IfxAsclin_Asc_write(&g_asc, data, (Ifx_SizeT *)&len, TIME_INFINITE);
}

static uint8 UART_receive(void)
{
    uint8 ch;
    Ifx_SizeT len = 1;
    IfxAsclin_Asc_read(&g_asc, &ch, &len, TIME_INFINITE);
    return ch;
}

int core0_main(void)
{
    UART_init();

    const uint8 hello[] = "UART ready\r\n";
    UART_send(hello, sizeof(hello) - 1u);

    while (1)
    {
        uint8 ch = UART_receive();  /* 수신 대기 (블로킹) */
        UART_send(&ch, 1u);         /* 에코 */
    }
    return 0;
}
```

---

## 7. 예제 — 인터럽트 + 링 버퍼 수신

폴링은 수신을 기다리는 동안 CPU가 블로킹된다.  
인터럽트 + 링 버퍼를 쓰면 데이터가 쌓이는 동안 CPU는 다른 작업을 할 수 있다.

```c
#include "IfxAsclin_Asc.h"
#include "IfxSrc.h"

#define RING_BUF_SIZE  64u
#define ISR_PRIO_RX    30u
#define ISR_PRIO_TX    31u
#define ISR_PRIO_ERR   32u

static IfxAsclin_Asc_Driver g_asc;

/* iLLD Asc Hld가 내부적으로 사용할 버퍼 — 링 버퍼 역할 */
static uint8 g_txBuf[RING_BUF_SIZE];
static uint8 g_rxBuf[RING_BUF_SIZE];

/* ISR — Hld의 핸들러 함수를 그대로 호출 */
IFX_INTERRUPT(ASCLIN0_TX_ISR,  0, ISR_PRIO_TX)  { IfxAsclin_Asc_isrTransmit(&g_asc); }
IFX_INTERRUPT(ASCLIN0_RX_ISR,  0, ISR_PRIO_RX)  { IfxAsclin_Asc_isrReceive(&g_asc);  }
IFX_INTERRUPT(ASCLIN0_ERR_ISR, 0, ISR_PRIO_ERR) { IfxAsclin_Asc_isrError(&g_asc);    }

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

    /* 인터럽트 우선순위 설정 */
    cfg.interrupt.txPriority  = ISR_PRIO_TX;
    cfg.interrupt.rxPriority  = ISR_PRIO_RX;
    cfg.interrupt.erPriority  = ISR_PRIO_ERR;
    cfg.interrupt.typeOfService = IfxSrc_Tos_cpu0;

    /* 링 버퍼 연결 */
    cfg.txBuffer     = g_txBuf;
    cfg.txBufferSize = RING_BUF_SIZE;
    cfg.rxBuffer     = g_rxBuf;
    cfg.rxBufferSize = RING_BUF_SIZE;

    IfxAsclin_Asc_initModule(&g_asc, &cfg);
}

int core0_main(void)
{
    UART_init();
    __enable();

    const uint8 msg[] = "Interrupt UART ready\r\n";
    Ifx_SizeT len = sizeof(msg) - 1u;
    IfxAsclin_Asc_write(&g_asc, msg, &len, TIME_INFINITE);

    while (1)
    {
        /* RX 버퍼에 데이터가 쌓이면 읽어서 에코 */
        if (IfxAsclin_Asc_getReadCount(&g_asc) > 0u)
        {
            uint8 ch;
            Ifx_SizeT one = 1;
            IfxAsclin_Asc_read(&g_asc, &ch, &one, TIME_INFINITE);
            IfxAsclin_Asc_write(&g_asc, &ch, &one, TIME_INFINITE);
        }
    }
    return 0;
}
```

> `IfxAsclin_Asc_isrReceive()` 안에서 RX FIFO → 링 버퍼 복사가 일어난다.  
> 메인 루프는 링 버퍼에서 데이터를 꺼내기만 하면 된다.

---

## 8. 예제 — printf 리다이렉트

디버그 출력에 `printf`를 그대로 쓰고 싶을 때,  
`IfxAsclin_Asc_write`를 `stdout` 출력 함수에 연결한다.

```c
#include "IfxAsclin_Asc.h"
#include <stdio.h>

static IfxAsclin_Asc_Driver g_asc;

/* newlib/GCC: stdout으로 나가는 write를 가로채는 함수 */
int _write(int fd, const char *buf, int count)
{
    (void)fd;
    Ifx_SizeT len = (Ifx_SizeT)count;
    IfxAsclin_Asc_write(&g_asc, (const uint8 *)buf, &len, TIME_INFINITE);
    return count;
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

int core0_main(void)
{
    UART_init();

    uint32 count = 0u;
    while (1)
    {
        printf("Hello AURIX! count = %lu\r\n", (unsigned long)count++);
        volatile uint32 i; for (i = 0; i < 2000000u; i++);
    }
    return 0;
}
```

> `_write`는 컴파일러(GCC/Tasking)에 따라 함수 이름이 다를 수 있다.  
> Tasking의 경우 `__putchar`나 `fdev_setup_stream`을 사용하는 방식을 확인할 것.

---

## 정리

| 항목 | 레지스터 | iLLD |
|---|---|---|
| 보드레이트 설정 | BRG 분주비 직접 계산 | `cfg.baudrate.baudrate` |
| 프레임 설정 | FRAMECON, DATCON 직접 | `cfg.frame.*` |
| 핀 라우팅 | IOCR 직접 | `cfg.pins.tx/rx` |
| 송신 | TXFIFO 공간 확인 후 TXDATA 쓰기 | `IfxAsclin_Asc_write()` |
| 수신 | RXFIFO FILL 확인 후 RXDATA 읽기 | `IfxAsclin_Asc_read()` |
| 인터럽트 연동 | SRC 직접 + ISR 직접 작성 | `cfg.interrupt.*` + `isrReceive/Transmit` |

---

*다음 챕터: SPI — QSPI 구조와 IfxQspi_SpiMaster*
