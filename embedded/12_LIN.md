# 12. LIN — ASCLIN LIN 모드와 IfxAsclin_Lin

## 목차

- [1. iLLD 계층 구조 — LIN 관점](#1-illd-계층-구조--lin-관점)
- [2. LIN 통신 기초 개념](#2-lin-통신-기초-개념)
- [3. ASCLIN LIN 모드 동작 원리](#3-asclin-lin-모드-동작-원리)
- [4. 핵심 레지스터 요약](#4-핵심-레지스터-요약)
- [5. iLLD API — IfxAsclin_Lin](#5-illd-api--ifxasclin_lin)
- [6. UART vs LIN 핵심 차이](#6-uart-vs-lin-핵심-차이)
- [7. 레지스터 방식 vs iLLD 비교](#7-레지스터-방식-vs-illd-비교)
  - [7-1. LIN 마스터 초기화](#7-1-lin-마스터-초기화)
  - [7-2. 프레임 송수신](#7-2-프레임-송수신)
- [8. 예제 — LIN 마스터 프레임 송신](#8-예제--lin-마스터-프레임-송신)
- [9. 예제 — LIN 마스터 프레임 수신 요청](#9-예제--lin-마스터-프레임-수신-요청)
- [10. 예제 — LIN 슬레이브 응답](#10-예제--lin-슬레이브-응답)
- [정리](#정리)

---

> LIN(Local Interconnect Network)은 차량 내부 저속 통신에 쓰이는 단선 직렬 버스다.  
> CAN보다 느리지만 배선이 단순해 시트 모터, 윈도우 스위치, 조명 제어 같은  
> 저가 노드에 적합하다.  
> AURIX의 ASCLIN 모듈은 UART/SPI뿐 아니라 **LIN 모드**도 지원한다.

---

## 1. iLLD 계층 구조 — LIN 관점

```
Your Code
    │
    ▼
[Hld]  IfxAsclin_Lin.h / IfxAsclin_Lin.c
       iLLD/TC3xx/Tricore/Asclin/Lin/
       (마스터/슬레이브 init, 프레임 송수신 래핑)
    │
    ▼
[Lld]  IfxAsclin.h / IfxAsclin.c
       iLLD/TC3xx/Tricore/Asclin/Std/
       (브레이크 필드, 체크섬, 보드레이트 설정)
    │
    ▼
[SFR]  ASCLIN_FRAMECON, LINCON, LINBTIMER, TXDATA ...
    │
    ▼
[HW]   LIN 단선 버스 (12V, LIN 트랜시버 필요)
```

UART와 동일하게 ASCLIN 모듈을 쓰지만,  
`IfxAsclin_Asc` 대신 **`IfxAsclin_Lin`** Hld를 사용한다.

---

## 2. LIN 통신 기초 개념

```
LIN 버스
        풀업저항(1kΩ)
             │
VBat(12V) ───┤
             │
LIN 선 ──────┴─── 마스터 ─── 슬레이브1 ─── 슬레이브2
              (단선, 모든 노드 공유)
```

**LIN 프레임 구조:**
```
Break Field │ Sync │ PID(1byte) │ Data(1~8byte) │ Checksum
```

| 필드 | 설명 |
|---|---|
| Break Field | 13비트 이상의 dominant — 프레임 시작 신호 |
| Sync | 0x55 — 보드레이트 동기화 |
| PID | Protected ID — Frame ID(6bit) + Parity(2bit) |
| Data | 페이로드 (1~8바이트) |
| Checksum | 데이터 무결성 검증 (Classic 또는 Enhanced) |

**마스터/슬레이브 구조:**
```
마스터가 모든 통신을 주도한다.
슬레이브는 마스터의 요청에만 응답한다.

마스터 → 슬레이브 (데이터 전송):
  마스터: [Break][Sync][PID][Data][Checksum]

마스터 ← 슬레이브 (데이터 요청):
  마스터: [Break][Sync][PID]
  슬레이브:                  [Data][Checksum]
```

**LIN 버전별 차이:**

| 항목 | LIN 1.x | LIN 2.x |
|---|---|---|
| 최대 속도 | 20 kbps | 20 kbps |
| 체크섬 | Classic (Data만) | Enhanced (PID 포함) |
| 슬레이브 수 | 최대 16개 | 최대 16개 |

---

## 3. ASCLIN LIN 모드 동작 원리

```
LIN 마스터 송신 흐름
    │
    ▼
Break Field 자동 생성 (ASCLIN이 처리)
    │
    ▼
Sync Byte(0x55) 자동 전송
    │
    ▼
PID 전송 (Frame ID + Parity 자동 계산)
    │
    ▼
Data 전송 (TX FIFO)
    │
    ▼
Checksum 자동 계산 및 전송
```

ASCLIN LIN 모드의 핵심은 **Break, Sync, Parity, Checksum을 하드웨어가 자동 처리**한다는 것이다.  
소프트웨어는 Frame ID와 Data만 설정하면 된다.

---

## 4. 핵심 레지스터 요약

| 레지스터 | 역할 |
|---|---|
| `ASCLINx_FRAMECON` | LIN 모드 선택, 데이터 길이 |
| `ASCLINx_LINCON` | 마스터/슬레이브 모드, 체크섬 타입 |
| `ASCLINx_LINBTIMER` | Break 필드 길이 설정 |
| `ASCLINx_LINSRSEL` | 슬레이브 응답 타임아웃 |
| `ASCLINx_BRG` | 보드레이트 분주비 |
| `ASCLINx_TXDATA` | 송신 데이터 (→ TX FIFO) |
| `ASCLINx_RXDATA` | 수신 데이터 (← RX FIFO) |
| `ASCLINx_FLAGS` | 송수신 완료, 에러 플래그 |

---

## 5. iLLD API — IfxAsclin_Lin

파일 위치: `iLLD/TC3xx/Tricore/Asclin/Lin/IfxAsclin_Lin.h`

| 함수 | 동작 |
|---|---|
| `IfxAsclin_Lin_initMasterConfig(config, asclin)` | 마스터 config 기본값 초기화 |
| `IfxAsclin_Lin_initMaster(driver, config)` | LIN 마스터 초기화 |
| `IfxAsclin_Lin_initSlaveConfig(config, asclin)` | 슬레이브 config 기본값 초기화 |
| `IfxAsclin_Lin_initSlave(driver, config)` | LIN 슬레이브 초기화 |
| `IfxAsclin_Lin_sendFrame(driver, header, data, len)` | 마스터: 프레임 헤더 + 데이터 송신 |
| `IfxAsclin_Lin_requestFrame(driver, header)` | 마스터: 슬레이브 응답 요청 |
| `IfxAsclin_Lin_readFrame(driver, data, len)` | 수신 데이터 읽기 |
| `IfxAsclin_Lin_isrTransmit(driver)` | TX 인터럽트 핸들러 |
| `IfxAsclin_Lin_isrReceive(driver)` | RX 인터럽트 핸들러 |
| `IfxAsclin_Lin_isrError(driver)` | 에러 인터럽트 핸들러 |

---

## 6. UART vs LIN 핵심 차이

| 항목 | UART (ASCLIN Asc) | LIN (ASCLIN Lin) |
|---|---|---|
| 신호선 | TX / RX (2선) | 단선 (1선) |
| 전압 레벨 | 3.3V / 5V | 12V (트랜시버 필요) |
| 통신 방식 | 비동기, 양방향 | 비동기, 마스터 주도 단방향 |
| 속도 | 최대 수 Mbps | 최대 20 kbps |
| 프레임 구조 | 없음 (원시 바이트) | Break / Sync / PID / Data / Checksum |
| 체크섬 | 없음 (상위 계층 처리) | 하드웨어 자동 생성/검증 |
| 주 용도 | 디버그, PC 통신 | 차량 저속 노드 (시트, 조명 등) |

---

## 7. 레지스터 방식 vs iLLD 비교

### 7-1. LIN 마스터 초기화

**레지스터 직접 접근**
```c
/* ASCLIN0 LIN 마스터 모드, 19200bps */
ASCLIN0_CLC.B.DISR        = 0u;
ASCLIN0_FRAMECON.B.MODE   = 3u;    /* LIN 모드 */
ASCLIN0_FRAMECON.B.STOP   = 1u;
ASCLIN0_LINCON.B.MS       = 1u;    /* 마스터 */
ASCLIN0_LINCON.B.CSEN     = 1u;    /* Enhanced 체크섬 */
ASCLIN0_LINBTIMER.B.BREAK = 13u;   /* Break 길이 13비트 */
/* BRG 분주비 계산 및 핀 라우팅 별도 설정 */
```

**iLLD**
```c
#include "IfxAsclin_Lin.h"

IfxAsclin_Lin_Master g_linMaster;

IfxAsclin_Lin_MasterConfig cfg;
IfxAsclin_Lin_initMasterConfig(&cfg, &MODULE_ASCLIN0);

cfg.baudrate.baudrate      = 19200u;
cfg.frame.checksumMode     = IfxAsclin_Lin_ChecksumMode_enhanced;
cfg.pins.tx                = &IfxAsclin0_TX_P14_0_OUT;
cfg.pins.rx                = &IfxAsclin0_RXA_P14_1_IN;

IfxAsclin_Lin_initMaster(&g_linMaster, &cfg);
```

Break 필드 생성, Sync 바이트 전송, Parity 계산이 모두 자동 처리된다.

---

### 7-2. 프레임 송수신

**레지스터 직접 접근**
```c
/* 마스터 → 슬레이브 데이터 전송 */
/* 1. Frame ID와 Parity 계산 */
uint8 pid = frameId | (parity_p0 << 6u) | (parity_p1 << 7u);

/* 2. LIN 헤더 전송 트리거 (Break + Sync + PID 자동 생성) */
ASCLIN0_LINCON.B.MASTER = 1u;
ASCLIN0_TXDATA.U = pid;

/* 3. 헤더 전송 완료 대기 후 데이터 전송 */
while (!ASCLIN0_FLAGS.B.TH);
for (uint8 i = 0; i < len; i++)
    ASCLIN0_TXDATA.U = data[i];

/* 4. 체크섬 자동 전송 완료 대기 */
while (!ASCLIN0_FLAGS.B.TC);
```

**iLLD**
```c
IfxAsclin_Lin_Frame header;
header.id  = 0x10u;   /* Frame ID */
header.dlc = 4u;      /* 데이터 길이 */

uint8 tx_data[4] = {0x01u, 0x02u, 0x03u, 0x04u};
IfxAsclin_Lin_sendFrame(&g_linMaster, &header, tx_data, 4u);
```

---

## 8. 예제 — LIN 마스터 프레임 송신

> ASCLIN0, LIN 마스터, 19200bps.  
> Frame ID=0x10에 4바이트 데이터를 100ms마다 송신.

```c
#include "IfxAsclin_Lin.h"
#include "IfxSrc.h"

#define ISR_PRIO_LIN_TX   30u
#define ISR_PRIO_LIN_RX   31u
#define ISR_PRIO_LIN_ERR  32u

static IfxAsclin_Lin_Master g_linMaster;

IFX_INTERRUPT(LIN_TX_ISR,  0, ISR_PRIO_LIN_TX)  { IfxAsclin_Lin_isrTransmit(&g_linMaster); }
IFX_INTERRUPT(LIN_RX_ISR,  0, ISR_PRIO_LIN_RX)  { IfxAsclin_Lin_isrReceive(&g_linMaster);  }
IFX_INTERRUPT(LIN_ERR_ISR, 0, ISR_PRIO_LIN_ERR) { IfxAsclin_Lin_isrError(&g_linMaster);    }

static void LIN_init(void)
{
    IfxAsclin_Lin_MasterConfig cfg;
    IfxAsclin_Lin_initMasterConfig(&cfg, &MODULE_ASCLIN0);

    cfg.baudrate.baudrate          = 19200u;
    cfg.baudrate.oversampling      = IfxAsclin_OversamplingFactor_16;
    cfg.frame.checksumMode         = IfxAsclin_Lin_ChecksumMode_enhanced;
    cfg.pins.tx                    = &IfxAsclin0_TX_P14_0_OUT;
    cfg.pins.rx                    = &IfxAsclin0_RXA_P14_1_IN;
    cfg.interrupt.txPriority       = ISR_PRIO_LIN_TX;
    cfg.interrupt.rxPriority       = ISR_PRIO_LIN_RX;
    cfg.interrupt.erPriority       = ISR_PRIO_LIN_ERR;
    cfg.interrupt.typeOfService    = IfxSrc_Tos_cpu0;

    IfxAsclin_Lin_initMaster(&g_linMaster, &cfg);
}

int core0_main(void)
{
    LIN_init();
    __enable();

    IfxAsclin_Lin_Frame header;
    header.id  = 0x10u;
    header.dlc = 4u;

    uint8    tx_data[4] = {0x00u, 0x00u, 0x00u, 0x00u};
    uint32   counter    = 0u;

    while (1)
    {
        tx_data[0] = (uint8)(counter >> 24u);
        tx_data[1] = (uint8)(counter >> 16u);
        tx_data[2] = (uint8)(counter >>  8u);
        tx_data[3] = (uint8)(counter);
        counter++;

        IfxAsclin_Lin_sendFrame(&g_linMaster, &header, tx_data, 4u);

        volatile uint32 i; for (i = 0; i < 10000000u; i++);  /* ~100ms */
    }
    return 0;
}
```

---

## 9. 예제 — LIN 마스터 프레임 수신 요청

> 마스터가 Frame ID=0x20으로 슬레이브에 데이터를 요청하고 응답을 수신.

```c
#include "IfxAsclin_Lin.h"

extern IfxAsclin_Lin_Master g_linMaster;  /* LIN_init()에서 초기화 */

static boolean LIN_requestData(uint8 frameId, uint8 *rxBuf, uint8 dlc)
{
    IfxAsclin_Lin_Frame header;
    header.id  = frameId;
    header.dlc = dlc;

    /* 헤더만 전송 → 슬레이브가 데이터를 응답 */
    IfxAsclin_Lin_requestFrame(&g_linMaster, &header);

    /* 응답 수신 대기 (타임아웃 처리 추가 권장) */
    volatile uint32 timeout = 100000u;
    while (IfxAsclin_Lin_getReadCount(&g_linMaster) < dlc)
    {
        if (--timeout == 0u) return FALSE;
    }

    Ifx_SizeT len = dlc;
    IfxAsclin_Lin_readFrame(&g_linMaster, rxBuf, &len);
    return TRUE;
}

int core0_main(void)
{
    LIN_init();
    __enable();

    uint8 rx_data[4];

    while (1)
    {
        if (LIN_requestData(0x20u, rx_data, 4u))
        {
            /* rx_data[0~3]: 슬레이브 응답 데이터 */
        }
        volatile uint32 i; for (i = 0; i < 10000000u; i++);
    }
    return 0;
}
```

---

## 10. 예제 — LIN 슬레이브 응답

> ASCLIN1을 LIN 슬레이브로 설정.  
> 마스터가 ID=0x20을 요청하면 4바이트 응답.

```c
#include "IfxAsclin_Lin.h"
#include "IfxSrc.h"

#define ISR_PRIO_SLAVE_TX   40u
#define ISR_PRIO_SLAVE_RX   41u
#define ISR_PRIO_SLAVE_ERR  42u

static IfxAsclin_Lin_Slave g_linSlave;
static uint8 g_slaveData[4] = {0xAAu, 0xBBu, 0xCCu, 0xDDu};

IFX_INTERRUPT(SLAVE_TX_ISR,  0, ISR_PRIO_SLAVE_TX)  { IfxAsclin_Lin_isrTransmit(&g_linSlave); }
IFX_INTERRUPT(SLAVE_RX_ISR,  0, ISR_PRIO_SLAVE_RX)  { IfxAsclin_Lin_isrReceive(&g_linSlave);  }
IFX_INTERRUPT(SLAVE_ERR_ISR, 0, ISR_PRIO_SLAVE_ERR) { IfxAsclin_Lin_isrError(&g_linSlave);    }

static void LIN_slaveInit(void)
{
    IfxAsclin_Lin_SlaveConfig cfg;
    IfxAsclin_Lin_initSlaveConfig(&cfg, &MODULE_ASCLIN1);

    cfg.baudrate.baudrate       = 19200u;
    cfg.frame.checksumMode      = IfxAsclin_Lin_ChecksumMode_enhanced;
    cfg.pins.tx                 = &IfxAsclin1_TX_P15_0_OUT;
    cfg.pins.rx                 = &IfxAsclin1_RXA_P15_1_IN;
    cfg.interrupt.txPriority    = ISR_PRIO_SLAVE_TX;
    cfg.interrupt.rxPriority    = ISR_PRIO_SLAVE_RX;
    cfg.interrupt.erPriority    = ISR_PRIO_SLAVE_ERR;
    cfg.interrupt.typeOfService = IfxSrc_Tos_cpu0;

    IfxAsclin_Lin_initSlave(&g_linSlave, &cfg);
}

int core0_main(void)
{
    LIN_slaveInit();
    __enable();

    while (1)
    {
        /* 마스터로부터 헤더 수신 확인 */
        if (IfxAsclin_Lin_isHeaderReceived(&g_linSlave))
        {
            uint8 received_id = IfxAsclin_Lin_getReceivedId(&g_linSlave);

            if (received_id == 0x20u)
            {
                /* ID=0x20 요청 → 4바이트 응답 */
                IfxAsclin_Lin_Frame header;
                header.id  = 0x20u;
                header.dlc = 4u;
                IfxAsclin_Lin_sendFrame(&g_linSlave, &header,
                                        g_slaveData, 4u);
            }
        }
    }
    return 0;
}
```

> LIN 슬레이브는 자동 보드레이트 동기화 기능이 있다.  
> Sync 바이트(0x55)를 수신해 마스터 클럭에 맞게 보드레이트를 자동 보정한다.  
> 따라서 슬레이브 측 보드레이트 설정이 약간 달라도 통신이 된다.

---

## 정리

| 항목 | 레지스터 | iLLD |
|---|---|---|
| 마스터 초기화 | FRAMECON, LINCON, LINBTIMER 직접 | `IfxAsclin_Lin_initMaster()` |
| 슬레이브 초기화 | 동일 + 슬레이브 모드 비트 | `IfxAsclin_Lin_initSlave()` |
| 보드레이트 설정 | BRG 분주비 계산 | `cfg.baudrate.baudrate` |
| 체크섬 타입 | `LINCON.B.CSEN` | `cfg.frame.checksumMode` |
| 핀 라우팅 | IOCR 직접 | `cfg.pins.tx/rx` |
| 프레임 송신 | PID Parity 계산 + TX FIFO | `IfxAsclin_Lin_sendFrame()` |
| 슬레이브 요청 | 헤더만 전송 후 RX 대기 | `IfxAsclin_Lin_requestFrame()` |
| 인터럽트 연동 | SRC 직접 + ISR 직접 | `cfg.interrupt.*` + `isrReceive/Transmit` |

LIN에서 가장 자주 실수하는 두 가지:  
첫째, **체크섬 타입(Classic vs Enhanced)** 이 마스터/슬레이브 간에 다르면 체크섬 에러가 발생한다. LIN 2.x 디바이스는 Enhanced, LIN 1.x는 Classic을 쓴다.  
둘째, **LIN 트랜시버** (TJA1027 등)가 없으면 버스에 신호가 나오지 않는다. ASCLIN TX 핀 → 트랜시버 → LIN 버스 순서로 연결해야 한다.
