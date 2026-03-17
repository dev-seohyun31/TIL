# 06. SPI — QSPI 구조와 IfxQspi_SpiMaster

## 목차

- [1. iLLD 계층 구조 — SPI 관점](#1-illd-계층-구조--spi-관점)
- [2. QSPI 동작 원리](#2-qspi-동작-원리)
- [3. 핵심 레지스터 요약](#3-핵심-레지스터-요약)
- [4. iLLD API — IfxQspi_SpiMaster](#4-illd-api--ifxqspi_spimaster)
- [5. UART vs SPI 핵심 차이](#5-uart-vs-spi-핵심-차이)
- [6. 레지스터 방식 vs iLLD 비교](#6-레지스터-방식-vs-illd-비교)
  - [6-1. QSPI 마스터 초기화](#6-1-qspi-마스터-초기화)
  - [6-2. 데이터 송수신](#6-2-데이터-송수신)
- [7. 예제 — SPI 마스터 단방향 송신](#7-예제--spi-마스터-단방향-송신)
- [8. 예제 — SPI 풀듀플렉스 송수신](#8-예제--spi-풀듀플렉스-송수신)
- [9. 예제 — CS 핀 수동 제어](#9-예제--cs-핀-수동-제어)
- [정리](#정리)

---

> QSPI(Queued SPI)는 AURIX의 SPI 전용 모듈이다.  
> ASCLIN의 SPI 모드와 달리, DMA 연동과 큐(Queue) 기반 전송을 지원해  
> 고속/대용량 데이터 전송에 적합하다.

---

## 1. iLLD 계층 구조 — SPI 관점

```
Your Code
    │
    ▼
[Hld]  IfxQspi_SpiMaster.h / IfxQspi_SpiMaster.c
       iLLD/TC3xx/Tricore/Qspi/SpiMaster/
       (채널 init, exchange, CS 제어 래핑)
    │
    ▼
[Lld]  IfxQspi.h / IfxQspi.c
       iLLD/TC3xx/Tricore/Qspi/Std/
       (FIFO, 클럭 설정, 레지스터 접근)
    │
    ▼
[SFR]  QSPI0_GLOBALCON, ECON, BACON, TXFIFO, RXFIFO ...
    │
    ▼
[HW]   SCLK / MOSI / MISO / CS 핀
```

---

## 2. QSPI 동작 원리

```
SPI 기본 신호
├── SCLK  : 마스터가 생성하는 클럭
├── MOSI  : 마스터 → 슬레이브 데이터
├── MISO  : 슬레이브 → 마스터 데이터
└── CS    : 슬레이브 선택 (active low)

전송 흐름
TX FIFO → Shift Register → MOSI 핀
MISO 핀 → Shift Register → RX FIFO
```

SPI는 **전이중(Full-Duplex)** 이다.  
송신과 수신이 동시에 일어나므로, 수신 데이터가 필요 없더라도  
TX에 더미 바이트를 보내야 SCLK가 생성되어 MISO 데이터를 받아올 수 있다.

**CPOL / CPHA 모드:**

| 모드 | CPOL | CPHA | 설명 |
|---|---|---|---|
| 0 | 0 | 0 | 클럭 평시 LOW, 상승 에지 샘플 |
| 1 | 0 | 1 | 클럭 평시 LOW, 하강 에지 샘플 |
| 2 | 1 | 0 | 클럭 평시 HIGH, 하강 에지 샘플 |
| 3 | 1 | 1 | 클럭 평시 HIGH, 상승 에지 샘플 |

슬레이브 디바이스의 데이터시트에서 지원하는 모드를 확인해야 한다.

---

## 3. 핵심 레지스터 요약

| 레지스터 | 역할 |
|---|---|
| `QSPIx_GLOBALCON` | 모듈 클럭, 마스터/슬레이브 모드, 전역 설정 |
| `QSPIx_ECON0~7` | 채널별 클럭 분주, CPOL/CPHA 설정 |
| `QSPIx_BACON` | 전송 시 적용할 설정 (데이터 길이, CS, 채널) |
| `QSPIx_DATAENTRY` | TX FIFO 쓰기 (데이터 + BACON 동시) |
| `QSPIx_RXEXIT` | RX FIFO 읽기 |
| `QSPIx_STATUS` | FIFO 상태, 에러 플래그 |

---

## 4. iLLD API — IfxQspi_SpiMaster

파일 위치: `iLLD/TC3xx/Tricore/Qspi/SpiMaster/IfxQspi_SpiMaster.h`

| 함수 | 동작 |
|---|---|
| `IfxQspi_SpiMaster_initModuleConfig(config, qspi)` | 모듈 config 기본값 초기화 |
| `IfxQspi_SpiMaster_initModule(master, config)` | QSPI 마스터 초기화 |
| `IfxQspi_SpiMaster_initChannelConfig(chConfig, master)` | 채널 config 기본값 초기화 |
| `IfxQspi_SpiMaster_initChannel(channel, chConfig)` | SPI 채널(CS 단위) 초기화 |
| `IfxQspi_SpiMaster_exchange(channel, tx, rx, count)` | 풀듀플렉스 송수신 |
| `IfxQspi_SpiMaster_getStatus(channel)` | 전송 완료 여부 확인 |

채널 = CS 핀 하나에 해당한다.  
슬레이브 디바이스가 여러 개면 채널을 여러 개 만들어 각각 CS를 배정한다.

---

## 5. UART vs SPI 핵심 차이

| 항목 | UART (ASCLIN) | SPI (QSPI) |
|---|---|---|
| 클럭 | 비동기 (보드레이트 양쪽 맞춤) | 동기 (마스터가 SCLK 생성) |
| 방향 | 반이중 가능 | 항상 풀듀플렉스 |
| CS 신호 | 없음 | 있음 (슬레이브 선택) |
| 속도 | 최대 수 Mbps | 최대 수십 Mbps |
| 주 용도 | PC 통신, 디버그, LIN | 센서, 플래시, DAC/ADC |

---

## 6. 레지스터 방식 vs iLLD 비교

### 6-1. QSPI 마스터 초기화

**레지스터 직접 접근**
```c
/* 1. 모듈 클럭 enable */
QSPI0_CLC.B.DISR = 0u;

/* 2. 마스터 모드, 클럭 설정 */
QSPI0_GLOBALCON.B.MS    = 0u;    /* 마스터 */
QSPI0_GLOBALCON.B.CLKSEL = 0u;

/* 3. 채널 0: 1MHz, Mode 0 (CPOL=0, CPHA=0) */
QSPI0_ECON0.B.Q    = 49u;  /* 분주비: 100MHz / (2*(49+1)) = 1MHz */
QSPI0_ECON0.B.CPOL = 0u;
QSPI0_ECON0.B.CPHA = 0u;

/* 4. 핀 라우팅 (SCLK, MOSI, MISO, CS) 별도 설정 */
```

**iLLD**
```c
#include "IfxQspi_SpiMaster.h"

IfxQspi_SpiMaster         g_spiMaster;
IfxQspi_SpiMaster_Channel g_spiChannel;

IfxQspi_SpiMaster_Config masterCfg;
IfxQspi_SpiMaster_initModuleConfig(&masterCfg, &MODULE_QSPI0);

masterCfg.base.mode             = SpiIf_Mode_master;
masterCfg.base.maximumBaudrate  = 1000000u;  /* 1MHz */
masterCfg.pins.sclk = &IfxQspi0_SCLK_P20_11_OUT;
masterCfg.pins.mtsr = &IfxQspi0_MTSR_P20_14_OUT;  /* MOSI */
masterCfg.pins.mrst = &IfxQspi0_MRST_P20_12_IN;   /* MISO */

IfxQspi_SpiMaster_initModule(&g_spiMaster, &masterCfg);

/* 채널(CS) 설정 */
IfxQspi_SpiMaster_ChannelConfig chCfg;
IfxQspi_SpiMaster_initChannelConfig(&chCfg, &g_spiMaster);

chCfg.base.baudrate = 1000000u;
chCfg.base.mode.cpol = SpiIf_ClockPolarity_idleLow;
chCfg.base.mode.cpha = SpiIf_DataShiftEdge_lagging;
chCfg.spiMasterChannel = IfxQspi_Channel_0;
chCfg.base.csActiveState = SpiIf_ActiveState_low;
chCfg.pins.cs = &IfxQspi0_SLSO0_P20_6_OUT;  /* CS 핀 */

IfxQspi_SpiMaster_initChannel(&g_spiChannel, &chCfg);
```

---

### 6-2. 데이터 송수신

**레지스터 직접 접근**
```c
/* CS LOW */
/* BACON 설정 후 DATAENTRY에 쓰기 */
QSPI0_BACON.U = ...;
QSPI0_DATAENTRY0.U = tx_byte;

/* TX 완료 대기 */
while (QSPI0_STATUS.B.TXFIFOLEVEL > 0u);

/* RX FIFO 읽기 */
uint8 rx_byte = (uint8)QSPI0_RXEXIT.U;
/* CS HIGH */
```

**iLLD**
```c
uint8 tx_buf[4] = {0x01, 0x02, 0x03, 0x04};
uint8 rx_buf[4] = {0};

IfxQspi_SpiMaster_exchange(&g_spiChannel, tx_buf, rx_buf, 4u);

/* 전송 완료 대기 */
while (IfxQspi_SpiMaster_getStatus(&g_spiChannel) == SpiIf_Status_busy);
```

CS 토글, BACON 설정, FIFO 관리가 모두 내부에서 처리된다.

---

## 7. 예제 — SPI 마스터 단방향 송신

> QSPI0, 1MHz, Mode 0.  
> 4바이트 데이터를 주기적으로 슬레이브에 송신.

```c
#include "IfxQspi_SpiMaster.h"

static IfxQspi_SpiMaster         g_spiMaster;
static IfxQspi_SpiMaster_Channel g_spiChannel;

static void SPI_init(void)
{
    IfxQspi_SpiMaster_Config masterCfg;
    IfxQspi_SpiMaster_initModuleConfig(&masterCfg, &MODULE_QSPI0);
    masterCfg.base.mode            = SpiIf_Mode_master;
    masterCfg.base.maximumBaudrate = 1000000u;
    masterCfg.pins.sclk = &IfxQspi0_SCLK_P20_11_OUT;
    masterCfg.pins.mtsr = &IfxQspi0_MTSR_P20_14_OUT;
    masterCfg.pins.mrst = &IfxQspi0_MRST_P20_12_IN;
    IfxQspi_SpiMaster_initModule(&g_spiMaster, &masterCfg);

    IfxQspi_SpiMaster_ChannelConfig chCfg;
    IfxQspi_SpiMaster_initChannelConfig(&chCfg, &g_spiMaster);
    chCfg.base.baudrate          = 1000000u;
    chCfg.base.mode.cpol         = SpiIf_ClockPolarity_idleLow;
    chCfg.base.mode.cpha         = SpiIf_DataShiftEdge_lagging;
    chCfg.spiMasterChannel       = IfxQspi_Channel_0;
    chCfg.base.csActiveState     = SpiIf_ActiveState_low;
    chCfg.pins.cs                = &IfxQspi0_SLSO0_P20_6_OUT;
    IfxQspi_SpiMaster_initChannel(&g_spiChannel, &chCfg);
}

int core0_main(void)
{
    SPI_init();

    uint8 tx_data[4] = {0xDE, 0xAD, 0xBE, 0xEF};

    while (1)
    {
        IfxQspi_SpiMaster_exchange(&g_spiChannel, tx_data, NULL_PTR, 4u);
        while (IfxQspi_SpiMaster_getStatus(&g_spiChannel) == SpiIf_Status_busy);

        volatile uint32 i; for (i = 0; i < 1000000u; i++);
    }
    return 0;
}
```

---

## 8. 예제 — SPI 풀듀플렉스 송수신

> 레지스터 주소를 TX로 보내고, 슬레이브 응답을 RX로 받는 패턴.  
> (예: SPI 센서 레지스터 읽기)

```c
#include "IfxQspi_SpiMaster.h"

extern IfxQspi_SpiMaster_Channel g_spiChannel;  /* SPI_init()에서 초기화 */

/* SPI 레지스터 1바이트 읽기 */
uint8 SPI_readRegister(uint8 reg_addr)
{
    /* SPI 센서: 첫 바이트 = 레지스터 주소(MSB=1: read),
                 두 번째 바이트 = 더미 (슬레이브 응답 수신용) */
    uint8 tx_buf[2] = {reg_addr | 0x80u, 0x00u};
    uint8 rx_buf[2] = {0x00u, 0x00u};

    IfxQspi_SpiMaster_exchange(&g_spiChannel, tx_buf, rx_buf, 2u);
    while (IfxQspi_SpiMaster_getStatus(&g_spiChannel) == SpiIf_Status_busy);

    return rx_buf[1];  /* 첫 바이트는 더미, 두 번째가 실제 데이터 */
}

/* SPI 레지스터 1바이트 쓰기 */
void SPI_writeRegister(uint8 reg_addr, uint8 value)
{
    uint8 tx_buf[2] = {reg_addr & 0x7Fu, value};  /* MSB=0: write */

    IfxQspi_SpiMaster_exchange(&g_spiChannel, tx_buf, NULL_PTR, 2u);
    while (IfxQspi_SpiMaster_getStatus(&g_spiChannel) == SpiIf_Status_busy);
}
```

---

## 9. 예제 — CS 핀 수동 제어

iLLD는 `exchange()` 호출 시 CS를 자동으로 토글한다.  
여러 바이트 전송 사이에 CS를 유지해야 하는 슬레이브라면  
CS 핀을 GPIO로 직접 제어해야 한다.

```c
#include "IfxQspi_SpiMaster.h"
#include "IfxPort.h"

#define CS_PORT  (&MODULE_P20)
#define CS_PIN   6u

static void CS_low(void)  { IfxPort_setPinLow(CS_PORT, CS_PIN);  }
static void CS_high(void) { IfxPort_setPinHigh(CS_PORT, CS_PIN); }

static void SPI_init_manualCS(void)
{
    /* CS 핀을 GPIO 출력으로 설정 (chCfg.pins.cs는 NULL_PTR) */
    IfxPort_setPinModeOutput(CS_PORT, CS_PIN,
                             IfxPort_OutputMode_pushPull,
                             IfxPort_PadDriver_cmosAutomotiveSpeed1);
    CS_high();  /* 초기: 비선택 */

    /* 나머지 SPI 설정은 동일 ... */
}

void SPI_transferBurst(const uint8 *data, uint32 len)
{
    CS_low();

    /* 여러 exchange 사이에 CS가 유지됨 */
    for (uint32 i = 0u; i < len; i++)
    {
        IfxQspi_SpiMaster_exchange(&g_spiChannel, &data[i], NULL_PTR, 1u);
        while (IfxQspi_SpiMaster_getStatus(&g_spiChannel) == SpiIf_Status_busy);
    }

    CS_high();
}
```

---

## 정리

| 항목 | 레지스터 | iLLD |
|---|---|---|
| 모듈 초기화 | GLOBALCON, CLC 직접 | `IfxQspi_SpiMaster_initModule()` |
| 채널(CS) 설정 | ECON, BACON 직접 | `IfxQspi_SpiMaster_initChannel()` |
| 클럭/모드 설정 | ECON.Q, CPOL, CPHA | `chCfg.base.baudrate/mode` |
| 핀 라우팅 | IOCR 직접 | `masterCfg.pins.sclk/mtsr/mrst` |
| 송수신 | DATAENTRY 쓰기 + RXEXIT 읽기 | `IfxQspi_SpiMaster_exchange()` |
| 완료 확인 | STATUS.TXFIFOLEVEL 폴링 | `IfxQspi_SpiMaster_getStatus()` |

SPI는 TX/RX가 항상 동시에 일어난다.  
수신이 필요 없더라도 `rx`에 `NULL_PTR`을 넣으면  
iLLD가 내부적으로 더미 수신 처리를 해준다.

---

*다음 챕터: ADC — EVADC 구조와 IfxEvadc_Adc*
