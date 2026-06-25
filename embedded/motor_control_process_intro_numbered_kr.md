# 모터 제어를 처음 볼 때 알아야 할 실행 흐름

대상 독자: 전기전자와 모터 제어를 처음 접하는 임베디드 개발자입니다.

목적: 모터 제어의 모든 이론을 설명하기보다, 실제 모터 제어 코드를 읽고 실험하기 전에 반드시 알아야 할 최소 실행 흐름을 정리합니다.

## 목차

1. [모터 제어는 전력을 직접 다루는 소프트웨어입니다](#section-1)
2. [모터 제어 하드웨어는 전원 흐름과 구동 흐름으로 나눌 수 있습니다](#section-2)
3. [PMIC는 MCU에 전원을 공급하고 MCU를 감시합니다](#section-3)
4. [MCU는 모터를 직접 돌리지 않고 인버터를 제어합니다](#section-4)
5. [모터는 stator의 자기장을 rotor가 따라가며 회전합니다](#section-5)
6. [인버터는 DC 전력을 U/V/W 3상 전력으로 바꿉니다](#section-6)
7. [게이트 드라이버는 MCU PWM을 전력 스위치 구동 신호로 바꿉니다](#section-7)
8. [3상 인버터는 보통 6개의 스위치로 구성됩니다](#section-8)
9. [Dead time은 상하단 스위치의 동시 ON을 막습니다](#section-9)
10. [PWM은 평균 전압과 전류를 제어하기 위한 방식입니다](#section-10)
11. [모터 제어는 피드백이 있어야 안정적으로 동작합니다](#section-11)
12. [속도 제어는 목표 토크를 만들고 전류 제어는 PWM duty를 만듭니다](#section-12)
13. [FOC는 3상 전류를 d/q축으로 바꿔 제어합니다](#section-13)
14. [PWM과 ADC는 정해진 타이밍에 맞춰 움직입니다](#section-14)
15. [전류 제어는 ISR에서 빠르게 처리하는 경우가 많습니다](#section-15)
16. [속도 제어와 Display 처리는 RTOS task로 둘 수 있습니다](#section-16)
17. [RTOS 설계에서는 priority, stack, heap을 함께 봐야 합니다](#section-17)
18. [실제 실행 흐름은 목표 입력에서 피드백 보정까지 반복됩니다](#section-18)
19. [처음 실험할 때는 MCU 단품 검증과 실제 모터 검증을 구분해야 합니다](#section-19)
20. [정리](#section-20)

<a id="section-1"></a>

<img width="1492" height="826" alt="모터제어도식도" src="https://github.com/user-attachments/assets/fd824df1-7baf-457a-8446-67ecd4f6249e" />


## 1. 모터 제어는 전력을 직접 다루는 소프트웨어입니다

모터 제어는 단순히 모터를 켜고 끄는 코드가 아닙니다. 모터 제어는 전력을 다루는 하드웨어를 MCU가 매우 짧은 주기로 제어하는 구조입니다.

예를 들어 모터를 단순히 켜기만 하면 부하에 따라 속도가 달라질 수 있습니다. 하지만 실제 시스템에서는 보통 다음과 같은 요구가 있습니다.

```text
- 목표 속도로 회전해야 합니다.
- 목표 토크를 만들어야 합니다.
- 사용자가 속도를 올리면 부드럽게 가속해야 합니다.
- 부하가 바뀌어도 속도를 유지해야 합니다.
- 이상 상황에서는 안전하게 멈춰야 합니다.
```

따라서 모터 제어는 다음 일을 반복하는 과정입니다.

```text
목표값을 받습니다.
현재 상태를 측정합니다.
목표와 현재의 차이를 계산합니다.
다음 PWM duty를 갱신합니다.
인버터가 모터에 전력을 공급합니다.
다시 현재 상태를 측정합니다.
```

이 반복이 충분히 빠르고 안정적으로 이루어져야 모터가 원하는 속도와 토크로 동작합니다.

<a id="section-2"></a>

## 2. 모터 제어 하드웨어는 전원 흐름과 구동 흐름으로 나눌 수 있습니다

모터 제어 하드웨어는 크게 두 흐름으로 나눠서 보면 이해하기 쉽습니다.

첫 번째는 MCU가 안정적으로 동작하기 위한 전원 흐름입니다.

```text
외부 전원
  ↓
PMIC
  ↓
MCU
```

두 번째는 모터를 실제로 돌리기 위한 구동 흐름입니다.

```text
Display / 상위 제어기
  ↓
MCU
  ↓ PWM
Gate Driver
  ↓
Inverter
  ↓ U/V/W 3상 전력
Motor
```

그리고 모터 제어에는 피드백 흐름이 반드시 필요합니다.

```text
Motor / Inverter
  ↓
전류 센서, 전압 센서, 온도 센서, Hall/Encoder
  ↓
MCU
```

즉 MCU는 명령을 받아서 PWM을 출력하고, 인버터는 PWM을 바탕으로 모터에 전력을 공급하고, 센서는 현재 상태를 다시 MCU에 알려줍니다.

<a id="section-3"></a>

## 3. PMIC는 MCU에 전원을 공급하고 MCU를 감시합니다

MCU는 외부 전원을 그대로 사용하지 않습니다. 예를 들어 보드에 12V가 들어와도 MCU는 보통 5V, 3.3V, 또는 더 낮은 코어 전압을 사용합니다.

이때 PMIC가 중간에서 전압을 변환하고 관리합니다.

```text
외부 전원 12V
  ↓
PMIC
  ↓
5V / 3.3V / MCU core 전원
  ↓
MCU
```

PMIC는 단순한 전압 변환기가 아닙니다. MCU가 정상적으로 동작하는지도 감시합니다.

PMIC의 대표 역할은 다음과 같습니다.

```text
- MCU에 필요한 전압을 공급합니다.
- 전압이 너무 낮거나 높은지 감시합니다.
- MCU reset을 제어합니다.
- watchdog으로 MCU가 멈췄는지 확인합니다.
- fault 발생 시 safe state로 전환할 수 있습니다.
```

예를 들어 MCU가 소프트웨어 오류로 멈추면 watchdog 갱신이 끊길 수 있습니다. 그러면 PMIC는 MCU가 정상적으로 동작하지 않는다고 판단하고 reset을 걸거나 시스템을 안전 상태로 보낼 수 있습니다.

중요한 점은 모터를 돌리는 큰 전력이 PMIC를 통해 모터로 가는 것이 아니라는 점입니다. 모터 전력은 별도의 DC 링크 전원에서 인버터를 거쳐 모터로 전달됩니다.

<a id="section-4"></a>

## 4. MCU는 모터를 직접 돌리지 않고 인버터를 제어합니다

MCU는 모터에 직접 큰 전류를 공급하지 않습니다. MCU가 출력하는 것은 보통 3.3V 또는 5V 수준의 작은 PWM 제어 신호입니다.

모터를 실제로 돌리는 큰 전력은 인버터가 담당합니다.

```text
MCU
  ↓ 작은 PWM 제어 신호
Gate Driver
  ↓ 전력 스위치 구동 신호
Inverter
  ↓ U/V/W 3상 전력
Motor
```

따라서 MCU의 역할은 모터에 직접 전기를 넣는 것이 아니라, 인버터의 전력 스위치를 언제 켜고 끌지 결정하는 것입니다.

한 문장으로 정리하면 다음과 같습니다.

```text
MCU는 모터를 직접 돌리지 않고, 인버터가 모터를 돌리도록 PWM으로 지시합니다.
```

<a id="section-5"></a>

## 5. 모터는 stator의 자기장을 rotor가 따라가며 회전합니다

PMSM 또는 BLDC 모터를 단순하게 보면 다음 구조입니다.

```text
Motor
  ├─ Stator
  ├─ Rotor
  └─ Shaft
```

Stator는 고정되어 있는 부분입니다. Stator 안에는 U, V, W 3상 코일이 있습니다. 인버터가 이 코일들에 전류를 흘리면 자기장이 만들어집니다.

Rotor는 실제로 회전하는 부분입니다. PMSM의 rotor에는 영구자석이 들어 있습니다. 영구자석은 전류를 넣지 않아도 자기장을 가지고 있는 자석입니다.

Stator의 U/V/W 코일이 회전 자기장을 만들면 rotor의 영구자석은 그 자기장을 따라가려고 하면서 회전합니다.

Shaft는 rotor와 함께 회전하는 축입니다. Shaft는 외부 기구에 동력을 전달합니다.

```text
U/V/W 전류
  ↓
Stator에 회전 자기장 생성
  ↓
Rotor 영구자석이 자기장을 따라 회전
  ↓
Shaft 회전
  ↓
외부 기구에 동력 전달
```

즉 모터에서 전기는 stator 코일로 들어가고, 힘은 rotor에 생기고, 출력은 shaft로 나갑니다.

<a id="section-6"></a>

## 6. 인버터는 DC 전력을 U/V/W 3상 전력으로 바꿉니다

인버터는 DC 전원을 받아서 모터에 필요한 U/V/W 3상 전력으로 바꾸는 전력 변환 장치입니다.

```text
DC 전원
  ↓
Inverter
  ↓
U / V / W 3상 출력
  ↓
Motor
```

여기서 U/V/W를 준다는 말은 모터의 세 권선에 전압과 전류를 공급한다는 뜻입니다.

모터에는 보통 세 개의 전력 단자가 있습니다.

```text
U 단자
V 단자
W 단자
```

인버터는 이 세 단자에 시간적으로 서로 어긋난 전압과 전류를 공급합니다. 이 세 상의 전류가 stator에서 회전 자기장을 만들고, rotor가 그 자기장을 따라 회전합니다.

<a id="section-7"></a>

## 7. 게이트 드라이버는 MCU PWM을 전력 스위치 구동 신호로 바꿉니다

MCU의 PWM은 작은 디지털 신호입니다. 이 신호만으로는 MOSFET, IGBT, SiC MOSFET 같은 전력 스위치를 직접 빠르고 안정적으로 켜고 끄기 어렵습니다.

그래서 MCU와 인버터 사이에 게이트 드라이버가 들어갑니다.

```text
MCU PWM
  ↓
Gate Driver
  ↓
MOSFET / IGBT gate 신호
```

게이트 드라이버의 역할은 다음과 같습니다.

```text
- MCU PWM을 전력 스위치 구동 신호로 바꿉니다.
- 게이트를 빠르게 충전하고 방전합니다.
- high-side 스위치를 구동할 수 있게 합니다.
- fault 상태를 MCU에 알려줄 수 있습니다.
- 보호 기능 또는 dead time 보조 기능을 가질 수 있습니다.
```

즉 게이트 드라이버는 MCU의 작은 명령을 인버터 스위치가 실제로 알아들을 수 있는 전력 구동 신호로 바꿔주는 인터페이스입니다.

<a id="section-8"></a>

## 8. 3상 인버터는 보통 6개의 스위치로 구성됩니다

3상 인버터는 보통 6개의 전력 스위치를 사용합니다.

```text
U상: UH, UL
V상: VH, VL
W상: WH, WL
```

각 상에는 high-side 스위치와 low-side 스위치가 있습니다.

U상 하나만 보면 다음과 같습니다.

```text
DC+
 |
UH
 |
U 출력 → 모터 U상
 |
UL
 |
DC-
```

UH가 켜지면 U 출력은 DC+ 쪽에 연결됩니다. UL이 켜지면 U 출력은 DC- 쪽에 연결됩니다.

이 구조가 U, V, W에 각각 있으므로 총 6개의 스위치가 필요합니다.

```text
3상 × 상단/하단 2개 = 6개 스위치
```

모터가 6개의 입력을 요구하는 것이 아닙니다. 모터로 가는 전력선은 U/V/W 3개입니다. 6개의 PWM은 인버터 내부의 6개 스위치를 제어하기 위한 신호입니다.

<a id="section-9"></a>

## 9. Dead time은 상하단 스위치의 동시 ON을 막습니다

한 상의 high-side와 low-side 스위치는 동시에 켜지면 안 됩니다.

예를 들어 U상에서 UH와 UL이 동시에 켜지면 다음과 같은 일이 생깁니다.

```text
DC+ → UH → UL → DC-
```

즉 DC+와 DC-가 거의 바로 연결됩니다. 이 상태를 shoot-through라고 합니다. 매우 큰 전류가 흐를 수 있고, 전력 스위치나 게이트 드라이버가 손상될 수 있습니다.

이를 막기 위해 스위치를 바꿔 켤 때 아주 짧은 시간 동안 둘 다 OFF로 둡니다. 이 시간이 dead time입니다.

```text
UH ON
  ↓
UH OFF
  ↓
dead time
  ↓
UL ON
```

Dead time은 너무 짧으면 shoot-through 위험이 있고, 너무 길면 전압 왜곡과 토크 리플이 커질 수 있습니다. 그래서 실제 시스템에서는 전력 소자와 게이트 드라이버 특성을 보고 적절한 값을 정해야 합니다.

<a id="section-10"></a>

## 10. PWM은 평균 전압과 전류를 제어하기 위한 방식입니다

PWM은 Pulse Width Modulation의 약자입니다. MCU가 아날로그 전압을 직접 만드는 것이 아니라 0과 1을 빠르게 반복하면서 평균값을 조절하는 방식입니다.

예를 들어 3.3V PWM에서 duty가 50%라면 실제 핀 전압은 0V와 3.3V를 반복하지만, 평균적으로는 1.65V처럼 동작합니다.

```text
Duty 25% → 평균 전압 낮음
Duty 50% → 평균 전압 중간
Duty 75% → 평균 전압 높음
```

모터 제어에서는 PWM이 게이트 드라이버와 인버터를 거쳐 큰 전력 PWM으로 바뀝니다.

MCU는 U/V/W 세 상에 대해 duty를 계산합니다.

```text
duty_u → UH/UL
duty_v → VH/VL
duty_w → WH/WL
```

모터 권선은 인덕턴스가 있기 때문에 전류가 전압처럼 순간적으로 튀지 않습니다. 그래서 빠른 PWM 전압을 가해도 전류는 비교적 부드럽게 변합니다.

즉 PWM은 모터에 실제로 필요한 평균 전압과 전류를 만들기 위한 방식입니다.

<a id="section-11"></a>

## 11. 모터 제어는 피드백이 있어야 안정적으로 동작합니다

목표 속도를 1000 rpm으로 설정했다고 해서 모터가 자동으로 정확히 1000 rpm으로 도는 것은 아닙니다.

부하가 걸리면 속도가 떨어질 수 있습니다. 부하가 줄면 속도가 올라갈 수 있습니다. 전원 전압이나 온도에 따라서도 동작이 달라질 수 있습니다.

그래서 MCU는 현재 모터 상태를 계속 알아야 합니다.

대표적인 피드백은 다음과 같습니다.

```text
- Hall sensor 또는 Encoder: rotor 위치와 속도
- 전류 센서: U/V/W 전류 또는 DC-link 전류
- 전압 센서: DC-link 전압
- 온도 센서: 모터 또는 인버터 온도
- Fault 신호: gate driver 또는 PMIC 이상 상태
```

Hall sensor나 Encoder는 shaft 또는 rotor 위치를 알려줍니다.

```text
Motor shaft
  ↓
Hall / Encoder
  ↓
MCU GPT, GPIO, Input Capture 등
```

전류 센서는 보통 인버터 쪽에 있습니다. 이 전류 값은 MCU의 ADC로 들어갑니다.

```text
Inverter current sensor
  ↓
MCU ADC
  ↓
Iu, Iv, Iw 측정
```

모터 제어는 목표 명령을 내는 것으로 끝나지 않습니다. 현재 상태를 보고 다음 PWM을 계속 고치는 피드백 과정입니다.

<a id="section-12"></a>

## 12. 속도 제어는 목표 토크를 만들고 전류 제어는 PWM duty를 만듭니다

속도 제어와 전류 제어는 역할이 다릅니다.

속도 제어는 목표 속도와 현재 속도의 차이를 보고 필요한 토크를 계산합니다.

```text
speed_ref - speed_actual
  ↓
속도 PI 제어기
  ↓
Iq_ref
```

PMSM에서는 토크가 주로 q축 전류인 Iq와 관련됩니다. 그래서 속도 제어기의 출력은 보통 `Iq_ref`가 됩니다.

전류 제어는 이 `Iq_ref`를 실제 전류로 만들기 위해 필요한 전압과 PWM duty를 계산합니다.

```text
Iq_ref - Iq
  ↓
전류 PI 제어기
  ↓
Vq
  ↓
SVPWM
  ↓
PWM duty
```

즉 속도 제어는 “목표 속도를 만족하려면 얼마나 밀어야 하는가”를 결정합니다. 전류 제어는 “그 힘을 만들기 위해 지금 어떤 PWM을 내야 하는가”를 결정합니다.

<a id="section-13"></a>

## 13. FOC는 3상 전류를 d/q축으로 바꿔 제어합니다

FOC는 Field Oriented Control의 약자입니다. 처음부터 모든 수식을 이해할 필요는 없습니다. 최소 흐름만 먼저 보면 됩니다.

먼저 MCU는 전류 센서로 현재 3상 전류를 측정합니다.

```text
Iu, Iv, Iw
```

이 전류를 Clarke 변환과 Park 변환을 통해 d/q축 전류로 바꿉니다.

```text
Iu, Iv, Iw
  ↓ Clarke 변환
Iα, Iβ
  ↓ Park 변환
Id, Iq
```

`Id`는 rotor 자석 방향의 전류 성분입니다. `Iq`는 rotor 자석과 직각 방향의 전류 성분입니다.

PMSM에서는 보통 `Iq`가 토크를 만드는 주된 전류입니다.

```text
토크 ≈ Kt × Iq
```

전류 제어기는 목표 전류와 현재 전류의 차이를 계산합니다.

```text
Id_error = Id_ref - Id
Iq_error = Iq_ref - Iq
```

이 오차를 PI 제어기에 넣으면 필요한 전압 명령이 나옵니다.

```text
Id_error → PI → Vd
Iq_error → PI → Vq
```

이후 다시 역변환을 수행합니다.

```text
Vd, Vq
  ↓ inverse Park
Vα, Vβ
  ↓ SVPWM
duty_u, duty_v, duty_w
```

최종적으로 `duty_u`, `duty_v`, `duty_w`가 나오고, 이 값이 다음 PWM 출력에 반영됩니다.

<a id="section-14"></a>

## 14. PWM과 ADC는 정해진 타이밍에 맞춰 움직입니다

모터 제어에서는 타이밍이 매우 중요합니다.

예를 들어 PWM을 20 kHz로 동작시킨다면 PWM 주기는 50 us입니다.

```text
20 kHz = 1초에 20000번
1 / 20000 = 50 us
```

PWM이 스위칭되는 순간에는 노이즈가 많을 수 있습니다. 그래서 보통 PWM 주기 중 전류가 비교적 안정적인 지점에서 ADC를 trigger합니다.

Center-aligned PWM에서는 PWM 중앙 근처에서 전류를 측정하는 경우가 많습니다.

```text
GTM TOM PWM
  ↓
특정 시점에서 EVADC trigger
  ↓
전류 변환 완료
  ↓
EVADC interrupt 발생
  ↓
전류 제어 ISR 실행
```

즉 PWM timer는 단순히 출력만 만드는 것이 아니라, ADC가 언제 전류를 측정할지도 함께 결정할 수 있습니다.

<a id="section-15"></a>

## 15. 전류 제어는 ISR에서 빠르게 처리하는 경우가 많습니다

전류 제어는 다음 PWM duty를 정해진 시간 안에 계산해야 합니다. 그래서 보통 높은 우선순위 ISR에서 처리하는 경우가 많습니다.

예를 들어 EVADC 전류 변환 완료 interrupt가 발생하면 ISR에서 다음 흐름을 처리할 수 있습니다.

```text
ADC 결과 읽기
  ↓
Iu, Iv, Iw 계산
  ↓
Clarke/Park 변환
  ↓
Id/Iq 전류 제어
  ↓
Vd/Vq 계산
  ↓
SVPWM
  ↓
다음 PWM duty 업데이트
```

이 ISR은 하드웨어 peripheral interrupt에 의해 실행되는 코드입니다. 전류 제어는 PWM/ADC 타이밍과 직접 묶여 있으므로 지연에 민감합니다.

즉 전류 제어 ISR은 빠르고 짧게 끝나야 합니다. printf, 긴 loop, blocking 대기 같은 작업은 넣지 않는 것이 좋습니다.

<a id="section-16"></a>

## 16. 속도 제어와 Display 처리는 RTOS task로 둘 수 있습니다

전류 제어와 달리 속도 제어는 더 느린 주기로 실행해도 됩니다. Rotor에는 관성이 있기 때문에 속도는 50 us 단위로 급격하게 변하지 않습니다.

예를 들어 다음과 같이 주기를 나눌 수 있습니다.

```text
전류 제어 ISR: 20 kHz, 50 us마다
속도 제어 Task: 1 kHz, 1 ms마다
Display Task: 10~100 Hz
통신 Task: 시스템 요구에 따라 설정
```

RTOS task로 두기 좋은 작업은 다음과 같습니다.

```text
- 속도 제어
- 목표 속도 ramp
- Display 갱신
- 통신 처리
- 상태 모니터링
- 로그 처리
```

기준은 다음과 같습니다.

```text
PWM 주기 안에 반드시 끝나야 하는가?
  → ISR에 가까움

몇 ms 늦어도 바로 위험하지 않은가?
  → RTOS task에 가까움
```

<a id="section-17"></a>

## 17. RTOS 설계에서는 priority, stack, heap을 함께 봐야 합니다

RTOS를 사용할 때는 task를 만드는 것만으로 끝나지 않습니다. Priority, stack, heap을 함께 고려해야 합니다.

Priority는 어떤 task를 먼저 실행할지 결정합니다.

```text
높은 priority:
제어에 직접 관련된 빠른 작업

낮은 priority:
Display, logging, shell, 일반 상태 표시
```

높은 priority task가 오래 실행되면 낮은 priority task가 실행되지 못할 수 있습니다. 따라서 높은 priority task는 짧게 실행하고, 필요하면 delay나 blocking wait로 CPU를 양보해야 합니다.

Stack은 task가 사용하는 지역 변수와 함수 호출에 필요합니다. Display나 printf를 많이 쓰는 task는 stack을 더 많이 사용할 수 있습니다.

Heap은 task, queue, semaphore, timer 등을 동적으로 만들 때 사용될 수 있습니다. 임베디드에서는 heap 사용량을 명확히 관리하는 것이 중요합니다.

실무에서는 다음을 켜두는 것이 좋습니다.

```text
- Stack overflow hook
- Malloc failed hook
- Task high water mark 확인
- HardFault 또는 trap 원인 기록
```

처음에는 task를 많이 만들기보다, 각 task가 어떤 주기와 우선순위로 실행되어야 하는지 먼저 정리하는 것이 좋습니다.

<a id="section-18"></a>

## 18. 실제 실행 흐름은 목표 입력에서 피드백 보정까지 반복됩니다

목표 속도를 1000 rpm으로 설정하는 상황을 예로 들어보겠습니다.

1. 사용자가 display에서 start를 누르고 목표 속도를 1000 rpm으로 설정합니다.

```text
Display → MCU
speed_ref = 1000 rpm
```

2. 모터는 처음에 정지해 있습니다.

```text
현재 속도 = 0 rpm
목표 속도 = 1000 rpm
```

3. 속도 제어기는 속도 오차가 크다고 판단합니다.

```text
speed_ref - speed_actual = 큰 양수
```

4. 속도 PI 제어기는 더 큰 토크가 필요하다고 보고 `Iq_ref`를 증가시킵니다.

```text
속도 PI 출력 → Iq_ref 증가
```

5. 전류 제어 ISR은 ADC로 현재 전류를 읽습니다.

```text
Iu, Iv, Iw 측정
```

6. 현재 전류를 d/q축으로 변환합니다.

```text
Iu, Iv, Iw → Id, Iq
```

7. 목표 전류와 실제 전류의 차이를 계산합니다.

```text
Iq_error = Iq_ref - Iq
Id_error = Id_ref - Id
```

8. 전류 PI 제어기가 필요한 전압을 계산합니다.

```text
Iq_error → Vq
Id_error → Vd
```

9. 전압 명령을 SVPWM을 통해 duty로 바꿉니다.

```text
Vd, Vq → SVPWM → duty_u, duty_v, duty_w
```

10. MCU가 다음 PWM duty를 업데이트합니다.

```text
duty_u → U상 PWM
duty_v → V상 PWM
duty_w → W상 PWM
```

11. 인버터는 U/V/W 3상 전력을 모터에 공급하고 모터가 회전합니다.

12. Hall 또는 Encoder가 현재 속도와 위치를 다시 MCU에 알려줍니다.

```text
Motor shaft → Hall/Encoder → MCU
```

이 과정이 계속 반복되면서 모터는 목표 속도에 가까워집니다.

<a id="section-19"></a>

## 19. 처음 실험할 때는 MCU 단품 검증과 실제 모터 검증을 구분해야 합니다

처음 모터 제어를 실험할 때는 실제 모터를 바로 돌리기보다 단계적으로 확인하는 것이 좋습니다.

MCU만 있는 상황에서 확인할 수 있는 것은 다음과 같습니다.

```text
- MCU 부팅
- Debugger 연결
- OneEye/DAS 연결
- FreeRTOS task scheduling
- GPIO 출력
- PWM pin activity
- ADC 입력 read
- Hall/Encoder 입력 에뮬레이션
- FOC 계산 dry-run
```

하지만 MCU만으로 확인할 수 없는 것도 명확합니다.

```text
- 실제 모터 회전
- 전류 피드백 polarity
- Gate driver 보호 동작
- Inverter dead time 안전성
- DC-link 측정 정확도
- FOC loop의 실제 안정성
- 부하 조건에서의 토크/속도 제어 성능
```

따라서 초기 실험의 목적은 “모터가 잘 도는지”가 아니라, MCU와 소프트웨어 기반이 제대로 준비되어 있는지 확인하는 것입니다.

<a id="section-20"></a>

## 20. 정리

모터 제어는 한 번 명령을 내리고 끝나는 구조가 아닙니다. 목표값을 받고, 현재 상태를 측정하고, 목표와 현재의 차이를 계산하고, 다음 PWM duty를 갱신하는 과정을 계속 반복하는 구조입니다.

하드웨어 관점에서는 다음과 같이 볼 수 있습니다.

```text
PMIC → MCU 안정 동작 지원
MCU → PWM 생성
Gate Driver → 전력 스위치 구동
Inverter → DC를 U/V/W 3상 전력으로 변환
Motor → 전기 에너지를 회전 운동으로 변환
Sensor → 현재 상태를 MCU에 피드백
```

소프트웨어 관점에서는 다음과 같이 볼 수 있습니다.

```text
전류 센싱
  ↓
전류 제어 ISR
  ↓
PWM duty 업데이트
  ↓
속도 제어 task
  ↓
Iq_ref 갱신
  ↓
Display/통신 task
  ↓
사용자 명령 및 상태 표시
```

처음에는 Clarke, Park, SVPWM 같은 수식보다 전체 흐름을 먼저 잡는 것이 좋습니다. 전력이 어디서 와서 어디로 가는지, MCU가 어떤 신호를 만들고 어떤 피드백을 받는지, 어떤 주기로 제어가 반복되는지를 이해하면 모터 제어 코드를 훨씬 쉽게 읽을 수 있습니다.

