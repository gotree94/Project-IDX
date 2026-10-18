# Project-IDX — STM32F411에서 FPGA까지, 같은 로직을 두 번 만들기

> Google Project IDX(Firebase Studio) 환경에서 **STM32F411(CMSIS 레지스터 접근, HAL 없음)** 을 개발하고, <br>
> 그 예제를 **VHDL / Verilog FPGA 로직으로 그대로 옮겨 보는** 실습 문서 세트입니다.<br>
> <br>
> 대상 보드: **NUCLEO-F411RE** / FPGA 예시 보드: **Basys 3**<br>
> 작성 기준일: 2026-10-18

---

## ⚠️ 먼저 알아둘 점

Google이 **Firebase Studio(구 Project IDX) 종료를 발표**했습니다.

| 날짜 | 내용 |
|---|---|
| 2025-04-15 | Project IDX → **Firebase Studio**로 이름 변경 |
| 2026-03-19 | Firebase Studio 종료 발표 |
| 2026-06-22 | 신규 가입 / 신규 워크스페이스 생성 중단 |
| **2027-03-22** | **서비스 종료 예정** |

따라서 이 문서는 **(A) IDX/Firebase Studio 기준 설정**과 **(B) GitHub Codespaces · 로컬 VS Code에서
동일 환경 재현**을 함께 다룹니다. **새로운 교육과정에 IDX를 도입하는 용도에는 적합하지 않습니다.**
최신 상태는 공식 안내(<https://firebase.google.com/docs/studio/migrating-project>)에서 확인하세요.

---

## 문서 구성

| # | 문서 | 내용 | 검증 상태 |
|---|---|---|---|
| 01 | [idx_environment](01_idx_environment.md) | IDX/Codespaces/로컬 환경, `dev.nix`, `setup.sh`, `Makefile`, `link.ld`, 부팅 흐름 | Ubuntu `gcc-arm-none-eabi` 13.2로 빌드 확인 |
| 02 | [stm32f411_examples](02_stm32f411_examples.md) | CMSIS 레지스터 방식 예제 7개 (GPIO·SysTick·UART·PWM·ADC·EXTI·PLL) | **예제 1~7 빌드/링크 + 벡터 테이블 확인** (실보드 미확인) |
| 03 | [vhdl_implementation](03_vhdl_implementation.md) | 예제를 VHDL-93 로직으로 재구현 + 테스트벤치 4개 | **GHDL 4.1 시뮬레이션 전부 PASS** (합성/실보드 미확인) |
| 04 | [verilog_implementation](04_verilog_implementation.md) | 03과 1:1 대응하는 Verilog-2001 구현 + VHDL↔Verilog 문법 대응표 | **Icarus Verilog 전부 PASS** (합성/실보드 미확인) |

각 문서 하단에 "다음 문서 / 이전 문서" 링크가 있어 순서대로 읽으면 됩니다.

---

## 학습 경로

```
01 환경 구성
   └─ 02 예제 1 → 2 → 3 → 4 → 6 → 5 → 7        (MCU에서 원리를 이해)
        └─ 03 VHDL 구현  ─┐
           04 Verilog 구현 ┴─ 동일 로직을 하드웨어로 재설계 + 시뮬레이션
```

- **02번 문서의 권장 순서**: 예제 1(출력) → 2(입력) → 3(UART) → 4(PWM) → 6(인터럽트) → 5(ADC) → 7(클럭 트리)
- ADC(예제 5)와 PLL(예제 7)은 벤더 IP가 필요해 FPGA 문서에서는 **의도적으로 제외**되어 있습니다.

---

## 빠른 시작

### A. STM32 경로 (01 → 02)

```bash
bash setup.sh   # CMSIS 헤더 내려받기 (최초 1회)
make            # build/f411.elf, build/f411.bin 생성
```

1. `src/main.c`를 원하는 예제 코드로 교체 → `make`
2. `build/f411.bin`을 내려받아 **로컬 PC**에서 플래시
   - 가장 쉬움: Nucleo USB 연결 → `NODE_F411RE` 드라이브에 `.bin` 복사 (드래그 앤 드롭)
   - 또는 `st-flash --reset write f411.bin 0x8000000`
3. 시리얼 터미널 **115200 / 8N1** 접속 (ST-LINK 가상 COM 포트)

> 클라우드 IDE(IDX/Codespaces)는 **USB 장치에 접근할 수 없으므로** 빌드는 클라우드,
> 플래시·디버깅은 로컬 PC에서 하는 구조입니다.

### B. FPGA 경로 (03 또는 04)

```bash
# VHDL (GHDL)
ghdl -a --std=08 blink.vhd tick_gen.vhd debounce.vhd uart_tx.vhd uart_rx.vhd pwm.vhd breathing.vhd top.vhd tb_top_check.vhd
ghdl -e --std=08 tb_top_check
ghdl -r --std=08 tb_top_check

# Verilog (Icarus)
iverilog -g2012 -o sim_top tb_top_check.v top.v blink.v tick_gen.v debounce.v uart_tx.v uart_rx.v pwm.v breathing.v
vvp sim_top
```

`TOP CHECK: PASS`가 나오면 통과입니다. 합성은 Vivado에서 `*.xdc` 제약 파일과 함께 진행합니다.

---

## 예제 ↔ FPGA 모듈 대응

| STM32 예제 (02) | FPGA 모듈 (03/04) | 구현 방식 |
|---|---|---|
| 1. LED 토글 + `delay_ms(500)` | `blink` | 0.5초마다 반전하는 카운터 |
| SysTick 1 ms / `millis()` | `tick_gen` | 분주 카운터 + 1 클럭 펄스 |
| 2 / 6. 버튼, 바운싱, EXTI | `debounce` | 2단 동기화 + 안정 시간 카운터 + 에지 펄스 |
| 3. USART2 TX / RX | `uart_tx` / `uart_rx` | 상태 머신 + 비트 중앙 샘플링 |
| 4. TIM2 PWM | `pwm`, `breathing` | PSC/ARR/CCR과 동일 구조 |
| 5. ADC | — | 벤더 IP 필요 · 제외 |
| 7. PLL | — | MMCM/PLL IP 필요 · 제외 |

---

## 저장소 구조

```
Project-IDX/
├─ README.md                     ← 이 문서 (커버)
├─ 01_idx_environment.md         # 개발 환경 (IDX / Codespaces / 로컬)
├─ 02_stm32f411_examples.md      # STM32 예제 7개 + 공용 모듈(board, uart)
├─ 03_vhdl_implementation.md     # VHDL 구현 + 테스트벤치 + basys3.xdc
└─ 04_verilog_implementation.md  # Verilog 구현 + 문법 대응표
```

> **참고**: 코드는 전부 문서 안에 포함되어 있습니다. 실행하려면 해당 코드 블록을
> `dev.nix`, `setup.sh`, `Makefile`, `link.ld`, `src/*.c`, `vhdl/*.vhd`, `verilog/*.v` 등
> 문서에 표기된 파일 경로로 복사해야 합니다.

---

## 검증 상태 (정직한 표기)

| 항목 | 확인됨 | 확인 안 됨 |
|---|---|---|
| STM32 예제 1~7 빌드/링크 (gcc-arm-none-eabi 13.2) | ✅ | — |
| STM32 벡터 테이블(초기 SP, 리셋/SysTick/EXTI 핸들러) | ✅ | — |
| **STM32 실보드(Nucleo) 동작** | — | ❌ |
| VHDL 테스트벤치 3개 + top 통합 (GHDL 4.1) | ✅ | — |
| Verilog 테스트벤치 3개 + top 통합 (Icarus) | ✅ | — |
| **Vivado 합성 · 타이밍 클로저** | — | ❌ |
| **FPGA 실보드 동작** | — | ❌ |

레지스터 값은 **RM0383 / STM32F411 데이터시트** 기준이며, 핀 제약 파일은 **예시**입니다.
실제 보드에서 이상하면 해당 참고 문서와 대조하세요.

---

## 요구 사항

**MCU 쪽**
- 보드: Nucleo-F411RE (또는 STM32F411 기반 보드 — LED/버튼 핀이 다르면 `board.h` 수정)
- 도구: Arm GNU Toolchain + `make` (로컬), 또는 IDX/Codespaces 환경
- 계정: Google 계정(IDX) 또는 GitHub 계정(Codespaces)

**FPGA 쪽**
- 보드: Basys 3 (100 MHz, 버튼 액티브 하이) 또는 동일 조건의 다른 보드
- 도구: GHDL + GTKWave 또는 Icarus Verilog (시뮬레이션), Vivado (합성)

---

## 문제 해결 요약

| 증상 | 해결 |
|---|---|
| `arm-none-eabi-gcc: command not found` | `dev.nix` 저장 후 Rebuild Environment |
| `stm32f4xx.h: No such file` | `bash setup.sh` 미실행 → `ls third_party/` 확인 |
| `startup_stm32f411xe.s` 없음 | `ls third_party/cmsis_device_f4/Source/Templates/gcc` 로 확인 후 Makefile 수정 |
| `undefined reference to _exit/_sbrk` | `--specs=nosys.specs` 누락 |
| 플래시했는데 LED 불량 | 시작 주소 `0x08000000`, 보드 모델, PA5 핀 확인 |
| UART 글자 깨짐 | 클럭/BRR 불일치, 터미널 115200 확인 |
| 시뮬레이션 `PASS` 안 나옴 | `$finish`/`done` 이전 assertion 메시지 확인 |

---

## 참고 자료

- STM32F411 레퍼런스 매뉴얼: **RM0383**
- STM32F411 데이터시트
- CMSIS: <https://github.com/STMicroelectronics/cmsis_core>, <https://github.com/STMicroelectronics/cmsis_device_f4>
- Firebase Studio 마이그레이션: <https://firebase.google.com/docs/studio/migrating-project>

---

## 알려진 한계 / 보완이 필요한 부분

1. **실보드 검증 부재** — STM32와 FPGA 모두 시뮬레이션·빌드까지만 확인되었습니다.
2. **코드가 문서에만 존재** — 실행용 파일(`Makefile`, `link.ld`, `setup.sh`, `*.vhd`, `*.v`)이
   별도로 저장되어 있지 않아 복사/붙여넣기가 필요합니다.
3. **IDX 신규 사용 불가** — 종료 일정상 새 프로젝트/교육 도입용으로는 부적합합니다.
4. **LICENSE 미지정** — 공개 배포 전 라이선스를 정해야 합니다.
5. 문서 03/04의 "폴더 구성"에는 `tb_top_check.vhd` / `tb_top_check.v`가 빠져 있습니다(부록에 존재).
