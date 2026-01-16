# 실제 개념
## RAM이란
MCU에서 사용하는 RAM은 대부분 SRAM (Static RAM)으로, 휘발성이고 빠르지만 비싼 메모리입니다. 
### RAM의 물리적 특성
* 읽기
  * CPU 버스에 직접 연결되어 있고, 중간 컨트롤러 없이 바로 접근합니다. 
  * 셀 구조 덕분에 빠른 비파괴 읽기
  * 접근 지연이 작고, 반복 읽기에 제한이 없습니다.
* 쓰기
  * 전기적 상태를 즉시 변경할 수 있습니다.
  * overwrite 가능하고 erase 개념이 없어서 빠릅니다
  * 쓰기 횟수 제한이 없음
  * page, sector 개념이 없습니다.

### RAM 사용 목적
이런 특성 덕분에 MCU에서는 RAM을 다음과 같이 실행 중(런타임) 존재하는 작업 공간으로 사용합니다. 
* 실행 중 변수 저장
* Stack 프레임 관리
* 통신 버퍼
* 타이밍 민감 코드 실행 영역 등

## RAM 내부 구역
RAM은 물리적으로 하나의 메모리이지만,
링커 스크립트가 논리적 영역(Data / Stack / Heap)을 나누어 배치합니다.
### 1. Data 영역
런타임에 정적 데이터를 저장합니다.
* 전역 변수: `int g`
* 정적 변수: `static int g`
* 프로그램 시작 시, 초기화된 상태로 존재해야 하는 데이터

Data 영역은 두 영역으로 나누어서 저장합니다.
* `.data` 영역: 초기값 있는 데이터 (`int g = 3;`)
  * 초기값은 Flash에 저장되어 있다가, 부팅 시 startup code가 RAM으로 복사하여 저장합니다. 덕분에 실행 중에 빠르게 데이터에 접근할 수 있습니다.
* `.bss` 영역: 초기값 없는 데이터를 0으로 초기화하여 저장 (`int g;`)
  * 부팅 시 startup code `.bss` 영역 전체를 0으로 세팅합니다.
### 2. Stack 영역
CPU가 자동으로 관리한는 실행 흐름 저장 영역입니다. 주로 다음을 저장합니다. 
* 함수 호출 리턴 주소
* 저장된 레지스터
* 지역 변수
* 인터럽트 진입 시 자동 저장되는 컨텍스트
```C
void foo(void) {
  int a = 1;     // stack
  char buf[128]; // stack
  bar(a);        // stack에 리턴 주소, 레지스터 등이 stack에 쌓임
}
```
* stack은 매우 빠르고, 자동으로 관리가 되지만, 용량이 넘치면 시스템이 망가집니다. (Stack Overflow)
* Stack Overflow 시, 즉시 예외가 나지 않고 메모리를 덮어쓰면서 trap/hardFault/리셋/오동작으로 결과가 나타나는 위험이 있습니다. 
### 3. Heap 영역
* Heap은 `malloc/free` 같은 동적 할당이 쓰는 영역입니다.
* 예: `Sensor* p = malloc(sizeof(Sensor));`
* MCU에서는 malloc이 언제 얼마나 걸릴지 예측이 안되고, 조각화 현상때문에 권장되지 않습니다. 
* 따라서, 정적 할당을 권장하고, 필요하면 고정크기 메모리 풀을 직접 만들어서 사용합니다.
  ```C
  // 메모리 풀 방식으로, Heap 없이 동적 관리가 가능하다.
  #define POOL_SIZE 10
  
  static Sensor pool[POOL_SIZE];
  static bool used[POOL_SIZE];
  
  Sensor* alloc_sensor(void)
  {
      for(int i=0;i<POOL_SIZE;i++)
      {
          if(!used[i])
          {
              used[i] = true;
              return &pool[i];
          }
      }
      return NULL;
  }
  ```
  ```C
  // Static 구조체
  static Sensor sensor;
  ```

## MCU 부팅 시 RAM의 흐름
1. Reset 발생: PC, SP, CPU 상태 초기화
2. Stack pointer 초기화: CPU의 SP 레지스터에 Stack 시작 주소를 설정
3. `.data` 영역 초기화: Flash에 저장된 초기값을 RAM으로 복사하여, 실행 중 빠른 접근 실현
4. `.bss` 영역 초기화: `memset(.bss, 0)`로 0으로 초기화
5. `main()` 진입:
    * Stack: 함수 호출 시 자동 사용
    * Heap: malloc 사용 시 생성
    * Data: 전역/정적 변수 읽고 쓰기

# 내가 이해한 바 
# 개발자 관점에서 신경쓸 점 
## 1. ISR Stack 사용량 (context 저장, 스택 소비) 을 고려하여 ISR 안에서 큰 지역 배열이나 깊은 함수 호출을 피한다. 
## 2. Heap 사용을 최소화하고 정적 할당을 우선한다. 
# 기술지원/고객 대응 관점에서 신경쓸 점 
# 개선된 점 / 깨달음
* 서버와 달리, Stack Overflow도 크리티컬한 이슈이므로 스택 사용량을 계산하며 개발해야 한다. 
