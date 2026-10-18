# 02. STM32F411 기본 예제 — GPIO / SysTick / UART / PWM / ADC / EXTI / PLL

> 환경 구성은 **01_idx_environment.md** 를 먼저 끝낸 상태를 가정합니다.
> 기준 보드: **NUCLEO-F411RE**. 모든 예제는 CMSIS 레지스터 접근 방식(HAL 없음)입니다.

> **검증 상태**: 예제 1~7의 `main.c`와 공용 모듈을 Arm GNU Toolchain 13.2(Ubuntu `gcc-arm-none-eabi`) + 01번 문서의 `Makefile`/`link.ld`로 **모두 빌드·링크**했고, 벡터 테이블(초기 SP `0x20020000`, 리셋 벡터, SysTick/EXTI 핸들러 연결)을 확인했습니다. **실제 Nucleo 보드에서의 동작은 확인하지 못했습니다.** 레지스터 값은 RM0383/데이터시트 기준으로 작성했으니, 보드에서 이상하면 해당 문서와 대조해 주세요.

---

## 0. 시작하기 전에

### 0-1. 보드 핀 맵 (NUCLEO-F411RE)

| 기능 | 핀 | 비고 |
|---|---|---|
| LD2 (초록 LED) | **PA5** | High = 켜짐. TIM2_CH1(AF1)로도 사용 가능 |
| B1 (사용자 버튼) | **PC13** | 외부 풀업, **누르면 Low** |
| USART2 TX / RX | **PA2 / PA3** (AF7) | ST-LINK 가상 COM 포트와 연결 |
| 아날로그 입력 A0 | **PA0** (ADC1_IN0) | 가변저항 등을 연결해 시험 |

> 다른 보드(예: Black Pill)를 쓰면 LED·버튼 핀과 극성이 다릅니다. `board.h`의 `LED_PIN`과 포트를 바꾸고, 해당 보드의 회로도로 확인하세요.

### 0-2. 레지스터 접근의 기본 3단계

STM32의 주변장치는 **메모리 맵 I/O**입니다. 레지스터는 특정 주소에 있는 변수처럼 읽고 씁니다.

1. **클럭 켜기** — 리셋 후 대부분의 주변장치 클럭은 꺼져 있습니다 (`RCC->AHB1ENR` 등).
2. **핀/모드 설정** — GPIO 모드(`MODER`), 대체 기능(`AFR`) 등.
3. **주변장치 설정 후 동작** — 보드레이트, 타이머 주기, 변환 시작 등.

비트 조작 패턴은 세 가지만 기억하면 됩니다.

```c
REG |=  (1U << n);                        // n번 비트 세트
REG &= ~(1U << n);                        // n번 비트 클리어
REG  = (REG & ~(MASK << pos)) | (VAL << pos);   // 여러 비트 필드 교체
```

### 0-3. 폴더 구성과 예제 사용법

각 예제는 `main.c`만 바뀝니다. `board.c/h`, `uart.c/h`는 공용입니다.
예제를 시험하려면 해당 `main.c`를 `src/main.c`로 복사하고 `make` 하면 됩니다.

---

## 1. 공용 모듈

### 1-1. board — LED, SysTick, delay

**파일:** `src/board.h`

```c
#ifndef BOARD_H
#define BOARD_H

#include "stm32f4xx.h"

#define LED_PIN        5U                       /* PA5 */
#define LED_ON()       (GPIOA->BSRR = (1U << LED_PIN))
#define LED_OFF()      (GPIOA->BSRR = (1U << (LED_PIN + 16U)))
#define LED_TOGGLE()   (GPIOA->ODR ^= (1U << LED_PIN))

void     board_init(void);        /* SysTick 1 ms + LED 핀 설정 */
uint32_t millis(void);            /* 부팅 후 경과 ms */
void     delay_ms(uint32_t ms);

#endif
```

**파일:** `src/board.c`

