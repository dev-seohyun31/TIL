# RTOS를 처음 볼 때 알아야 할 실행 흐름

> 대상 독자: RTOS 기반 임베디드 프로젝트를 처음 접하는 개발자  
> 목적: RTOS의 모든 기능을 설명하기보다, 실제 코드를 읽고 작성하기 전에 반드시 알아야 할 최소 실행 흐름을 정리합니다.

---

## 목차

1. [RTOS는 제어의 역전 구조입니다](#1-rtos는-제어의-역전-구조입니다)
2. [Task는 보통 계속 살아있는 실행 흐름입니다](#2-task는-보통-계속-살아있는-실행-흐름입니다)
3. [Task는 상태를 가집니다](#3-task는-상태를-가집니다)
4. [Scheduler는 priority와 상태를 보고 실행할 task를 고릅니다](#4-scheduler는-priority와-상태를-보고-실행할-task를-고릅니다)
5. [Tick interrupt는 RTOS의 시간 관리 기준입니다](#5-tick-interrupt는-rtos의-시간-관리-기준입니다)
6. [Preemptive와 Cooperative scheduling](#6-preemptive와-cooperative-scheduling)
7. [Queue는 task 간 메시지 전달 통로입니다](#7-queue는-task-간-메시지-전달-통로입니다)
8. [Semaphore는 이벤트를 알리는 데 사용할 수 있습니다](#8-semaphore는-이벤트를-알리는-데-사용할-수-있습니다)
9. [ISR에서는 짧게 처리하고 task를 깨우는 것이 좋습니다](#9-isr에서는-짧게-처리하고-task를-깨우는-것이-좋습니다)
10. [높은 priority task가 오래 돌면 다른 task가 굶을 수 있습니다](#10-높은-priority-task가-오래-돌면-다른-task가-굶을-수-있습니다)
11. [Delay, busy wait, blocking은 서로 다른 대기 방식입니다](#11-delay-busy-wait-blocking은-서로-다른-대기-방식입니다)
12. [모터 제어처럼 jitter에 민감한 코드는 주의해야 합니다](#12-모터-제어처럼-jitter에-민감한-코드는-주의해야-합니다)
13. [공유 자원은 보호해야 합니다](#13-공유-자원은-보호해야-합니다)
14. [Critical section과 scheduler suspend는 신중히 사용해야 합니다](#14-critical-section과-scheduler-suspend는-신중히-사용해야-합니다)
15. [AURIX 같은 멀티코어에서는 코어별 실행 구조도 고려해야 합니다](#15-aurix-같은-멀티코어에서는-코어별-실행-구조도-고려해야-합니다)
16. [정리](#16-정리)

---

## 1. RTOS는 제어의 역전 구조입니다

Bare-metal 구조에서는 개발자가 코드의 실행 흐름을 직접 제어합니다.

예를 들어 다음과 같은 구조입니다.

```c
int main(void)
{
    initHardware();

    while (1)
    {
        readSensor();
        controlMotor();
        processCommunication();
        runDiagnosis();
    }
}
```

이 구조에서는 `while (1)` 안에 작성된 순서대로 코드가 실행됩니다.

즉, 개발자가 직접 “무엇을 먼저 실행하고, 무엇을 나중에 실행할지” 결정합니다.

반면 RTOS를 사용하면 실행 흐름의 일부를 RTOS scheduler에게 넘기게 됩니다.

```c
int main(void)
{
    initHardware();

    xTaskCreate(ControlTask, "Control", 512, NULL, 3, NULL);
    xTaskCreate(ComTask,     "Com",     512, NULL, 2, NULL);
    xTaskCreate(DiagTask,    "Diag",    512, NULL, 1, NULL);

    vTaskStartScheduler();

    while (1)
    {
        /* 보통 여기까지 오지 않습니다. */
    }
}
```

개발자는 task를 만들고, 각 task의 priority와 stack size 등을 설정합니다.

그 이후에는 scheduler가 현재 실행 가능한 task들 중 어떤 task를 실행할지 결정합니다.

이처럼 개발자가 직접 흐름을 모두 제어하던 방식에서, RTOS scheduler가 실행 흐름을 관리하는 방식으로 바뀌기 때문에 이를 “제어의 역전”이라고 표현할 수 있습니다.

Spring 같은 framework에서 application 코드가 framework에 의해 호출되는 것과 비슷한 관점으로 볼 수 있습니다.

다만 RTOS는 더 낮은 계층에서 CPU, stack, interrupt, context switching 같은 실행 흐름을 직접 다룬다는 차이가 있습니다.

---

## 2. Task는 보통 계속 살아있는 실행 흐름입니다

RTOS에서 task는 단순히 한 번 호출되는 함수가 아닙니다.

Task는 RTOS가 관리하는 하나의 실행 흐름입니다.

일반적으로 task는 다음처럼 `while (1)` 또는 `for (;;)` 구조를 가집니다.

```c
void ComTask(void *argument)
{
    for (;;)
    {
        Message_t msg;

        xQueueReceive(rxQueue, &msg, portMAX_DELAY);

        processMessage(&msg);
    }
}
```

이 task는 시스템이 동작하는 동안 계속 살아 있습니다.

하지만 항상 CPU를 사용하고 있는 것은 아닙니다.

위 코드에서 queue에 메시지가 없으면 `ComTask`는 기다리는 상태가 됩니다.

이때 task는 CPU를 점유하지 않고 `Blocked` 상태로 들어갑니다.

메시지가 queue에 들어오면 task는 다시 `Ready` 상태가 되고, scheduler가 선택하면 `Running` 상태가 되어 코드를 실행합니다.

---

## 3. Task는 상태를 가집니다

RTOS task는 보통 다음과 같은 상태를 가집니다.

| 상태 | 의미 |
|---|---|
| Ready | 실행할 준비가 되었지만 아직 CPU를 받지 못한 상태입니다 |
| Running | 현재 CPU에서 실행 중인 상태입니다 |
| Blocked | 시간, queue, semaphore, notification 등을 기다리는 상태입니다 |
| Suspended | 명시적으로 정지된 상태입니다 |
| Deleted | 삭제된 상태입니다 |

중요한 것은 `Blocked`와 `Suspended`의 차이입니다.

`Blocked`는 기다리는 조건이 있는 상태입니다.

예를 들어:

```c
vTaskDelay(pdMS_TO_TICKS(10));
```

를 호출하면 task는 10ms 동안 `Blocked` 상태가 됩니다.

10ms가 지나면 자동으로 다시 `Ready` 상태가 됩니다.

또는:

```c
xQueueReceive(rxQueue, &msg, portMAX_DELAY);
```

를 호출했는데 queue에 데이터가 없다면 task는 `Blocked` 상태가 됩니다.

나중에 queue에 데이터가 들어오면 다시 `Ready` 상태가 됩니다.

반면 `Suspended`는 강제로 멈춘 상태입니다.

```c
vTaskSuspend(taskHandle);
```

로 task를 멈추면, 시간이나 queue 이벤트가 발생해도 자동으로 깨어나지 않습니다.

다시 실행하려면 다음처럼 명시적으로 resume해야 합니다.

```c
vTaskResume(taskHandle);
```

따라서 일반적인 이벤트 대기에는 `Blocked` 상태를 사용하고, `Suspended`는 특별히 task 실행을 강제로 멈춰야 할 때 조심해서 사용하는 것이 좋습니다.

---

## 4. Scheduler는 priority와 상태를 보고 실행할 task를 고릅니다

RTOS scheduler는 현재 `Ready` 상태인 task들 중에서 실행할 task를 고릅니다.

일반적으로 FreeRTOS는 priority 기반 scheduler를 사용합니다.

즉, 높은 priority task가 `Ready` 상태이면 낮은 priority task보다 먼저 실행됩니다.

예를 들어 다음과 같은 task들이 있다고 하겠습니다.

```text
Priority 3: ControlTask
Priority 2: ComTask
Priority 1: LoggingTask
```

이때 `ControlTask`가 `Ready` 상태이면 scheduler는 `ControlTask`를 실행합니다.

`ControlTask`가 delay, queue 대기, semaphore 대기 등으로 `Blocked` 상태가 되면 그다음 priority의 task가 실행될 수 있습니다.

중요한 점은 priority가 “중요도”와 완전히 같은 의미는 아니라는 점입니다.

RTOS에서는 priority를 “얼마나 빨리 응답해야 하는가”, “deadline이 얼마나 짧은가”의 관점으로 정하는 것이 좋습니다.

Logging도 중요할 수 있지만, 1ms 제어 task보다 즉시 실행되어야 하는 작업은 아닐 수 있습니다.

---

## 5. Tick interrupt는 RTOS의 시간 관리 기준입니다

RTOS에는 보통 tick interrupt가 있습니다.

예를 들어 `configTICK_RATE_HZ`가 1000이면 1ms마다 tick interrupt가 발생합니다.

```c
#define configTICK_RATE_HZ 1000
```

이 경우 RTOS의 시간 기준은 1ms가 됩니다.

Tick interrupt에서는 보통 다음과 같은 일이 일어납니다.

```text
1. tick count 증가
2. delay 중인 task의 시간이 끝났는지 확인
3. software timer 처리 준비
4. 필요하면 context switch 요청
```

즉, tick interrupt는 RTOS가 시간 흐름을 관리하는 기준입니다.

Task delay, timeout, software timer, 같은 priority task 간 time slicing 등이 tick을 기준으로 동작할 수 있습니다.

다만 RTOS의 모든 실행 전환이 tick에서만 발생하는 것은 아닙니다.

높은 priority task가 계속 `Ready` 상태이고 block되지 않는다면, 그 task는 여러 tick 동안 계속 실행될 수 있습니다.

또한 ISR에서 queue나 semaphore를 통해 높은 priority task를 깨우면 tick 사이에도 context switching이 일어날 수 있습니다.

예를 들어:

```text
LoggingTask 실행 중
  → CAN RX interrupt 발생
  → ISR에서 xQueueSendFromISR() 호출
  → ComTask가 Ready 상태가 됨
  → ComTask priority가 더 높으면 즉시 전환될 수 있음
```

따라서 tick은 RTOS의 시간 관리 기준이며, scheduler가 task의 delay와 timeout을 판단하는 중요한 기준입니다.

---

## 6. Preemptive와 Cooperative scheduling

RTOS scheduler에는 크게 preemptive 방식과 cooperative 방식이 있습니다.

Preemptive scheduling에서는 높은 priority task가 `Ready` 상태가 되면 현재 실행 중인 낮은 priority task를 중단시키고 CPU를 가져갈 수 있습니다.

FreeRTOS에서는 보통 다음 설정과 관련이 있습니다.

```c
#define configUSE_PREEMPTION 1
```

이 설정이 1이면 preemptive scheduling을 사용합니다.

반대로 cooperative scheduling에서는 task가 스스로 CPU를 양보해야 다른 task가 실행될 수 있습니다.

예를 들어 task가 다음과 같은 API를 호출해야 합니다.

```c
taskYIELD();
vTaskDelay(...);
xQueueReceive(...);
xSemaphoreTake(...);
```

즉, cooperative 방식에서는 task가 양보하지 않고 계속 실행되면 다른 task가 실행되기 어렵습니다.

FreeRTOS에서 cooperative 방식은 보통 다음처럼 설정합니다.

```c
#define configUSE_PREEMPTION 0
```

다만 실무에서는 preemptive scheduling을 사용하는 경우가 많습니다.

대신 high priority task가 너무 오래 실행되지 않도록 설계해야 합니다.

---

## 7. Queue는 task 간 메시지 전달 통로입니다

RTOS queue는 task 간 데이터를 전달하기 위한 통신 객체입니다.

예를 들어 CAN 메시지를 수신하는 ISR이 있고, 이 메시지를 `ComTask`에서 처리한다고 하겠습니다.

```c
void CAN_RxISR(void)
{
    Message_t msg;

    readCanMessage(&msg);

    xQueueSendFromISR(rxQueue, &msg, NULL);
}
```

`ComTask`는 queue에서 메시지를 기다립니다.

```c
void ComTask(void *argument)
{
    Message_t msg;

    for (;;)
    {
        xQueueReceive(rxQueue, &msg, portMAX_DELAY);

        processCanMessage(&msg);
    }
}
```

이 흐름은 다음과 같습니다.

```text
ComTask가 xQueueReceive() 호출
  → queue에 메시지가 없으면 Blocked 상태
  → CAN RX ISR에서 queue에 메시지 넣음
  → ComTask가 Ready 상태가 됨
  → scheduler가 선택하면 Running 상태로 전환
  → 메시지 처리
```

Queue에는 message가 들어갑니다.

Task는 queue에 데이터가 들어오기를 기다리며 `Blocked` 상태가 될 수 있습니다.

실행 가능한 task들이 scheduler를 기다리는 내부 구조는 ready list 또는 ready queue라고 부릅니다.

이 둘은 역할이 다르므로 구분해서 이해하는 것이 좋습니다.

---

## 8. Semaphore는 이벤트를 알리는 데 사용할 수 있습니다

Semaphore는 task 간에 “어떤 일이 발생했다”는 신호를 전달할 때 사용할 수 있습니다.

예를 들어 ADC 변환 완료 interrupt가 발생하면, ISR에서 semaphore를 give하고 `AdcTask`가 깨어나 후처리를 할 수 있습니다.

```c
void ADC_CompleteISR(void)
{
    BaseType_t xHigherPriorityTaskWoken = pdFALSE;

    xSemaphoreGiveFromISR(adcDoneSem, &xHigherPriorityTaskWoken);

    portYIELD_FROM_ISR(xHigherPriorityTaskWoken);
}
```

`AdcTask`는 semaphore를 기다립니다.

```c
void AdcTask(void *argument)
{
    for (;;)
    {
        xSemaphoreTake(adcDoneSem, portMAX_DELAY);

        processAdcResult();
    }
}
```

이때 semaphore를 받았다고 해서 task가 무조건 즉시 실행되는 것은 아닙니다.

정확히는:

```text
semaphore가 give됨
  → 기다리던 task가 Blocked에서 Ready 상태로 이동
  → scheduler가 priority를 보고 실행 여부 결정
```

만약 깨어난 task가 현재 실행 중인 task보다 priority가 높다면 바로 실행될 수 있습니다.

하지만 priority가 낮다면 Ready 상태가 되어도 나중에 실행됩니다.

즉, semaphore는 task를 “실행”시키는 것이 아니라 task를 “실행 가능 상태로 깨우는” 역할을 합니다.

---

## 9. ISR에서는 짧게 처리하고 task를 깨우는 것이 좋습니다

RTOS를 사용하더라도 interrupt는 여전히 중요합니다.

특히 fault, ADC, PWM, timer, communication RX 같은 이벤트는 ISR에서 시작되는 경우가 많습니다.

다만 ISR 안에서 모든 일을 처리하면 interrupt latency가 길어지고 시스템 전체 응답성이 나빠질 수 있습니다.

따라서 일반적인 원칙은 다음과 같습니다.

```text
ISR에서는 짧게 처리합니다.
무거운 처리는 task에게 넘깁니다.
```

예를 들어 CAN 메시지 처리의 경우:

```text
CAN RX ISR
  → CAN message를 읽음
  → queue에 message를 넣음
  → ComTask를 깨움
  → 실제 parsing과 처리는 ComTask에서 수행
```

이런 식으로 설계하면 ISR은 짧게 유지하면서도 복잡한 처리는 task context에서 수행할 수 있습니다.

단, fault 차단이나 PWM emergency stop처럼 즉시 처리해야 하는 보호 동작은 ISR 또는 하드웨어 safety mechanism에서 바로 처리해야 할 수 있습니다.

---

## 10. 높은 priority task가 오래 돌면 다른 task가 굶을 수 있습니다

RTOS를 사용한다고 해서 CPU가 자동으로 공평하게 나뉘는 것은 아닙니다.

특히 priority 기반 RTOS에서는 높은 priority task가 계속 `Ready` 상태로 오래 실행되면 낮은 priority task는 실행되지 못할 수 있습니다.

나쁜 예:

```c
void HighPriorityTask(void *argument)
{
    for (;;)
    {
        while (flag == 0)
        {
            /* busy wait */
        }

        process();
    }
}
```

위 코드는 `flag`가 바뀔 때까지 CPU를 계속 점유합니다.

이 task의 priority가 높다면 낮은 priority task들이 실행되지 못할 수 있습니다.

더 좋은 예:

```c
void HighPriorityTask(void *argument)
{
    for (;;)
    {
        ulTaskNotifyTake(pdTRUE, portMAX_DELAY);

        process();
    }
}
```

이 구조에서는 이벤트가 없을 때 task가 `Blocked` 상태가 됩니다.

따라서 CPU를 낭비하지 않고 다른 task가 실행될 수 있습니다.

RTOS에서는 “기다릴 때 CPU를 점유하지 않게 만드는 것”이 매우 중요합니다.

---

## 11. Delay, busy wait, blocking은 서로 다른 대기 방식입니다

이 세 개념은 꼭 구분해야 합니다.

### Busy wait

CPU를 계속 사용하면서 기다리는 방식입니다.

```c
while (flag == 0)
{
}
```

이 방식은 가능하면 피하는 것이 좋습니다.

### Delay

정해진 시간 동안 task를 재우는 방식입니다.

```c
vTaskDelay(pdMS_TO_TICKS(10));
```

이 task는 10ms 동안 `Blocked` 상태가 되고, 그동안 CPU는 다른 task가 사용할 수 있습니다.

주기 task라면 `vTaskDelay()`보다 `vTaskDelayUntil()`이 더 적합할 수 있습니다.

```c
void PeriodicTask(void *argument)
{
    TickType_t lastWakeTime = xTaskGetTickCount();

    for (;;)
    {
        runPeriodicJob();

        vTaskDelayUntil(&lastWakeTime, pdMS_TO_TICKS(10));
    }
}
```

### Blocking

Queue, semaphore, notification 등을 기다리며 task가 잠드는 것입니다.

```c
xQueueReceive(rxQueue, &msg, portMAX_DELAY);
```

이때 queue에 메시지가 없으면 task는 `Blocked` 상태가 됩니다.

메시지가 들어오면 다시 `Ready` 상태가 됩니다.

정리하면:

| 방식 | CPU 사용 여부 | 사용 예 |
|---|---:|---|
| Busy wait | 사용함 | 가능한 피해야 함 |
| Delay | 사용하지 않음 | 주기 대기 |
| Blocking | 사용하지 않음 | 이벤트/메시지 대기 |

---

## 12. 모터 제어처럼 jitter에 민감한 코드는 주의해야 합니다

RTOS task는 scheduler에 의해 실행되므로 실행 시점에 jitter가 생길 수 있습니다.

통신, logging, diagnosis, monitoring 같은 작업은 약간의 jitter가 허용되는 경우가 많습니다.

하지만 모터 전류 제어, ADC sampling, PWM update 같은 작업은 jitter에 매우 민감할 수 있습니다.

이런 경우에는 RTOS task만으로 고속 제어 loop를 처리하기보다 다음 구조를 고려합니다.

```text
PWM timer
  → ADC hardware trigger
  → ADC conversion complete ISR
  → current control / FOC 계산
  → PWM duty update
```

즉, 매우 빠르고 정확해야 하는 제어 경로는 hardware trigger와 high-priority ISR 중심으로 설계하고, RTOS task는 상대적으로 느린 작업을 담당하게 할 수 있습니다.

예를 들어:

```text
High-priority ISR:
  - ADC sampling 완료 처리
  - FOC 계산
  - PWM duty update
  - 긴급 fault 차단

RTOS task:
  - 속도 지령 처리
  - CAN 통신
  - 진단
  - 상태 모니터링
  - logging
```

다른 방법으로는 다음도 있습니다.

```text
1. RTOS task priority를 높이고 실행 시간을 매우 짧게 유지합니다.
2. vTaskDelayUntil()로 주기 task의 흔들림을 줄입니다.
3. 동일 priority task의 time slicing을 주의합니다.
4. control task와 logging/communication task를 분리합니다.
5. DMA, timer, peripheral trigger를 활용해 CPU 개입을 줄입니다.
6. 실제 target에서 GPIO toggle이나 trace tool로 jitter를 측정합니다.
```

중요한 것은 “RTOS task로 구현했으니 실시간이다”라고 가정하지 않는 것입니다.

실시간성은 반드시 측정하고 확인해야 합니다.

---

## 13. 공유 자원은 보호해야 합니다

여러 task 또는 ISR이 같은 자원에 접근하면 race condition이 발생할 수 있습니다.

여기서 “같은 자원”은 같은 변수만 의미하지 않습니다.

다음과 같은 것도 공유 자원입니다.

- 같은 global buffer
- 같은 communication queue
- 같은 SPI peripheral
- 같은 I2C bus
- 같은 flash driver
- 같은 diagnostic data structure
- 같은 hardware register set

예를 들어 두 개의 task가 모두 같은 SPI peripheral을 사용한다고 하겠습니다.

```text
SensorTask → SPI0를 통해 sensor 값을 읽음
DiagTask   → 같은 SPI0를 통해 external diagnostic IC 값을 읽음
```

이 경우 task는 두 개이지만, 실제 hardware 자원은 하나입니다.

두 task가 동시에 SPI0 register를 설정하거나 transmit buffer에 값을 넣으면, 한쪽 transfer가 끝나기 전에 다른 task가 SPI 설정을 바꿀 수 있습니다.

그러면 전송 데이터가 깨지거나, chip select timing이 꼬이거나, 예상하지 못한 slave device와 통신할 수 있습니다.

멀티코어 MCU에서는 이 문제가 더 직접적으로 드러날 수 있습니다.

예를 들어 AURIX처럼 CPU가 여러 개인 구조에서 다음처럼 동작할 수 있습니다.

```text
CPU0에서 SensorTask 실행 중
  → SPI0로 sensor read 시작

동시에 CPU1에서 DiagTask 실행 중
  → 같은 SPI0로 diagnostic IC read 시작
```

이처럼 두 task가 서로 다른 CPU에서 동시에 실행되더라도, 둘 다 같은 SPI0 peripheral을 사용한다면 같은 자원에 동시에 접근하는 문제가 됩니다.

반대로 SPI peripheral이 실제로 두 개이고, 각 task가 서로 다른 SPI를 사용한다면 상황이 다릅니다.

```text
SensorTask → SPI0 사용
DiagTask   → SPI1 사용
```

이 경우 SPI0와 SPI1은 서로 다른 hardware 자원이므로, 각 peripheral이 완전히 독립적이고 공유 buffer나 driver state도 분리되어 있다면 mutex가 필요 없을 수 있습니다.

따라서 중요한 질문은 “task가 몇 개인가?”가 아니라 “실제로 같은 자원을 공유하는가?”입니다.

같은 SPI0를 공유한다면 mutex로 보호할 수 있습니다.

```c
void SensorTask(void *argument)
{
    for (;;)
    {
        xSemaphoreTake(spi0Mutex, portMAX_DELAY);

        readSensorBySpi0();

        xSemaphoreGive(spi0Mutex);

        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

```c
void DiagTask(void *argument)
{
    for (;;)
    {
        xSemaphoreTake(spi0Mutex, portMAX_DELAY);

        readDiagIcBySpi0();

        xSemaphoreGive(spi0Mutex);

        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

Mutex는 “자원 자체를 잠그는” 개념에 가깝습니다.

한 task가 `spi0Mutex`를 잡고 있는 동안 다른 task는 같은 mutex를 얻을 수 없습니다.

다만 mutex를 잡은 상태에서 오래 block하면 안 됩니다.

가능하면 mutex를 잡는 구간은 짧게 유지해야 합니다.

---

## 14. Critical section과 scheduler suspend는 신중히 사용해야 합니다

공유 자원을 보호하는 방법에는 mutex 외에도 critical section이 있습니다.

FreeRTOS에서는 다음과 같은 API를 사용할 수 있습니다.

```c
taskENTER_CRITICAL();

/* 매우 짧은 공유 변수 접근 */

taskEXIT_CRITICAL();
```

Critical section 안에서는 interrupt 또는 scheduler 동작이 제한될 수 있습니다.

따라서 아주 짧은 코드에만 사용해야 합니다.

예:

```c
taskENTER_CRITICAL();
sharedCounter++;
taskEXIT_CRITICAL();
```

반면 긴 작업에는 critical section을 사용하면 안 됩니다.

나쁜 예:

```c
taskENTER_CRITICAL();

readSpiData();
processLargeData();
writeFlash();

taskEXIT_CRITICAL();
```

이런 코드는 system latency를 크게 악화시킬 수 있습니다.

Scheduler만 잠시 멈추는 API도 있습니다.

```c
vTaskSuspendAll();

/* scheduler 전환을 막고 싶은 짧은 구간 */

xTaskResumeAll();
```

하지만 이 역시 신중히 사용해야 합니다.

일반적인 공유 자원 보호에는 mutex를 먼저 고려하는 것이 좋습니다.

정리하면:

| 목적 | 사용 방법 |
|---|---|
| 짧은 공유 변수 보호 | critical section |
| task 간 공유 peripheral 보호 | mutex |
| ISR-task 간 flag 보호 | critical section 또는 atomic 설계 |
| 긴 작업 보호 | 구조 재검토 필요 |

---

## 15. AURIX 같은 멀티코어에서는 코어별 실행 구조도 고려해야 합니다

AURIX는 멀티코어 MCU입니다.

따라서 RTOS를 어떻게 올리느냐에 따라 구조가 달라질 수 있습니다.

가능한 구조는 크게 두 가지입니다.

```text
1. 코어별로 독립적인 RTOS scheduler를 두는 구조
2. 여러 코어를 하나의 RTOS가 관리하는 SMP 구조
```

많은 MCU 프로젝트에서는 코어별로 역할을 나누고, 각 코어에서 독립적인 scheduler 또는 독립적인 실행 흐름을 두는 방식이 사용됩니다.

다만 실제 구조는 사용 중인 FreeRTOS port, BSP, startup code, linker 설정에 따라 달라지므로 프로젝트 기준으로 확인해야 합니다.

멀티코어에서는 같은 시간에 여러 task 또는 ISR이 실제로 동시에 실행될 수 있습니다.

따라서 공유 자원 보호가 더 중요해집니다.

예:

```text
CPU0: ControlTask 실행 중
CPU1: ComTask 실행 중

두 task가 같은 shared buffer에 접근
  → 보호가 없으면 race condition 발생 가능
```

멀티코어 공유 자원은 다음을 확인해야 합니다.

```text
1. 이 자원이 어느 core에서 접근되는가?
2. ISR에서도 접근하는가?
3. cache coherency 문제가 있는가?
4. lock/mutex/spinlock이 필요한가?
5. peripheral register 접근 권한은 어떻게 되는가?
```

FreeRTOS mutex가 코어 간 보호까지 보장하는지는 port와 구조에 따라 다를 수 있습니다.

따라서 AURIX 멀티코어 환경에서는 RTOS 문서뿐 아니라 Infineon BSP, startup 구조, multicore synchronization 방식도 함께 확인해야 합니다.

---

## 16. 정리

RTOS를 처음 볼 때 가장 중요한 것은 API를 많이 외우는 것이 아닙니다.

실행 흐름을 이해하는 것입니다.

최소한 다음 정도는 알고 시작하는 것이 좋습니다.

```text
1. main에서 task를 만들고 scheduler를 시작합니다.
2. task는 보통 while(1) 구조로 계속 살아 있습니다.
3. task는 Ready, Running, Blocked, Suspended 상태를 오갑니다.
4. queue는 task 간 메시지 전달 통로입니다.
5. ISR에서는 짧게 처리하고 task를 깨우는 구조가 일반적입니다.
6. 높은 priority task가 오래 돌면 다른 task가 굶을 수 있습니다.
7. busy wait, delay, blocking은 서로 다른 대기 방식입니다.
8. jitter에 민감한 제어 코드는 timer, ADC, PWM, ISR 구조를 우선 고려해야 합니다.
```

RTOS는 코드를 자동으로 안전하게 만들어주는 도구가 아닙니다.

하지만 여러 실행 흐름을 명확하게 나누고, task 간 통신과 동기화를 구조적으로 관리할 수 있게 해주는 강력한 도구입니다.

따라서 RTOS를 사용할 때는 “이 task는 언제 깨어나는가?”, “무엇을 기다리는가?”, “누가 이 task를 깨우는가?”, “공유 자원은 어떻게 보호되는가?”를 계속 확인해야 합니다.

