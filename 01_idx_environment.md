# 01. Google Project IDX(Firebase Studio)에서 STM32F411 개발하기 — 사용환경 편

> 대상: STM32F411 (Cortex-M4F, 최대 100 MHz, Flash 512 KB / SRAM 128 KB)
> 기준 보드: **NUCLEO-F411RE** (온보드 ST-LINK 포함)
> 작성 기준일: 2026-10-18

---

## 0. 먼저 알아둘 점 — IDX는 현재 종료 절차 중입니다

| 날짜 | 내용 |
|---|---|
| 2024-05-14 | Project IDX 오픈 베타 |
| 2025-04-15 | **Firebase Studio**로 이름 변경 (IDX 흡수) |
| 2026-03-19 | Google이 Firebase Studio 종료 발표 |
| 2026-06-22 | 신규 가입 및 신규 워크스페이스 생성 중단 |
| **2027-03-22** | **서비스 종료 예정** |

- 공식 마이그레이션 안내: <https://firebase.google.com/docs/studio/migrating-project>
- Google은 다른 개발 도구(Antigravity, Google AI Studio 등)로의 이전을 안내하고 있습니다.

**이 문서에서의 의미**

1. 이미 만들어 둔 IDX 워크스페이스가 있으면 종료 전까지는 사용할 수 있습니다.
2. 새 워크스페이스는 만들 수 없으므로, 교육과정 등 **새로 도입하는 용도로는 적합하지 않습니다**.
3. 그래서 이 문서는 **(A) IDX/Firebase Studio 기준 설정**과 **(B) 같은 환경을 GitHub Codespaces·로컬 VS Code에서 재현하는 방법**을 함께 정리했습니다. 02번 문서의 예제 코드는 어느 환경에서든 그대로 동작합니다.

> 서비스 상태는 계속 바뀔 수 있으니 사용 전에 위 공식 링크에서 최신 상태를 확인하세요.

---

## 1. IDX란 무엇인가

| 항목 | 내용 |
|---|---|
| 형태 | 브라우저 기반 클라우드 IDE (VS Code 계열인 Code OSS 기반) |
| 실행 위치 | Google Cloud 위의 **개별 VM** (워크스페이스 단위) |
| 환경 정의 | **Nix** 기반 설정 파일 `.idx/dev.nix` |
| 확장 | Open VSX 레지스트리에 있는 확장만 설치 가능 (MS 마켓플레이스 아님) |
| 터미널 | 워크스페이스 VM의 Linux 셸 |
| 한계 | **로컬 PC의 USB 장치(ST-LINK 등)에 직접 접근 불가** |

핵심은 마지막 줄입니다. 임베디드 개발에서 IDX는 **"클라우드에서 빌드 → 파일을 내려받아 로컬에서 플래시"** 구조가 됩니다.

```
┌─────────── 클라우드 (IDX VM) ───────────┐        ┌──────── 로컬 PC ────────┐
│  편집 → make → build/f411.bin / .elf    │ ─다운로드→ │  Nucleo에 .bin 복사     │
│  arm-none-eabi-gcc, OpenOCD(사용 불가)  │        │  또는 STM32CubeProgrammer │
└─────────────────────────────────────────┘        │  + 시리얼 터미널          │
                                                   └─────────────────────────┘
```

---

## 2. (A) IDX 워크스페이스 설정

### 2-1. 워크스페이스 만들기 (생성이 가능한 계정/기존 워크스페이스 기준)

1. 빈(Blank/Other) 템플릿으로 워크스페이스를 만듭니다.
2. 탐색기에서 `.idx/dev.nix`를 엽니다. 없으면 새로 만듭니다.
3. 아래 `dev.nix`를 붙여 넣고 저장합니다.
4. 환경 재빌드(Rebuild Environment) 안내가 뜨면 수락합니다.

### 2-2. `.idx/dev.nix`

**파일:** `.idx/dev.nix`

```nix
{ pkgs, ... }: {
  # nixpkgs 채널 (사용 가능한 채널은 시점에 따라 다를 수 있음)
  channel = "stable-24.05";

  packages = [
    pkgs.gcc-arm-embedded   # arm-none-eabi-gcc, gdb, objcopy, size ...
    pkgs.gnumake
    pkgs.git
    pkgs.openocd            # 로컬/Codespaces 외 환경에서 디버깅용
    pkgs.stlink             # st-flash (로컬 사용 시)
    pkgs.picocom            # 시리얼 터미널 (로컬 장치가 있는 환경에서)
  ];

  idx = {
    extensions = [
      "marus25.cortex-debug"                       # 로컬 디버깅용
      "llvm-vs-code-extensions.vscode-clangd"      # 코드 자동완성
    ];

    workspace = {
      # 워크스페이스를 처음 만들 때 CMSIS 헤더를 받아옴
      onCreate = {
        setup-cmsis = "bash setup.sh";
      };
    };
  };
}
```