```c
#include "board.h"

static volatile uint32_t g_ms;

/* 벡터 테이블의 약한(weak) 심볼을 덮어쓰는 인터럽트 핸들러 */
void SysTick_Handler(void)
{
    g_ms++;
}

uint32_t millis(void)
{
    return g_ms;
}

void delay_ms(uint32_t ms)
{
    uint32_t t0 = g_ms;
    while ((g_ms - t0) < ms) {
        __NOP();
    }
}

void board_init(void)
{
    /* 1) SysTick: 코어 클럭 / 1000 마다 인터럽트 → 1 ms */
    SysTick_Config(SystemCoreClock / 1000U);

    /* 2) GPIOA 클럭 ON */
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
    (void)RCC->AHB1ENR;                 /* 클럭 안정화용 더미 읽기 */

    /* 3) PA5 → 출력 모드 (MODER 2비트: 00 입력, 01 출력, 10 AF, 11 아날로그) */
    GPIOA->MODER = (GPIOA->MODER & ~(3U << (LED_PIN * 2U)))
                                  |  (1U << (LED_PIN * 2U));
}
```

### 1-2. uart — USART2 (PA2/PA3)

**파일:** `src/uart.h`

```c
#ifndef UART_H
#define UART_H

#include <stdint.h>

void uart2_init(uint32_t pclk1_hz, uint32_t baud);
void uart2_putc(char c);
void uart2_puts(const char *s);          /* '\n' → "\r\n" 자동 변환 */
void uart2_put_u32(uint32_t v);
char uart2_getc(void);                   /* 수신될 때까지 대기 */

#endif
```

**파일:** `src/uart.c`

```c
#include "uart.h"
#include "stm32f4xx.h"

void uart2_init(uint32_t pclk1_hz, uint32_t baud)
{
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
    RCC->APB1ENR |= RCC_APB1ENR_USART2EN;
    (void)RCC->APB1ENR;

    /* PA2, PA3 → 대체 기능(10) */
    GPIOA->MODER &= ~((3U << (2U * 2U)) | (3U << (3U * 2U)));
    GPIOA->MODER |=  ((2U << (2U * 2U)) | (2U << (3U * 2U)));

    /* AFRL: 핀당 4비트. AF7 = USART1~3 */
    GPIOA->AFR[0] &= ~((0xFU << (2U * 4U)) | (0xFU << (3U * 4U)));
    GPIOA->AFR[0] |=  ((7U   << (2U * 4U)) | (7U   << (3U * 4U)));

    /* BRR = fPCLK / baud (OVER8 = 0, 16배 오버샘플링). 반올림 */
    USART2->BRR = (pclk1_hz + (baud / 2U)) / baud;

    USART2->CR1 = USART_CR1_TE | USART_CR1_RE | USART_CR1_UE;
}

void uart2_putc(char c)
{
    while (!(USART2->SR & USART_SR_TXE)) { }
    USART2->DR = (uint8_t)c;
}

void uart2_puts(const char *s)
{
    while (*s) {
        if (*s == '\n') {
            uart2_putc('\r');
        }
        uart2_putc(*s++);
    }
}

void uart2_put_u32(uint32_t v)
{
    char buf[11];
    int i = 0;

    if (v == 0U) {
        uart2_putc('0');
        return;
    }
    while (v) {
        buf[i++] = (char)('0' + (v % 10U));
        v /= 10U;
    }
    while (i--) {
        uart2_putc(buf[i]);
    }
}

char uart2_getc(void)
{
    while (!(USART2->SR & USART_SR_RXNE)) { }
    return (char)USART2->DR;
}
```

---

## 2. 예제 1 — LED 깜박임 (GPIO 출력 + SysTick)

### 목표
LD2를 0.5초 간격으로 토글한다. "클럭 켜기 → 핀 모드 → 출력"의 기본 흐름을 익힌다.

### 핵심 개념

| 개념 | 설명 |
|---|---|
| `RCC->AHB1ENR` | GPIO 포트는 AHB1 버스에 있음. GPIOA는 비트 0 |
| `MODER` | 핀당 2비트. 출력은 `01` |
| `BSRR` | 비트 세트/리셋 레지스터. 하위 16비트 = Set, 상위 16비트 = Reset. **읽기-수정-쓰기 없이 원자적으로** 핀 변경 |
| `ODR` | 출력 데이터 레지스터. `^=` 로 토글 (인터럽트와 겹치면 BSRR이 더 안전) |
| SysTick | 코어 내장 24비트 다운 카운터. 16 MHz / 1000 = 16000 틱마다 인터럽트 → 1 ms |

### 코드

