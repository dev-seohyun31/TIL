# MCU Memory System 정리

> **정의**: MCU에서 메모리는 **CPU가 프로그램을 실행하고 시스템 전체(주변장치·상태·데이터)를 제어하기 위해 접근하는 주소 공간 제공 시스템**이다.

이 문서는 Flash, RAM, ROM, Memory Map을 하나의 관점(주소 공간 기반 시스템)으로 통합해 정리한다.

---

## 1. MCU Memory의 큰 그림

MCU에서 CPU는 모든 자원을 **주소(Address)** 로 접근한다.

- 코드 실행 → Flash 주소 읽기
- 변수 접근 → RAM 주소 읽기/쓰기
- GPIO 제어 → Peripheral Register 주소 쓰기

즉, CPU 관점에서 보면:

> **Flash, RAM, Peripheral, ROM 모두 동일한 주소 공간에 매핑된 메모리 객체**다.

---

## 2. Flash (Program Memory)

### 2.1 개념
- 비휘발성 메모리
- 전원 OFF 후에도 데이터 유지
- 프로그램 코드 저장 및 실행 (XIP: Execute In Place)

### 2.2 특징
| 항목 | 특성 |
|------|------|
읽기(Read) | 빠름 |
쓰기(Program) | 느림 |
삭제(Erase) | 블록 단위 |
수명 | 제한적 (10만~100만 cycle) |

### 2.3 실무 포인트
- Flash는 RAM보다 느림 → 타이밍 민감 코드는 RAM 실행 고려
- Flash 쓰기 중 같은 Bank에서 코드 실행 불가 (MCU 의존)
- 설정값 저장 시 Wear Leveling 설계 필요

---

## 3. RAM (Data / Stack / Heap)

### 3.1 개념
- 휘발성 메모리
- 실행 중 데이터 저장 공간

### 3.2 구성

#### Data 영역 (Global / Static)
- 전역 변수, static 변수
- 프로그램 시작 시 초기화

#### Stack
- 함수 호출 프레임
- 지역 변수
- 인터럽트 컨텍스트 저장

특징:
- LIFO 구조
- 자동 관리
- 크기 제한 명확

#### Heap
- 동적 메모리 할당 (malloc)
- MCU에서는 사용 최소화 권장

### 3.3 실무 포인트
- Stack overflow = 시스템 즉사 위험
- ISR 내부 Stack 사용량 최소화
- Heap fragmentation 주의
- 정적 할당 우선 설계

---

## 4. ROM (Boot ROM)

### 4.1 개념
- MCU 제조사에서 내부에 고정 탑재한 코드 영역
- Reset 직후 가장 먼저 실행
- 사용자 수정 불가

### 4.2 역할

1. Reset Vector 처리
2. Boot Mode 결정
3. 최소 하드웨어 초기화
4. Flash Application 진입

### 4.3 부팅 흐름

```
Power ON / Reset
        ↓
Boot ROM 실행
        ↓
Boot Mode 결정
        ↓
Flash Application Jump
        ↓
Startup Code
        ↓
main()
```

### 4.4 실무 포인트
- Boot pin 설정 오류 → Flash 실행 안 됨
- Secure Boot 설정 시 서명 실패 → 실행 차단
- Flash 손상 시에도 ROM 기반 복구 가능

---

## 5. Memory Map

### 5.1 개념
- MCU 전체 주소 공간 구조 정의
- 각 주소 범위에 어떤 자원이 위치하는지 명시

### 5.2 전형적 구조 예시

```
0x0000_0000  Boot ROM
0x0800_0000  Flash
0x2000_0000  RAM
0x4000_0000  Peripheral Register
0xE000_0000  System Control
```

(MCU별 상이)

### 5.3 핵심 특징

- Peripheral도 메모리처럼 접근 (Memory-mapped I/O)

```c
#define GPIO_OUT (*(volatile uint32_t*)0xF003A100)
GPIO_OUT = 1;
```

CPU 입장에서는:
- Flash, RAM, GPIO 모두 "주소"일 뿐이다.

---

## 6. Linker Script와 Memory Map 관계

Linker Script는 Memory Map 위에 프로그램을 배치한다.

예시:

```ld
FLASH (rx) : ORIGIN = 0x08000000, LENGTH = 2M
RAM   (rwx): ORIGIN = 0x20000000, LENGTH = 512K
```

결정되는 요소:
- 코드 위치
- Stack 위치
- Heap 위치
- Vector Table 위치

---

## 7. 통합 관점 요약

### CPU 관점

CPU는 다음만 수행한다:

```
Instruction Fetch → 주소 읽기
Data Load/Store → 주소 읽기/쓰기
Peripheral Control → 주소 쓰기
```

### 따라서 Memory란?

> **Memory는 MCU에서 CPU가 실행, 데이터 처리, 하드웨어 제어를 수행하기 위해 사용하는 통합 주소 공간 시스템이다.**

---

## 8. Flash / RAM / ROM / Memory Map 관계 요약

| 구성요소 | 역할 |
-----------|------|
ROM | MCU 최초 부트 실행 |
Flash | 프로그램 저장 및 실행 |
RAM | 실행 중 데이터 저장 |
Memory Map | 전체 주소 구조 정의 |

---

## 9. 임베디드 개발자 관점 핵심 정리

- Flash는 저장소가 아니라 실행 자원
- RAM은 편한 공간이 아니라 위험 관리 대상
- ROM은 수정 불가한 시스템 관리자
- Memory Map은 시스템 전체 설계도

---

## 10. 한 줄 정리

> **MCU 메모리는 저장 장치가 아니라 CPU가 시스템 전체를 제어하기 위한 주소 기반 실행 인프라이다.**

---

(End of Document)