> 패키지 이름·채널은 시점에 따라 바뀔 수 있습니다. 재빌드가 실패하면 오류 메시지의 패키지 이름을 확인하세요.

### 2-3. 알아둘 제약

| 제약 | 대응 |
|---|---|
| `sudo apt` 사용 불가 | 모든 도구는 `dev.nix`의 `packages`로 추가 |
| USB 장치 접근 불가 | 빌드 결과물을 내려받아 로컬에서 플래시 |
| 확장 설치 범위 | Open VSX에 있는 확장만 |
| 서비스 종료 예정 | 중요한 코드는 반드시 Git 원격 저장소에 push |

---

## 3. (B) 같은 환경을 다른 곳에서 재현하기

### 3-1. GitHub Codespaces / Dev Container

**파일:** `.devcontainer/devcontainer.json`

```json
{
  "name": "stm32f411-dev",
  "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
  "postCreateCommand": "sudo apt-get update && sudo apt-get install -y gcc-arm-none-eabi binutils-arm-none-eabi libnewlib-arm-none-eabi make git openocd stlink-tools && bash setup.sh",
  "customizations": {
    "vscode": {
      "extensions": [
        "marus25.cortex-debug",
        "llvm-vs-code-extensions.vscode-clangd"
      ]
    }
  }
}
```

Codespaces도 클라우드 VM이므로 USB 제약은 IDX와 같습니다.

### 3-2. 로컬 PC (가장 단순)

- Windows: STM32CubeIDE 또는 VS Code + Arm GNU Toolchain + `make`
- Linux/macOS: 패키지 관리자로 `gcc-arm-none-eabi`, `make`, `openocd`(또는 `stlink-tools`) 설치
- 로컬에서는 ST-LINK에 바로 연결되므로 OpenOCD + Cortex-Debug 디버깅까지 가능합니다.

---

## 4. 프로젝트 구조

```
f411-lab/
├─ .idx/dev.nix
├─ setup.sh               # CMSIS 헤더 내려받기
├─ Makefile
├─ link.ld                # 링커 스크립트
├─ src/
│  ├─ main.c              # ← 02번 문서의 예제로 교체
│  ├─ board.c / board.h   # LED, SysTick, delay
│  └─ uart.c  / uart.h    # USART2 드라이버
└─ third_party/           # setup.sh가 생성
   ├─ cmsis_core/
   └─ cmsis_device_f4/
```

### 4-1. CMSIS 헤더 받기

**파일:** `setup.sh`

```bash
#!/usr/bin/env bash
set -e
mkdir -p third_party
cd third_party
[ -d cmsis_core ]      || git clone --depth 1 https://github.com/STMicroelectronics/cmsis_core.git
[ -d cmsis_device_f4 ] || git clone --depth 1 https://github.com/STMicroelectronics/cmsis_device_f4.git
echo "CMSIS ready"
```

- `cmsis_core`: Cortex-M4 코어 정의 (`core_cm4.h`, `SysTick_Config`, `NVIC_*`). 헤더 위치는 `third_party/cmsis_core/CMSIS/Core/Include`
- `cmsis_device_f4`: STM32F4 레지스터 정의(`stm32f411xe.h`), **시작 코드(startup)**, `system_stm32f4xx.c`
- 저장소 이름이나 경로가 다르면 `ls third_party/cmsis_device_f4/Source/Templates/gcc` 로 실제 파일명을 확인하세요.

### 4-2. 링커 스크립트

**파일:** `link.ld`