**파일:** `ex01_blink/main.c`

```c
#include "board.h"

int main(void)
{
    board_init();

    while (1) {
        LED_TOGGLE();
        delay_ms(500);
    }
}
```

### 동작 설명

1. `board_init()`이 SysTick을 1 ms 주기로 시작하고 PA5를 출력으로 설정한다.
2. `delay_ms()`는 `g_ms`(SysTick 핸들러가 1씩 올림)와의 차이로 시간을 잰다. `g_ms`가 `volatile`이어야 최적화로 루프가 사라지지 않는다.
3. 뺄셈 `(g_ms - t0)`는 부호 없는 정수 오버플로(약 49일 후)에도 올바르게 동작한다.

### 실습 과제
- 켜짐 100 ms / 꺼짐 900 ms로 바꿔 보기 (`LED_ON()`, `LED_OFF()` 사용)
- `BSRR` 대신 `ODR`을 직접 써서 같은 동작 만들기

---

## 3. 예제 2 — 버튼 입력 (GPIO 입력, 폴링)

### 목표
B1(PC13)을 눌렀을 때 LED를 토글한다.

### 핵심 개념

| 개념 | 설명 |
|---|---|
| 입력 모드 | `MODER` 필드 `00` (리셋 값) |
| `IDR` | 입력 데이터 레지스터. 핀 상태를 읽음 |
| 액티브 로우 | 누르면 0. 눌림 = **하강 에지** |
| 바운싱 | 기계식 스위치는 수 ms 동안 떨린다. 지연 또는 상태 확인으로 걸러야 함 |

### 코드

**파일:** `ex02_button/main.c`

```c
#include "board.h"

int main(void)
{
    board_init();

    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOCEN;
    (void)RCC->AHB1ENR;
    GPIOC->MODER &= ~(3U << (13U * 2U));          /* PC13 입력 */

    uint32_t last = 1U;                           /* 평소(안 눌림) = High */

    while (1) {
        uint32_t now = (GPIOC->IDR >> 13U) & 1U;

        if (last == 1U && now == 0U) {            /* 하강 에지 = 눌림 */
            LED_TOGGLE();
            delay_ms(30);                         /* 간단한 디바운스 */
        }
        last = now;
    }
}
```

### 동작 설명
이전 값(`last`)과 현재 값(`now`)을 비교해 **에지**를 찾는다. 레벨이 아니라 에지로 검출해야 버튼을 누르고 있는 동안 계속 토글되지 않는다.

### 한계와 개선
`delay_ms()`는 CPU를 붙잡는 **블로킹** 방식이다. 다른 작업이 있다면 `millis()`로 마지막 동작 시각을 기록해 비교하는 **논블로킹** 방식으로 바꾼다. (FPGA 구현에서는 이 문제가 카운터 기반 디바운스 회로로 바뀐다 → 03/04번 문서)

### 실습 과제
- 버튼을 누를 때마다 깜박임 주기를 100 → 250 → 500 → 1000 ms 로 순환시키기

---

## 4. 예제 3 — UART 송수신 (USART2, 에코)

### 목표
PC 터미널에 문자열을 출력하고, 받은 문자를 그대로 돌려보낸다(에코).

### 핵심 개념

| 개념 | 설명 |
|---|---|
| 대체 기능(AF) | PA2/PA3를 GPIO가 아니라 USART2 핀으로 연결. `MODER=10` + `AFR` 값 7 |
| `BRR` | `fPCLK / baud`. 기본 클럭에서 16,000,000 / 115,200 = 138.9 → **139** (오차 약 0.08%) |
| `SR.TXE` | 송신 데이터 레지스터 비어 있음 |
| `SR.RXNE` | 수신 데이터 있음 (`DR`을 읽으면 자동으로 지워짐) |
| 프레임 | 8N1: 시작 비트 1, 데이터 8, 패리티 없음, 정지 비트 1 |

### 코드

**파일:** `ex03_uart/main.c`

```c
#include "board.h"
#include "uart.h"

int main(void)
{
    board_init();
    uart2_init(16000000U, 115200U);      /* 리셋 후 PCLK1 = 16 MHz */

    uart2_puts("STM32F411 UART2 ready\n");

    while (1) {
        char c = uart2_getc();
        uart2_putc(c);                   /* 에코 */
        LED_TOGGLE();                    /* 수신 표시 */
    }
}
```

