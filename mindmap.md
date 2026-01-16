# TODO 
1. 아래 주제별 내용정리
2. 내가 순수하게 이해한 바만 글로 쓰기 (what why when where how도 괜찮고, 개념도 괜찮음)

# MCU 학습 마인드맵
```
MCU
├─ CPU (Core)
│  ├─ Instruction Set (RISC / CISC)
│  ├─ Pipeline
│  ├─ Interrupt 처리
│  └─ Privilege Mode (User / Supervisor)
│
├─ Memory
│  ├─ Flash (Program)
│  ├─ RAM (Data, Stack, Heap)
│  ├─ ROM (Boot)
│  └─ Memory Map
│
├─ Clock & Reset
│  ├─ Internal Oscillator
│  ├─ External Crystal
│  ├─ PLL
│  └─ Reset Source (POR, WDT, SW Reset)
│
├─ Peripheral (**)
│  ├─ GPIO
│  ├─ Timer / PWM
│  ├─ ADC / DAC
│  ├─ UART / SPI / I2C
│  └─ CAN / LIN / Ethernet (고급 MCU)
│
├─ Interrupt System (**)
│  ├─ IRQ Vector Table
│  ├─ Priority
│  └─ Preemption
│
├─ Power Management
│  ├─ Sleep
│  ├─ Standby
│  └─ Low Power Mode
│
├─ Debug
│  ├─ JTAG / SWD
│  ├─ Breakpoint
│  └─ Trace
│
└─ Toolchain
   ├─ Compiler
   ├─ Linker
   ├─ Startup Code
   └─ Linker Script
```

# Aurix 학습 마인드맵
```
AURIX (TC3xx 기준)
├─ CPU Architecture
│  ├─ TriCore (CPU0, CPU1, CPU2)
│  ├─ Lockstep Core (Safety)
│  └─ Harvard Architecture
│
├─ Safety (ISO 26262)
│  ├─ Lockstep 비교
│  ├─ ECC (Flash / RAM)
│  ├─ SMU (Safety Management Unit)
│  └─ Watchdog (HW / SW)
│
├─ Memory System
│  ├─ PFlash (Program)
│  ├─ DFlash (Data)
│  ├─ Local RAM (Core별)
│  └─ LMU (Shared RAM)
│
├─ Bus Architecture
│  ├─ SRI (System Resource Interconnect)
│  ├─ Crossbar
│  └─ Peripheral Bus
│
├─ IO & Peripheral (Automotive) (**)
│  ├─ GPT12 / GTM (고급 타이머)
│  ├─ EVADC (고속 ADC)
│  ├─ CAN / CAN-FD
│  ├─ FlexRay
│  └─ Ethernet 
│
├─ Startup & Boot
│  ├─ BootROM
│  ├─ CPU0 먼저 기동
│  ├─ CPU1/2 동기화
│  └─ User Startup Code
│
├─ Interrupt & Trap (**)
│  ├─ Interrupt (IRQ)
│  ├─ Trap (Exception)
│  └─ Priority & Core Routing
│
├─ Tool & Framework (**) - MCAL 제외
│  ├─ iLLD
│  ├─ AURIX Development Studio
│  └─ MCAL (AUTOSAR)
│
└─ Automotive OS (선택)
   ├─ Bare-metal
   ├─ AUTOSAR Classic
   └─ RTOS
```
# 물리/전자 학습 마인드맵
```
물리 / 전자 (임베디드 필수) (**전부)
├─ 전기 기초 (Physics Level)
│  ├─ 전압 (Voltage)
│  ├─ 전류 (Current)
│  ├─ 저항 (Resistance)
│  ├─ 전력 (Power)
│  └─ 옴의 법칙
│
├─ 신호 개념 (Signal)
│  ├─ 아날로그 신호
│  ├─ 디지털 신호
│  ├─ 노이즈
│  └─ 임피던스
│
├─ 회로 기본 요소 (Electronics)
│  ├─ Resistor
│  ├─ Capacitor
│  ├─ Inductor (개념만)
│  └─ Pull-up / Pull-down
│
├─ MCU 관점 전자
│  ├─ GPIO 전기적 특성
│  ├─ 입력 / 출력 전압 레벨
│  ├─ Sink / Source
│  └─ 보호 회로
│
├─ 전원 & 안정성
│  ├─ Power Supply
│  ├─ Decoupling Capacitor
│  ├─ Ground
│  └─ Reset 안정성
│
└─ 실무 문제 연결
   ├─ 왜 GPIO가 안 먹지?
   ├─ 왜 랜덤 리셋 되지?
   ├─ 왜 ADC 값이 튀지?
   └─ 왜 CAN이 불안정하지?
```