```ld
ENTRY(Reset_Handler)

_estack = ORIGIN(RAM) + LENGTH(RAM);
_Min_Heap_Size  = 0x200;
_Min_Stack_Size = 0x400;

MEMORY
{
  FLASH (rx)  : ORIGIN = 0x08000000, LENGTH = 512K
  RAM   (xrw) : ORIGIN = 0x20000000, LENGTH = 128K
}

SECTIONS
{
  .isr_vector :
  {
    . = ALIGN(4);
    KEEP(*(.isr_vector))
    . = ALIGN(4);
  } >FLASH

  .text :
  {
    . = ALIGN(4);
    *(.text)
    *(.text*)
    *(.glue_7)
    *(.glue_7t)
    *(.eh_frame)
    KEEP(*(.init))
    KEEP(*(.fini))
    . = ALIGN(4);
    _etext = .;
  } >FLASH

  .rodata :
  {
    . = ALIGN(4);
    *(.rodata)
    *(.rodata*)
    . = ALIGN(4);
  } >FLASH

  .ARM.extab : { *(.ARM.extab* .gnu.linkonce.armextab.*) } >FLASH

  .ARM :
  {
    __exidx_start = .;
    *(.ARM.exidx*)
    __exidx_end = .;
  } >FLASH

  .preinit_array :
  {
    PROVIDE_HIDDEN(__preinit_array_start = .);
    KEEP(*(.preinit_array*))
    PROVIDE_HIDDEN(__preinit_array_end = .);
  } >FLASH

  .init_array :
  {
    PROVIDE_HIDDEN(__init_array_start = .);
    KEEP(*(SORT(.init_array.*)))
    KEEP(*(.init_array*))
    PROVIDE_HIDDEN(__init_array_end = .);
  } >FLASH

  .fini_array :
  {
    PROVIDE_HIDDEN(__fini_array_start = .);
    KEEP(*(SORT(.fini_array.*)))
    KEEP(*(.fini_array*))
    PROVIDE_HIDDEN(__fini_array_end = .);
  } >FLASH

  _sidata = LOADADDR(.data);

  .data :
  {
    . = ALIGN(4);
    _sdata = .;
    *(.data)
    *(.data*)
    . = ALIGN(4);
    _edata = .;
  } >RAM AT> FLASH

  .bss :
  {
    . = ALIGN(4);
    _sbss = .;
    __bss_start__ = _sbss;
    *(.bss)
    *(.bss*)
    *(COMMON)
    . = ALIGN(4);
    _ebss = .;
    __bss_end__ = _ebss;
  } >RAM

  ._user_heap_stack :
  {
    . = ALIGN(8);
    PROVIDE(end = .);
    PROVIDE(_end = .);
    . = . + _Min_Heap_Size;
    . = . + _Min_Stack_Size;
    . = ALIGN(8);
  } >RAM

  .ARM.attributes 0 : { *(.ARM.attributes) }
}
```

### 4-3. Makefile

> 레시피 줄 앞의 Tab 대신 `.RECIPEPREFIX := >` 를 사용했습니다. 복사/붙여넣기 중 Tab이 공백으로 바뀌는 사고를 막기 위해서입니다. (GNU make 3.82 이상)

**파일:** `Makefile`

```make
TARGET     := f411
BUILD      := build
CMSIS_CORE := third_party/cmsis_core
CMSIS_DEV  := third_party/cmsis_device_f4

CC      := arm-none-eabi-gcc
OBJCOPY := arm-none-eabi-objcopy
SIZE    := arm-none-eabi-size

CPU     := -mcpu=cortex-m4 -mthumb -mfpu=fpv4-sp-d16 -mfloat-abi=hard
DEFS    := -DSTM32F411xE
INCS    := -I$(CMSIS_CORE)/CMSIS/Core/Include -I$(CMSIS_DEV)/Include -Isrc
CFLAGS  := $(CPU) $(DEFS) $(INCS) -Os -g3 -Wall -ffunction-sections -fdata-sections
ASFLAGS := $(CPU) -g3
LDFLAGS := $(CPU) -T link.ld -Wl,--gc-sections -Wl,-Map=$(BUILD)/$(TARGET).map --specs=nano.specs --specs=nosys.specs

CSRCS := src/main.c src/board.c src/uart.c $(CMSIS_DEV)/Source/Templates/system_stm32f4xx.c
ASRCS := $(CMSIS_DEV)/Source/Templates/gcc/startup_stm32f411xe.s
OBJS  := $(addprefix $(BUILD)/,$(notdir $(CSRCS:.c=.o) $(ASRCS:.s=.o)))

vpath %.c $(sort $(dir $(CSRCS)))
vpath %.s $(sort $(dir $(ASRCS)))

.RECIPEPREFIX := >
.PHONY: all clean flash

all: $(BUILD)/$(TARGET).bin

$(BUILD)/%.o: %.c | $(BUILD)
> $(CC) $(CFLAGS) -c $< -o $@

$(BUILD)/%.o: %.s | $(BUILD)
> $(CC) $(ASFLAGS) -c $< -o $@

$(BUILD)/$(TARGET).elf: $(OBJS) link.ld
> $(CC) $(OBJS) $(LDFLAGS) -o $@
> $(SIZE) $@

$(BUILD)/%.bin: $(BUILD)/%.elf
> $(OBJCOPY) -O binary $< $@

$(BUILD):
> mkdir -p $@

# 로컬 PC에서만 사용 (ST-LINK 연결 필요)
flash: $(BUILD)/$(TARGET).bin
> st-flash --reset write $< 0x8000000

clean:
> rm -rf $(BUILD)
```