### 확인 방법
터미널(115200 8N1)을 열고 Nucleo를 리셋하면 `STM32F411 UART2 ready`가 보인다. 키를 누르면 같은 글자가 돌아온다.

### 실습 과제
- 수신 문자가 `1`이면 LED ON, `0`이면 LED OFF
- 한 줄(`\r` 입력 시까지)을 버퍼에 모아 `LED ON`, `LED OFF` 명령을 해석하기

---

## 5. 예제 4 — PWM (TIM2 CH1, LED 밝기 조절)

### 목표
LD2(PA5 = TIM2_CH1)의 밝기를 서서히 올렸다 내리는 "숨쉬기" 효과를 만든다.

### 핵심 개념

| 레지스터 | 역할 | 이 예제의 값 |
|---|---|---|
| `PSC` | 입력 클럭 분주. 카운터 클럭 = fTIM / (PSC+1) | 15 → 1 MHz |
| `ARR` | 자동 리로드(주기). 주기 = (ARR+1) 카운트 | 999 → 1 kHz |
| `CCR1` | 비교값(듀티) | 0 ~ 999 |
| `CCMR1.OC1M` | 출력 비교 모드. `110` = PWM 모드 1 (CNT < CCR이면 High) | 110 |
| `CCER.CC1E` | 채널 출력 활성화 | 1 |
| `EGR.UG` | 업데이트 이벤트 발생 → PSC/ARR 즉시 반영 | 1 |

듀티비 = `CCR1 / (ARR+1)`.

### 코드

**파일:** `ex04_pwm/main.c`

```c
#include "board.h"

static void pwm_init(void)
{
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
    RCC->APB1ENR |= RCC_APB1ENR_TIM2EN;
    (void)RCC->APB1ENR;

    /* PA5 → 대체 기능, AF1 = TIM2_CH1 */
    GPIOA->MODER  = (GPIOA->MODER  & ~(3U   << (5U * 2U))) | (2U << (5U * 2U));
    GPIOA->AFR[0] = (GPIOA->AFR[0] & ~(0xFU << (5U * 4U))) | (1U << (5U * 4U));

    TIM2->PSC  = 16U - 1U;               /* 16 MHz / 16 = 1 MHz */
    TIM2->ARR  = 1000U - 1U;             /* 1 MHz / 1000 = 1 kHz */
    TIM2->CCR1 = 0U;

    TIM2->CCMR1 = TIM_CCMR1_OC1M_1 | TIM_CCMR1_OC1M_2   /* PWM mode 1 */
                | TIM_CCMR1_OC1PE;                       /* CCR 프리로드 */
    TIM2->CCER  = TIM_CCER_CC1E;
    TIM2->CR1   = TIM_CR1_ARPE;
    TIM2->EGR   = TIM_EGR_UG;                            /* 설정값 반영 */
    TIM2->CR1  |= TIM_CR1_CEN;                           /* 카운터 시작 */
}

int main(void)
{
    board_init();
    pwm_init();

    int duty = 0;
    int step = 1;

    while (1) {
        TIM2->CCR1 = (uint32_t)duty;
        duty += step;
        if (duty >= 999) { step = -1; }
        if (duty <= 0)   { step =  1; }
        delay_ms(1);                     /* 한 방향 약 1초, 왕복 약 2초 */
    }
}
```

### 동작 설명
`board_init()`이 PA5를 일반 출력으로 만들었지만, `pwm_init()`에서 다시 대체 기능으로 바꾼다. 이후 핀은 GPIO 출력 레지스터가 아니라 타이머가 구동한다.

### 실습 과제
- 주파수를 100 Hz로 낮춰 깜박임이 눈에 보이게 하고, 1 kHz와 비교해 보기
- 듀티 변화를 선형이 아닌 제곱 곡선(`duty*duty/999`)으로 바꿔 눈에 더 자연스러운 밝기 변화 만들기

---

## 6. 예제 5 — ADC 읽기 (PA0, 단일 변환)

### 목표
PA0의 전압을 12비트로 읽어 UART로 출력한다.

### 핵심 개념

