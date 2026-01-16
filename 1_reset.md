# 실제 개념
## Reset이란
MCU를 안전한 초기 상태로 돌리는 하드웨어 매커니즘으로, 모든 하드웨어와 CPU를 정의된 초기 상태로 되돌려서 안정적인 실행 환경을 다시 만드는 제어신호입니다.
1. CPU PC: Reset Vector로 이동
2. Core register 초기화
3. Interrupt 비활성화
4. Bus transaction 정지
5. 일부/전체 Peripheral reset
6. Clock source 기본값으로 복원 : Internal Oscillator
7. PLL disable
8. System clock 초기 상태 복귀
9. Reset cause flag 기록
즉, 시간, 상태, 실행흐름까지 모두 동시에 초기화시킵니다.
## Reset Source란
Reset을 발생시키는 원인입니다. 왜 리셋되었는지를 반드시 추적해야 하므로 하드웨어 레벨에서 소스를 구분한다. 
### 1. POR (Power-On Reset)
* 전원이 처음 인가될 떄 발생하는 리셋입니다.
### 2. WDT Reset (Watchdog Reset)
* MCU가 멈췄을 때 자동복구를 목적으로 SW가 일정시간 안에 정상 동작 신호를 주지 못하면 강제로 발생하는 리셋입니다.
* 예: 무한 루프, 데드락, stack overflow, 인터럽트 폭주 등 
### 3. SW Reset
* 소프트웨어가 의도적으로 발생하는 리셋입니다.
* 예: 펌웨어 업데이트 후 재부팅, 치명적 오류 감지 후 안전 재시작