`--specs=nano.specs`를 찾지 못한다는 오류가 나면 해당 옵션을 지워도 빌드는 됩니다. 코드 크기만 조금 커집니다.

> 이 `Makefile`과 `link.ld`는 Ubuntu의 `gcc-arm-none-eabi`(13.2) 환경에서 02번 문서의 예제 7개를 실제로 빌드해 확인했습니다. 링크 시 `_close`, `_read` 등이 구현되지 않았다는 newlib 경고가 나올 수 있으며, 이 예제들에서는 해당 함수를 호출하지 않으므로 무시해도 됩니다.

### 4-4. 부팅 흐름 이해하기

```
리셋 → 벡터 테이블[1] = Reset_Handler (startup_stm32f411xe.s)
      → 스택 포인터 = _estack (SRAM 끝)
      → .data 를 Flash(_sidata)에서 SRAM(_sdata~_edata)로 복사
      → .bss(_sbss~_ebss) 를 0으로 초기화
      → SystemInit()        (FPU 활성화 등, system_stm32f4xx.c)
      → __libc_init_array() (C 라이브러리 초기화)
      → main()
```

리셋 직후 클럭은 **HSI 16 MHz**입니다. `SystemCoreClock` 값도 16,000,000으로 시작하며, PLL로 클럭을 바꾸면 직접 갱신해야 합니다 (02번 문서의 예제 7).

---

## 5. 빌드 → 플래시 → 확인

### 5-1. 빌드 (IDX 터미널)

```bash
bash setup.sh      # 최초 1회
make               # build/f411.elf, build/f411.bin 생성
```

`make` 마지막에 `arm-none-eabi-size` 결과(text/data/bss)가 출력되면 정상입니다.

### 5-2. 내려받기

탐색기에서 `build/f411.bin` 우클릭 → **Download**.

### 5-3. 플래시 (로컬 PC)

| 방법 | 절차 |
|---|---|
| **드래그 앤 드롭 (가장 쉬움)** | Nucleo를 USB로 연결 → `NODE_F411RE` 드라이브가 나타남 → `f411.bin` 복사 |
| STM32CubeProgrammer | ST-LINK 선택 → `f411.bin` 시작 주소 `0x08000000`으로 Download |
| `st-flash` | `st-flash --reset write f411.bin 0x8000000` |

### 5-4. 시리얼 모니터 (UART 예제용)

Nucleo의 USB 연결은 ST-LINK 가상 COM 포트를 제공합니다. **115200 / 8N1** 로 접속합니다.

| OS | 포트 예시 |
|---|---|
| Windows | 장치 관리자에서 확인한 `COMx` |
| Linux | `/dev/ttyACM0` |
| macOS | `/dev/tty.usbmodem*` |

---

## 6. 디버깅과 시뮬레이션에 대한 메모

- **SWD 디버깅**(브레이크포인트, 단계 실행)은 USB가 필요하므로 **로컬 PC**에서 OpenOCD + Cortex-Debug로 합니다.
- 클라우드 환경에서 코드 동작만 확인하려면 Renode 같은 에뮬레이터를 쓸 수 있습니다. 다만 제공되는 보드 정의가 F411과 정확히 일치하지 않을 수 있으므로(예: STM32F4 Discovery는 F407 기반), 사용 전에 지원 플랫폼과 주변장치 범위를 확인하세요.

---

## 7. 문제 해결

| 증상 | 원인 / 해결 |
|---|---|
| `arm-none-eabi-gcc: command not found` | `dev.nix` 저장 후 환경 재빌드가 안 됨 → Rebuild 수행 |
| `stm32f4xx.h: No such file` | `setup.sh` 미실행 또는 경로 오류 → `ls third_party/` 확인 |
| `startup_stm32f411xe.s` 없음 | `ls third_party/cmsis_device_f4/Source/Templates/gcc` 로 파일명 확인 후 Makefile `ASRCS` 수정 |
| `undefined reference to _exit/_sbrk` | `--specs=nosys.specs` 누락 |
| 플래시했는데 LED가 안 깜박임 | `.bin` 시작 주소 확인(0x08000000), 보드가 F411RE인지, LED 핀(PA5) 확인 |
| UART 글자가 깨짐 | 클럭 설정과 BRR 값 불일치, 터미널 속도 115200 확인 |

---

## 8. 체크리스트

- [ ] `make` 성공, `build/f411.bin` 생성
- [ ] `.bin` 내려받기 및 Nucleo에 복사
- [ ] LED(LD2) 점멸 확인 → 02번 문서 예제 1 완료
- [ ] 코드를 Git 원격 저장소에 push (IDX 종료 대비)

다음 문서: **02_stm32f411_examples.md**