| 개념 | 설명 |
|---|---|
| 아날로그 모드 | 핀 `MODER = 11` |
| `RCC->APB2ENR.ADC1EN` | ADC1은 APB2 버스 |
| `SMPR2` | 채널 0~9 샘플링 시간(채널당 3비트). `111` = 480 사이클 (신호 임피던스가 높아도 안정적) |
| `SQR3` | 변환 시퀀스의 1번째 채널 번호 |
| `CR2.ADON` / `SWSTART` | ADC 켜기 / 소프트웨어 변환 시작 |
| `SR.EOC` | 변환 완료 (`DR`을 읽으면 지워짐) |
| 환산 | `mV = 값 × 3300 / 4095` (기준전압 3.3 V 가정) |

### 코드

**파일:** `ex05_adc/main.c`

```c
#include "board.h"
#include "uart.h"

static void adc_init(void)
{
    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOAEN;
    RCC->APB2ENR |= RCC_APB2ENR_ADC1EN;
    (void)RCC->APB2ENR;

    GPIOA->MODER |= (3U << (0U * 2U));   /* PA0 아날로그 */

    ADC1->SMPR2 = (7U << 0);             /* 채널 0: 480 사이클 */
    ADC1->SQR1  = 0U;                    /* 변환 개수 = 1 */
    ADC1->SQR3  = 0U;                    /* 첫 변환 = 채널 0 */
    ADC1->CR2   = ADC_CR2_ADON;          /* ADC 켜기 */
    delay_ms(1);                         /* 안정화 대기 */
}

static uint16_t adc_read(void)
{
    ADC1->CR2 |= ADC_CR2_SWSTART;
    while (!(ADC1->SR & ADC_SR_EOC)) { }
    return (uint16_t)ADC1->DR;
}

int main(void)
{
    board_init();
    uart2_init(16000000U, 115200U);
    adc_init();

    while (1) {
        uint32_t raw = adc_read();
        uint32_t mv  = (raw * 3300U) / 4095U;

        uart2_puts("ADC raw=");
        uart2_put_u32(raw);
        uart2_puts("  mV=");
        uart2_put_u32(mv);
        uart2_puts("\n");

        LED_TOGGLE();
        delay_ms(500);
    }
}
```

### 확인 방법
A0 핀을 3.3 V에 연결하면 약 4095, GND에 연결하면 약 0이 나온다. 아무것도 연결하지 않으면(플로팅) 값이 불안정한 것이 정상이다.

### 실습 과제
- 8회 평균을 내어 노이즈를 줄이기
- 내부 온도 센서(ADC1 채널 18)를 읽어 보기 (데이터시트의 변환 공식 참고, 별도 설정 필요)

---

## 7. 예제 6 — 외부 인터럽트 (EXTI, 버튼)

### 목표
버튼을 폴링하지 않고, **인터럽트**로 LED를 토글한다. 메인 루프는 슬립(`WFI`) 상태로 둔다.

### 핵심 개념

| 개념 | 설명 |
|---|---|
| SYSCFG | EXTI 라인을 어느 포트에 연결할지 선택. 클럭은 `APB2ENR.SYSCFGEN` |
| `EXTICR[3]` | EXTI12~15 선택. EXTI13은 비트 7:4, 값 `0x2` = 포트 C |
| `EXTI->IMR` | 해당 라인 인터럽트 마스크 해제 |
| `EXTI->FTSR` | 하강 에지 트리거 |
| `EXTI->PR` | 대기(pending) 플래그. **1을 써서 지움** |
| NVIC | `EXTI15_10_IRQn`(EXTI10~15 공용)를 활성화 |

### 코드

**파일:** `ex06_exti/main.c`

