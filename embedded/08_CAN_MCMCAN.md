# 08. CAN — MCMCAN 구조와 IfxCan_Can

## 목차

- [1. iLLD 계층 구조 — CAN 관점](#1-illd-계층-구조--can-관점)
- [2. CAN 통신 기초 개념](#2-can-통신-기초-개념)
- [3. MCMCAN 동작 원리](#3-mcmcan-동작-원리)
- [4. 핵심 레지스터 요약](#4-핵심-레지스터-요약)
- [5. iLLD API — IfxCan_Can](#5-illd-api--ifxcan_can)
- [6. 레지스터 방식 vs iLLD 비교](#6-레지스터-방식-vs-illd-비교)
  - [6-1. CAN 노드 초기화](#6-1-can-노드-초기화)
  - [6-2. 메시지 송신](#6-2-메시지-송신)
  - [6-3. 메시지 수신](#6-3-메시지-수신)
- [7. 예제 — CAN 메시지 송신](#7-예제--can-메시지-송신)
- [8. 예제 — CAN 메시지 수신 (폴링)](#8-예제--can-메시지-수신-폴링)
- [9. 예제 — CAN 수신 인터럽트](#9-예제--can-수신-인터럽트)
- [10. 예제 — CAN 루프백 테스트](#10-예제--can-루프백-테스트)
- [정리](#정리)

---

> CAN(Controller Area Network)은 차량 내부 ECU 간 통신에 쓰이는 표준 버스다.  
> AURIX TC3xx(2G)의 CAN 모듈은 **MCMCAN(Multi-Channel CAN)** 이라 불리며,  
> 여러 개의 독립 CAN 노드와 공유 메시지 RAM을 가진다.

---

## 1. iLLD 계층 구조 — CAN 관점

```
Your Code
    │
    ▼
[Hld]  IfxCan_Can.h / IfxCan_Can.c
       iLLD/TC3xx/Tricore/Can/Can/
       (노드 init, 메시지 객체 설정, 송수신 래핑)
    │
    ▼
[Lld]  IfxCan.h / IfxCan.c
       iLLD/TC3xx/Tricore/Can/Std/
       (비트타이밍, 메시지 RAM, 필터 설정)
    │
    ▼
[SFR]  CAN_N0CR, NBTP, TXBC, RXBC, IR ...
    │
    ▼
[HW]   CANTX / CANRX 핀 → CAN 트랜시버 → 버스
```

---

## 2. CAN 통신 기초 개념

```
CAN 버스
┌──────┐     ┌──────┐     ┌──────┐
│ ECU1 │─────│ ECU2 │─────│ ECU3 │
└──────┘     └──────┘     └──────┘
  노드         노드         노드

버스 신호: CAN_H / CAN_L (차동 신호)
종단 저항: 버스 양끝에 120Ω
```

**CAN 프레임 구조 (Classical CAN):**

```
SOF │ ID(11bit) │ RTR │ IDE │ DLC │ DATA(0~8byte) │ CRC │ ACK │ EOF
```

| 필드 | 설명 |
|---|---|
| ID | 메시지 식별자 (낮을수록 높은 우선순위) |
| DLC | 데이터 길이 (0~8 바이트) |
| DATA | 실제 페이로드 |
| RTR | Remote Transmission Request (데이터 요청) |

**중재(Arbitration):**  
여러 노드가 동시에 송신하면 ID가 낮은 메시지가 버스를 우선 차지한다.  
충돌 없이 자동으로 처리되는 것이 CAN의 핵심 특성이다.

**CAN FD vs Classical CAN:**

| 항목 | Classical CAN | CAN FD |
|---|---|---|
| 최대 페이로드 | 8 바이트 | 64 바이트 |
| 데이터 페이즈 속도 | 최대 1Mbps | 최대 8Mbps |
| ID | 11bit(Standard) / 29bit(Extended) | 동일 |

---

## 3. MCMCAN 동작 원리

```
MCMCAN 구조
├── CAN 모듈 (공유 메시지 RAM)
│   ├── Node 0  ─── TX Buffer → CANTX0 핀
│   │           ─── RX Buffer ← CANRX0 핀
│   │           └── 수신 필터 (ID 필터링)
│   ├── Node 1
│   └── Node N
```

**메시지 RAM:**  
MCMCAN은 메시지 RAM을 노드들이 공유한다.  
TX Buffer, RX Buffer, 필터 설정이 모두 이 RAM 안에 들어간다.  
iLLD는 `initNode()` 시점에 RAM 레이아웃을 자동으로 잡아준다.

**수신 필터:**  
수신하고 싶은 ID를 필터에 등록해두면,  
해당 ID의 메시지만 RX Buffer에 저장되고 나머지는 무시된다.

---

## 4. 핵심 레지스터 요약

| 레지스터 | 역할 |
|---|---|
| `CAN0_N0_CCCR` | 노드 설정 모드 진입 / 초기화 |
| `CAN0_N0_NBTP` | Nominal Bit Timing (보드레이트 설정) |
| `CAN0_N0_DBTP` | Data Bit Timing (CAN FD 데이터 페이즈 속도) |
| `CAN0_N0_TXBC` | TX Buffer 설정 (메시지 RAM 주소, 크기) |
| `CAN0_N0_RXBC` | RX Buffer 설정 |
| `CAN0_N0_GFC` | Global Filter Config (수신 필터 기본 동작) |
| `CAN0_N0_IR` | 인터럽트 플래그 (TX 완료, RX 준비 등) |
| `CAN0_N0_IE` | 인터럽트 enable |

---

## 5. iLLD API — IfxCan_Can

파일 위치: `iLLD/TC3xx/Tricore/Can/Can/IfxCan_Can.h`

| 함수 | 동작 |
|---|---|
| `IfxCan_Can_initModuleConfig(config, can)` | 모듈 config 기본값 초기화 |
| `IfxCan_Can_initModule(driver, config)` | CAN 모듈 초기화 |
| `IfxCan_Can_initNodeConfig(config, driver)` | 노드 config 기본값 초기화 |
| `IfxCan_Can_initNode(node, config)` | CAN 노드 초기화 (비트타이밍, 핀, 버퍼) |
| `IfxCan_Can_initMessage(msg)` | 메시지 구조체 기본값 초기화 |
| `IfxCan_Can_sendMessage(node, msg, data)` | 메시지 송신 |
| `IfxCan_Can_readMessage(node, msg, data)` | 수신 메시지 읽기 |
| `IfxCan_Can_isNewDataReceived(node, buffIdx)` | 수신 버퍼에 새 데이터 여부 |

---

## 6. 레지스터 방식 vs iLLD 비교

### 6-1. CAN 노드 초기화

CAN 비트타이밍 계산은 복잡하다 (Sync Seg, Prop Seg, Phase Seg, SJW).  
레지스터 직접 설정은 실수가 많고 오실로스코프 없이 검증이 어렵다.

**레지스터 직접 접근**
```c
/* Node 0 초기화 — 500kbps, 100MHz 기준 */
/* 1. 초기화 모드 진입 */
CAN0_N0_CCCR.B.INIT = 1u;
CAN0_N0_CCCR.B.CCE  = 1u;  /* 설정 변경 허용 */

/* 2. 비트타이밍 설정 (500kbps: BRP=4, TSEG1=15, TSEG2=4, SJW=4) */
CAN0_N0_NBTP.B.NBRP   = 4u;
CAN0_N0_NBTP.B.NTSEG1 = 15u;
CAN0_N0_NBTP.B.NTSEG2 = 4u;
CAN0_N0_NBTP.B.NSJW   = 4u;

/* 3. TX/RX 버퍼, 필터, 핀 라우팅 별도 설정 */
/* ... (생략, 수십 줄 필요) ... */

/* 4. 초기화 모드 해제 */
CAN0_N0_CCCR.B.INIT = 0u;
```

**iLLD**
```c
#include "IfxCan_Can.h"

IfxCan_Can        g_can;
IfxCan_Can_Node   g_canNode;

IfxCan_Can_Config canCfg;
IfxCan_Can_initModuleConfig(&canCfg, &MODULE_CAN0);
IfxCan_Can_initModule(&g_can, &canCfg);

IfxCan_Can_NodeConfig nodeCfg;
IfxCan_Can_initNodeConfig(&nodeCfg, &g_can);

nodeCfg.nodeId              = IfxCan_NodeId_0;
nodeCfg.clockSource         = IfxCan_ClockSource_both;
nodeCfg.baudRate.baudrate   = 500000u;       /* 500kbps */
nodeCfg.baudRate.samplePoint = 800u;          /* 80.0% 샘플 포인트 */
nodeCfg.pins.txPin          = &IfxCan_TXD00_P20_8_OUT;
nodeCfg.pins.rxPin          = &IfxCan_RXD00B_P20_7_IN;
nodeCfg.rxConfig.rxBufferDataFieldSize = IfxCan_DataFieldSize_8;
nodeCfg.txConfig.txBufferDataFieldSize = IfxCan_DataFieldSize_8;
nodeCfg.txConfig.dedicatedTxBuffersNumber = 2u;   /* TX 버퍼 2개 */

IfxCan_Can_initNode(&g_canNode, &nodeCfg);
```

비트타이밍 계산이 `baudrate` + `samplePoint` 두 값으로 추상화된다.

---

### 6-2. 메시지 송신

**레지스터 직접 접근**
```c
/* TX 버퍼 직접 채우기 */
/* 메시지 RAM 주소 계산 후 직접 쓰기 필요 */
uint32 *tx_buf = (uint32 *)(CAN_RAM_BASE + TX_OFFSET);
tx_buf[0] = (0x100u << 18u);        /* ID = 0x100 */
tx_buf[1] = (8u << 16u);            /* DLC = 8 */
tx_buf[2] = data_word0;
tx_buf[3] = data_word1;

/* TX 요청 */
CAN0_N0_TXBAR.U = (1u << buffer_idx);

/* 완료 대기 */
while (!(CAN0_N0_TXBTO.U & (1u << buffer_idx)));
```

**iLLD**
```c
IfxCan_Can_Message txMsg;
IfxCan_Can_initMessage(&txMsg);
txMsg.messageId     = 0x100u;
txMsg.messageIdLength = IfxCan_MessageIdLength_standard;
txMsg.dataLengthCode  = IfxCan_DataLengthCode_8;
txMsg.bufferNumber    = 0u;

uint32 tx_data[2] = {0x01020304u, 0x05060708u};
IfxCan_Can_sendMessage(&g_canNode, &txMsg, tx_data);
```

---

### 6-3. 메시지 수신

**레지스터 직접 접근**
```c
/* 수신 완료 플래그 확인 */
if (CAN0_N0_IR.B.DRX)
{
    /* RX 버퍼에서 직접 읽기 */
    uint32 *rx_buf = (uint32 *)(CAN_RAM_BASE + RX_OFFSET);
    uint32 id   = (rx_buf[0] >> 18u) & 0x7FFu;
    uint32 data0 = rx_buf[2];
    uint32 data1 = rx_buf[3];

    CAN0_N0_IR.B.DRX = 1u; /* 플래그 클리어 */
}
```

**iLLD**
```c
if (IfxCan_Can_isNewDataReceived(&g_canNode, 0u))
{
    IfxCan_Can_Message rxMsg;
    IfxCan_Can_initMessage(&rxMsg);
    rxMsg.bufferNumber = 0u;

    uint32 rx_data[2];
    IfxCan_Can_readMessage(&g_canNode, &rxMsg, rx_data);
    /* rxMsg.messageId, rx_data[0], rx_data[1] 사용 */
}
```

---

## 7. 예제 — CAN 메시지 송신

> CAN0 Node 0, 500kbps. 1초마다 ID=0x100, 8바이트 메시지 송신.

```c
#include "IfxCan_Can.h"
#include "IfxSrc.h"
#include "IfxStm.h"

static IfxCan_Can      g_can;
static IfxCan_Can_Node g_canNode;

static void CAN_init(void)
{
    IfxCan_Can_Config canCfg;
    IfxCan_Can_initModuleConfig(&canCfg, &MODULE_CAN0);
    IfxCan_Can_initModule(&g_can, &canCfg);

    IfxCan_Can_NodeConfig nodeCfg;
    IfxCan_Can_initNodeConfig(&nodeCfg, &g_can);
    nodeCfg.nodeId                            = IfxCan_NodeId_0;
    nodeCfg.clockSource                       = IfxCan_ClockSource_both;
    nodeCfg.baudRate.baudrate                 = 500000u;
    nodeCfg.baudRate.samplePoint              = 800u;
    nodeCfg.pins.txPin                        = &IfxCan_TXD00_P20_8_OUT;
    nodeCfg.pins.rxPin                        = &IfxCan_RXD00B_P20_7_IN;
    nodeCfg.rxConfig.rxBufferDataFieldSize    = IfxCan_DataFieldSize_8;
    nodeCfg.txConfig.txBufferDataFieldSize    = IfxCan_DataFieldSize_8;
    nodeCfg.txConfig.dedicatedTxBuffersNumber = 1u;
    IfxCan_Can_initNode(&g_canNode, &nodeCfg);
}

static void CAN_sendFrame(uint32 id, const uint32 *data, uint8 dlc)
{
    IfxCan_Can_Message msg;
    IfxCan_Can_initMessage(&msg);
    msg.messageId       = id;
    msg.messageIdLength = IfxCan_MessageIdLength_standard;
    msg.dataLengthCode  = (IfxCan_DataLengthCode)dlc;
    msg.bufferNumber    = 0u;
    IfxCan_Can_sendMessage(&g_canNode, &msg, (uint32 *)data);
}

int core0_main(void)
{
    CAN_init();

    uint32 tx_data[2] = {0xDEADBEEFu, 0xCAFEBABEu};
    uint32 counter    = 0u;

    while (1)
    {
        tx_data[0] = counter++;
        CAN_sendFrame(0x100u, tx_data, 8u);

        volatile uint32 i; for (i = 0; i < 100000000u; i++);  /* ~1s */
    }
    return 0;
}
```

---

## 8. 예제 — CAN 메시지 수신 (폴링)

> ID=0x200 메시지를 수신 필터에 등록, 폴링으로 읽기.

```c
#include "IfxCan_Can.h"

static IfxCan_Can      g_can;
static IfxCan_Can_Node g_canNode;

static void CAN_init(void)
{
    IfxCan_Can_Config canCfg;
    IfxCan_Can_initModuleConfig(&canCfg, &MODULE_CAN0);
    IfxCan_Can_initModule(&g_can, &canCfg);

    IfxCan_Can_NodeConfig nodeCfg;
    IfxCan_Can_initNodeConfig(&nodeCfg, &g_can);
    nodeCfg.nodeId                         = IfxCan_NodeId_0;
    nodeCfg.clockSource                    = IfxCan_ClockSource_both;
    nodeCfg.baudRate.baudrate              = 500000u;
    nodeCfg.baudRate.samplePoint           = 800u;
    nodeCfg.pins.txPin                     = &IfxCan_TXD00_P20_8_OUT;
    nodeCfg.pins.rxPin                     = &IfxCan_RXD00B_P20_7_IN;
    nodeCfg.rxConfig.rxBufferDataFieldSize = IfxCan_DataFieldSize_8;
    nodeCfg.txConfig.txBufferDataFieldSize = IfxCan_DataFieldSize_8;
    nodeCfg.txConfig.dedicatedTxBuffersNumber = 1u;

    /* 수신 필터: ID 0x200만 버퍼 0에 저장 */
    nodeCfg.filterConfig.messageIdLength = IfxCan_MessageIdLength_standard;
    nodeCfg.filterConfig.standardListSize = 1u;
    IfxCan_Can_initNode(&g_canNode, &nodeCfg);

    /* 필터 설정 (classic filter: ID=0x200, mask=0x7FF) */
    IfxCan_Filter filter;
    filter.number           = 0u;
    filter.elementConfiguration = IfxCan_FilterElementConfiguration_storeInRxBuffer;
    filter.id1              = 0x200u;
    filter.id2              = 0x7FFu;   /* 마스크: 모든 비트 일치 */
    filter.rxBufferOffset   = IfxCan_RxBufferOffset_0;
    IfxCan_Can_setStandardFilter(&g_canNode, &filter);
}

int core0_main(void)
{
    CAN_init();

    IfxCan_Can_Message rxMsg;
    uint32             rx_data[2];

    while (1)
    {
        if (IfxCan_Can_isNewDataReceived(&g_canNode, 0u))
        {
            IfxCan_Can_initMessage(&rxMsg);
            rxMsg.bufferNumber = 0u;
            IfxCan_Can_readMessage(&g_canNode, &rxMsg, rx_data);
            /* rxMsg.messageId == 0x200, rx_data[0~1] = 수신 데이터 */
        }
    }
    return 0;
}
```

---

## 9. 예제 — CAN 수신 인터럽트

> 수신 시마다 인터럽트로 처리, 메인 루프 블로킹 없음.

```c
#include "IfxCan_Can.h"
#include "IfxSrc.h"

#define ISR_PRIO_CAN_RX  40u

static IfxCan_Can      g_can;
static IfxCan_Can_Node g_canNode;
static uint32          g_rxData[2];
static uint32          g_rxId;
static boolean         g_rxFlag = FALSE;

IFX_INTERRUPT(CAN0_RX_ISR, 0, ISR_PRIO_CAN_RX)
{
    IfxCan_Can_Message rxMsg;
    IfxCan_Can_initMessage(&rxMsg);
    rxMsg.bufferNumber = 0u;
    IfxCan_Can_readMessage(&g_canNode, &rxMsg, g_rxData);
    g_rxId   = rxMsg.messageId;
    g_rxFlag = TRUE;

    /* 인터럽트 플래그 클리어 */
    IfxCan_Node_clearInterruptFlag(g_canNode.node, IfxCan_Interrupt_rxBufferNewMessage);
}

static void CAN_init(void)
{
    IfxCan_Can_Config canCfg;
    IfxCan_Can_initModuleConfig(&canCfg, &MODULE_CAN0);
    IfxCan_Can_initModule(&g_can, &canCfg);

    IfxCan_Can_NodeConfig nodeCfg;
    IfxCan_Can_initNodeConfig(&nodeCfg, &g_can);
    nodeCfg.nodeId                            = IfxCan_NodeId_0;
    nodeCfg.clockSource                       = IfxCan_ClockSource_both;
    nodeCfg.baudRate.baudrate                 = 500000u;
    nodeCfg.baudRate.samplePoint              = 800u;
    nodeCfg.pins.txPin                        = &IfxCan_TXD00_P20_8_OUT;
    nodeCfg.pins.rxPin                        = &IfxCan_RXD00B_P20_7_IN;
    nodeCfg.rxConfig.rxBufferDataFieldSize    = IfxCan_DataFieldSize_8;
    nodeCfg.txConfig.txBufferDataFieldSize    = IfxCan_DataFieldSize_8;
    nodeCfg.txConfig.dedicatedTxBuffersNumber = 1u;

    /* 수신 인터럽트 설정 */
    nodeCfg.interruptConfig.rxBufferNewMessageEnabled         = TRUE;
    nodeCfg.interruptConfig.rxf0NewMessageEnabled             = FALSE;
    nodeCfg.interruptConfig.reint.rxBufferNewMessage.priority = ISR_PRIO_CAN_RX;
    nodeCfg.interruptConfig.reint.rxBufferNewMessage.typeOfService = IfxSrc_Tos_cpu0;

    IfxCan_Can_initNode(&g_canNode, &nodeCfg);
}

int core0_main(void)
{
    CAN_init();
    __enable();

    while (1)
    {
        if (g_rxFlag)
        {
            g_rxFlag = FALSE;
            /* g_rxId, g_rxData 처리 */
        }
    }
    return 0;
}
```

---

## 10. 예제 — CAN 루프백 테스트

외부 CAN 버스나 트랜시버 없이 송수신 로직을 검증할 때 루프백 모드를 쓴다.  
내부적으로 TX → RX가 연결되어 전송한 메시지를 즉시 수신할 수 있다.

```c
#include "IfxCan_Can.h"

static IfxCan_Can      g_can;
static IfxCan_Can_Node g_canNode;

static void CAN_initLoopback(void)
{
    IfxCan_Can_Config canCfg;
    IfxCan_Can_initModuleConfig(&canCfg, &MODULE_CAN0);
    IfxCan_Can_initModule(&g_can, &canCfg);

    IfxCan_Can_NodeConfig nodeCfg;
    IfxCan_Can_initNodeConfig(&nodeCfg, &g_can);
    nodeCfg.nodeId                            = IfxCan_NodeId_0;
    nodeCfg.clockSource                       = IfxCan_ClockSource_both;
    nodeCfg.baudRate.baudrate                 = 500000u;
    nodeCfg.baudRate.samplePoint              = 800u;
    nodeCfg.loopBackMode                      = TRUE;   /* 루프백 활성화 */
    nodeCfg.rxConfig.rxBufferDataFieldSize    = IfxCan_DataFieldSize_8;
    nodeCfg.txConfig.txBufferDataFieldSize    = IfxCan_DataFieldSize_8;
    nodeCfg.txConfig.dedicatedTxBuffersNumber = 1u;
    /* 루프백: 핀 라우팅 불필요 */
    IfxCan_Can_initNode(&g_canNode, &nodeCfg);
}

int core0_main(void)
{
    CAN_initLoopback();

    uint32 tx_data[2] = {0x11223344u, 0x55667788u};
    uint32 rx_data[2] = {0u, 0u};

    /* 송신 */
    IfxCan_Can_Message txMsg;
    IfxCan_Can_initMessage(&txMsg);
    txMsg.messageId      = 0x123u;
    txMsg.dataLengthCode = IfxCan_DataLengthCode_8;
    txMsg.bufferNumber   = 0u;
    IfxCan_Can_sendMessage(&g_canNode, &txMsg, tx_data);

    /* 수신 확인 (루프백이므로 바로 수신됨) */
    while (!IfxCan_Can_isNewDataReceived(&g_canNode, 0u));

    IfxCan_Can_Message rxMsg;
    IfxCan_Can_initMessage(&rxMsg);
    rxMsg.bufferNumber = 0u;
    IfxCan_Can_readMessage(&g_canNode, &rxMsg, rx_data);

    /* tx_data == rx_data 이면 정상 */
    while (1) {}
    return 0;
}
```

---

## 정리

| 항목 | 레지스터 | iLLD |
|---|---|---|
| 모듈 초기화 | CLC, 메시지 RAM 직접 설정 | `IfxCan_Can_initModule()` |
| 노드 초기화 | CCCR, NBTP, TXBC, RXBC 직접 | `IfxCan_Can_initNode()` |
| 비트타이밍 | BRP, TSEG1, TSEG2, SJW 계산 | `nodeCfg.baudRate.baudrate/samplePoint` |
| 핀 라우팅 | IOCR 직접 | `nodeCfg.pins.txPin/rxPin` |
| 메시지 송신 | 메시지 RAM 직접 쓰기 + TXBAR | `IfxCan_Can_sendMessage()` |
| 메시지 수신 | IR 플래그 + 메시지 RAM 직접 읽기 | `IfxCan_Can_readMessage()` |
| 수신 확인 | `IR.DRX` 폴링 | `IfxCan_Can_isNewDataReceived()` |
| 수신 필터 | `GFC`, `SIDFC` 직접 설정 | `IfxCan_Can_setStandardFilter()` |

CAN은 비트타이밍 계산이 복잡해서 iLLD의 이점이 가장 두드러지는 peripheral이다.  
`samplePoint`(샘플 포인트, 단위 0.1%)를 800으로 설정하면 80.0%로,  
대부분의 CAN 시스템에서 안정적으로 동작하는 값이다.  
버스 길이나 트랜시버 특성에 따라 미세 조정이 필요할 수 있다.
