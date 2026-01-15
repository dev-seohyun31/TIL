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
├─ Peripheral
│  ├─ GPIO
│  ├─ Timer / PWM
│  ├─ ADC / DAC
│  ├─ UART / SPI / I2C
│  └─ CAN / LIN / Ethernet (고급 MCU)
│
├─ Interrupt System
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
├─ IO & Peripheral (Automotive)
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
├─ Interrupt & Trap
│  ├─ Interrupt (IRQ)
│  ├─ Trap (Exception)
│  └─ Priority & Core Routing
│
├─ Tool & Framework
│  ├─ iLLD
│  ├─ AURIX Development Studio
│  └─ MCAL (AUTOSAR)
│
└─ Automotive OS (선택)
   ├─ Bare-metal
   ├─ AUTOSAR Classic
   └─ RTOS
```