```c
#include "board.h"

static volatile uint32_t last_ms;

/* 핸들러 이름은 시작 코드(startup)의 벡터 테이블과 정확히 같아야 한다 */
void EXTI15_10_IRQHandler(void)
{
    if (EXTI->PR & (1U << 13)) {
        EXTI->PR = (1U << 13);                 /* 플래그 지우기 (1을 씀) */

        if ((millis() - last_ms) > 50U) {      /* 50 ms 이내 재진입은 바운싱으로 무시 */
            last_ms = millis();
            LED_TOGGLE();
        }
    }
}

int main(void)
{
    board_init();

    RCC->AHB1ENR |= RCC_AHB1ENR_GPIOCEN;
    RCC->APB2ENR |= RCC_APB2ENR_SYSCFGEN;
    (void)RCC->APB2ENR;

    GPIOC->MODER &= ~(3U << (13U * 2U));       /* PC13 입력 */

    /* EXTI13 ← 포트 C */
    SYSCFG->EXTICR[3] = (SYSCFG->EXTICR[3] & ~(0xFU << 4)) | (0x2U << 4);

    EXTI->IMR  |= (1U << 13);                  /* 인터럽트 허용 */
    EXTI->FTSR |= (1U << 13);                  /* 하강 에지 */

    NVIC_SetPriority(EXTI15_10_IRQn, 1);
    NVIC_EnableIRQ(EXTI15_10_IRQn);

    while (1) {
        __WFI();                               /* 인터럽트 올 때까지 슬립 */
    }
}
```

### 동작 설명
버튼 하강 에지 → EXTI13 펜딩 → NVIC → `EXTI15_10_IRQHandler` 실행. 핸들러에서 `PR`을 지우지 않으면 즉시 다시 진입하는 무한 인터럽트가 된다.

### 실습 과제
- 상승 에지(`RTSR`)도 함께 설정해 누를 때와 뗄 때 다른 동작을 만들기
- 인터럽트 우선순위를 SysTick보다 낮춰 `millis()` 동작 변화를 관찰하기

---

## 8. 예제 7 — PLL로 100 MHz 만들기 (클럭 트리)

### 목표
리셋 후 HSI 16 MHz로 동작하는 시스템을 **PLL로 100 MHz**까지 올리고, 그에 맞춰 SysTick·UART 설정을 다시 계산한다.

### 클럭 구성

```
HSI 16 MHz ─ /M(8) → 2 MHz ─ ×N(100) → 200 MHz (VCO) ─ /P(2) → 100 MHz = SYSCLK
                                                                   │
                                                        AHB /1 → HCLK 100 MHz
                                                        APB1 /2 → PCLK1 50 MHz (최대 50)
                                                        APB2 /1 → PCLK2 100 MHz (최대 100)
```

| 항목 | 값 | 근거 |
|---|---|---|
| 전압 스케일 | Scale 1 | 100 MHz 동작에 필요 |
| Flash 대기 상태 | 3 WS | 2.7~3.6 V, 90~100 MHz 구간 |
| APB1 분주 | /2 | APB1 최대 50 MHz |
| TIM2 입력 클럭 | 100 MHz | APB1 분주가 1이 아니면 타이머 클럭은 PCLK1 × 2 |
| UART2 BRR | 50,000,000 / 115,200 ≈ **434** | PCLK1 기준 |

> 값은 STM32F411 데이터시트/레퍼런스 매뉴얼(RM0383)의 클럭 제한을 기준으로 정리했습니다. 설계 변경 시에는 최신 문서로 다시 확인하세요.

### 코드

**파일:** `ex07_pll100/main.c`

```c
#include "board.h"
#include "uart.h"

static void clock_100mhz(void)
{
    /* 1) 전압 스케일 1 */
    RCC->APB1ENR |= RCC_APB1ENR_PWREN;
    (void)RCC->APB1ENR;
    PWR->CR |= PWR_CR_VOS;

    /* 2) Flash 대기 상태 3 + 캐시/프리페치 (클럭 올리기 전에 먼저!) */
    FLASH->ACR = FLASH_ACR_LATENCY_3WS | FLASH_ACR_ICEN
               | FLASH_ACR_DCEN        | FLASH_ACR_PRFTEN;

    /* 3) PLL: 소스 HSI, M=8, N=100, P=2(필드값 0), Q=4 */
    RCC->PLLCFGR = (8U << 0) | (100U << 6) | (0U << 16) | (4U << 24);
    RCC->CR |= RCC_CR_PLLON;
    while (!(RCC->CR & RCC_CR_PLLRDY)) { }

    /* 4) 버스 분주: AHB /1, APB1 /2, APB2 /1 */
    RCC->CFGR = (RCC->CFGR & ~(RCC_CFGR_HPRE | RCC_CFGR_PPRE1 | RCC_CFGR_PPRE2))
              | RCC_CFGR_PPRE1_DIV2;

    /* 5) 시스템 클럭을 PLL로 전환하고 완료 확인 */
    RCC->CFGR = (RCC->CFGR & ~RCC_CFGR_SW) | RCC_CFGR_SW_PLL;
    while ((RCC->CFGR & RCC_CFGR_SWS) != RCC_CFGR_SWS_PLL) { }

    /* 6) CMSIS 전역 변수 갱신 (SysTick_Config가 이 값을 사용) */
    SystemCoreClock = 100000000U;
}

int main(void)
{
    clock_100mhz();
    board_init();                        /* 새 SystemCoreClock 기준으로 1 ms 설정 */
    uart2_init(50000000U, 115200U);      /* PCLK1 = 50 MHz */

    uart2_puts("SYSCLK = 100 MHz\n");

    while (1) {
        LED_TOGGLE();
        delay_ms(500);                   /* 실제 시간은 예제 1과 동일해야 정상 */
    }
}
```

### 확인 방법
- LED 깜박임이 예제 1과 같은 속도(0.5초)여야 한다. 빠르거나 느리면 `SystemCoreClock` 또는 분주 설정이 틀린 것이다.
- UART 문자열이 깨지지 않으면 PCLK1 계산이 맞은 것이다.

### 흔한 실수

| 실수 | 결과 |
|---|---|
| Flash 대기 상태를 올리기 전에 클럭 전환 | 오동작, 하드 폴트 |
| `SystemCoreClock` 미갱신 | `delay_ms`가 실제와 다르게 동작 |
| APB1 분주를 잊음 | APB1 50 MHz 한계 초과 |
| 전환 후 `SWS` 확인 생략 | PLL 잠금 전 코드가 계속 진행됨 |

### 실습 과제
- 예제 4의 PWM을 100 MHz 환경에서 다시 1 kHz로 만들기 (타이머 클럭 100 MHz 기준 `PSC`, `ARR` 재계산)
- PLL 설정을 M=16, N=200 으로 바꿔도 같은 100 MHz가 나오는지 계산해 보기

---

## 9. 예제 요약

| # | 주제 | 배우는 것 | FPGA 대응 (03/04번 문서) |
|---|---|---|---|
| 1 | LED 깜박임 | RCC, MODER, BSRR, SysTick | `blink`, `tick_gen` |
| 2 | 버튼 입력 | IDR, 에지 검출, 바운싱 | `debounce` |
| 3 | UART | AF, BRR, SR 플래그 | `uart_tx`, `uart_rx` |
| 4 | PWM | PSC/ARR/CCR, 출력 비교 | `pwm`, `breathing` |
| 5 | ADC | 샘플링 시간, 변환 시퀀스 | (FPGA 내장 ADC는 벤더별로 다름, 제외) |
| 6 | EXTI | 인터럽트 체인 | `debounce`의 `press_pulse` |
| 7 | PLL | 클럭 트리, 대기 상태 | (FPGA는 MMCM/PLL IP, 제외) |

### 권장 학습 순서
`01 환경` → 예제 1 → 2 → 3 → 4 → 6 → 5 → 7 → `03/04 HDL 구현`

---

## 10. 자주 묻는 질문

**Q. HAL을 안 쓰는 이유는?**
레지스터 구조와 클럭·핀 설정의 원리를 먼저 이해하기 위해서다. 원리를 알면 HAL/LL/CubeMX가 내부에서 하는 일이 보인다. 양산 코드는 HAL/LL이 유지보수에 유리한 경우가 많다.

**Q. `volatile`은 언제 쓰나?**
인터럽트 핸들러와 메인 코드가 같이 쓰는 변수(`g_ms`), 하드웨어 레지스터처럼 값이 코드 밖에서 바뀌는 변수에 쓴다. CMSIS의 레지스터 구조체는 이미 `volatile`로 선언되어 있다.

**Q. printf를 쓰고 싶다.**
`_write()`를 구현해 UART로 연결하면 된다. newlib-nano에서 부동소수점 printf는 별도 링커 옵션이 필요하고 코드 크기가 크게 늘어난다. 이 예제들은 의도적으로 직접 만든 출력 함수만 사용했다.

다음 문서: **03_vhdl_implementation.md**, **04_verilog_implementation.md**
