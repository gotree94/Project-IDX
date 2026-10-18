# 03. VHDL 구현 — STM32F411 예제를 FPGA 로직으로 옮기기

> 02번 문서의 예제(LED 깜박임, SysTick, 버튼, UART, PWM)를 **FPGA 하드웨어로 구현**합니다.
> 같은 내용의 Verilog 버전은 **04_verilog_implementation.md** 에 있습니다.
> 코드는 VHDL-93 문법을 기준으로 작성했고 `ieee.numeric_std`만 사용합니다.

---

## 1. 소프트웨어 ↔ 하드웨어 대응

MCU에서는 **CPU가 순서대로 실행**하지만, FPGA에서는 **모든 모듈이 클럭마다 동시에 동작**합니다. 같은 기능을 "코드 흐름"이 아니라 "카운터 + 상태 머신"으로 표현하게 됩니다.

| STM32 (02번 문서) | FPGA 모듈 | 구현 방식 |
|---|---|---|
| 예제 1: LED 토글 + `delay_ms(500)` | `blink` | 카운터가 0.5초마다 LED 반전 |
| SysTick 1 ms, `millis()` | `tick_gen` | 분주 카운터 + 1 클럭 펄스 + ms 카운터 |
| 예제 2/6: 버튼, 바운싱, EXTI | `debounce` | 2단 동기화 + 안정 시간 카운터 + 에지 펄스 |
| 예제 3: USART2 TX | `uart_tx` | 상태 머신 (IDLE/START/DATA/STOP) |
| 예제 3: USART2 RX | `uart_rx` | 시작 비트 검출 + 비트 중앙 샘플링 |
| 예제 4: TIM2 PWM | `pwm`, `breathing` | PSC/ARR/CCR과 같은 구조의 카운터·비교기 |
| 예제 5: ADC | — | 벤더 IP(예: Xilinx XADC)가 필요해 제외 |
| 예제 7: PLL | — | MMCM/PLL IP가 필요해 제외 (보드 클럭을 그대로 사용) |

용어 대응: `PSC`(분주) = 클럭 인에이블 카운터, `ARR`(주기) = 카운터 최댓값, `CCR`(듀티) = 비교 레지스터, `TXE/RXNE` = `busy`/`valid` 신호.

---

## 2. 설계 규칙 (이 문서 전체에 적용)

1. **단일 클럭, 동기 설계**: 모든 레지스터는 `rising_edge(clk)`에서만 갱신합니다.
2. **외부 입력은 반드시 동기화**: 버튼, UART RX 같은 비동기 신호는 2단 플립플롭으로 받습니다 (메타스터빌리티 방지).
3. **클럭 가정**: 기본 `CLK_HZ = 100_000_000` (100 MHz). 보드가 다르면 generic만 바꿉니다.
4. **리셋**: 동기 리셋, 액티브 하이(`rst = '1'`).
5. **펄스 신호**: `tick`, `press_pulse`, `valid` 는 **1 클럭 폭** 펄스입니다.

---

## 3. 폴더 구성

```
vhdl/
├─ blink.vhd
├─ tick_gen.vhd
├─ debounce.vhd
├─ uart_tx.vhd
├─ uart_rx.vhd
├─ pwm.vhd
├─ breathing.vhd
├─ top.vhd
├─ tb_blink.vhd
├─ tb_debounce.vhd
├─ tb_uart_loop.vhd
└─ basys3.xdc        # 예시 제약 파일 (보드별로 수정)
```

---

## 4. 모듈

### 4-1. blink — LED 깜박임 (예제 1)

STM32의 `LED_TOGGLE(); delay_ms(500);` 에 해당합니다.

**파일:** `blink.vhd`

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity blink is
  generic (
    CLK_HZ : positive := 100_000_000
  );
  port (
    clk : in  std_logic;
    rst : in  std_logic;
    led : out std_logic
  );
end entity blink;

architecture rtl of blink is
  constant HALF  : positive := CLK_HZ / 2;          -- 0.5 초에 해당하는 클럭 수
  signal   cnt   : integer range 0 to HALF - 1 := 0;
  signal   led_r : std_logic := '0';
begin

  process (clk)
  begin
    if rising_edge(clk) then
      if rst = '1' then
        cnt   <= 0;
        led_r <= '0';
      elsif cnt = HALF - 1 then
        cnt   <= 0;
        led_r <= not led_r;
      else
        cnt <= cnt + 1;
      end if;
    end if;
  end process;

  led <= led_r;

end architecture rtl;
```

### 4-2. tick_gen — 1 ms 틱 (SysTick, `millis()`)

**파일:** `tick_gen.vhd`

```vhdl
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

entity tick_gen is
  generic (
    CLK_HZ  : positive := 100_000_000;
    TICK_HZ : positive := 1000
  );
  port (
    clk      : in  std_logic;
    rst      : in  std_logic;
    tick     : out std_logic;                      -- 1 클럭 폭 펄스 (SysTick 인터럽트에 해당)
    ms_count : out unsigned(31 downto 0)           -- millis() 에 해당
  );
end entity tick_gen;

architecture rtl of tick_gen is
  constant DIV    : positive := CLK_HZ / TICK_HZ;
  signal   cnt    : integer range 0 to DIV - 1 := 0;
  signal   tick_r : std_logic := '0';
  signal   ms_r   : unsigned(31 downto 0) := (others => '0');
begin

  process (clk)
  begin
    if rising_edge(clk) then
      tick_r <= '0';                               -- 기본값: 펄스 아님
      if rst = '1' then
        cnt  <= 0;
        ms_r <= (others => '0');
      elsif cnt = DIV - 1 then
        cnt    <= 0;
        tick_r <= '1';
        ms_r   <= ms_r + 1;
      else
        cnt <= cnt + 1;
      end if;
    end if;
  end process;

  tick     <= tick_r;
  ms_count <= ms_r;

end architecture rtl;
```

### 4-3. debounce — 버튼 처리 (예제 2, 6)

- 2단 플립플롭으로 입력을 클럭에 동기화합니다.
- 입력이 `STABLE_MS` 동안 같은 값으로 유지되어야 새 상태로 인정합니다.
- 인정되는 순간 `press_pulse`(눌림) 또는 `release_pulse`(뗌)를 1 클럭 동안 냅니다. 이것이 EXTI 인터럽트의 하드웨어 버전입니다.
- `ACTIVE_LOW => true`이면 Nucleo 버튼처럼 "누르면 0"인 입력을 처리합니다.

**파일:** `debounce.vhd`

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity debounce is
  generic (
    CLK_HZ     : positive := 100_000_000;
    STABLE_MS  : positive := 20;
    ACTIVE_LOW : boolean  := true
  );
  port (
    clk           : in  std_logic;
    rst           : in  std_logic;
    btn_in        : in  std_logic;
    btn_state     : out std_logic;     -- 디바운스된 상태 (눌림 = '1')
    press_pulse   : out std_logic;     -- 눌린 순간 1 클럭
    release_pulse : out std_logic      -- 뗀 순간 1 클럭
  );
end entity debounce;

architecture rtl of debounce is

  function idle_level (al : boolean) return std_logic is
  begin
    if al then
      return '1';
    else
      return '0';
    end if;
  end function;

  constant LIMIT    : positive  := (CLK_HZ / 1000) * STABLE_MS;
  constant IDLE_LVL : std_logic := idle_level(ACTIVE_LOW);

  signal s0, s1  : std_logic := IDLE_LVL;           -- 2단 동기화
  signal raw     : std_logic;                       -- 눌림 = '1' 로 정규화한 값
  signal state_r : std_logic := '0';
  signal cnt     : integer range 0 to LIMIT - 1 := 0;

begin

  raw <= not s1 when ACTIVE_LOW else s1;

  process (clk)
  begin
    if rising_edge(clk) then
      s0 <= btn_in;
      s1 <= s0;
      press_pulse   <= '0';
      release_pulse <= '0';

      if rst = '1' then
        s0      <= IDLE_LVL;
        s1      <= IDLE_LVL;
        state_r <= '0';
        cnt     <= 0;
      elsif raw = state_r then
        cnt <= 0;                                   -- 변화 없음 → 카운터 초기화
      elsif cnt = LIMIT - 1 then
        cnt     <= 0;                               -- 충분히 안정됨 → 상태 확정
        state_r <= raw;
        if raw = '1' then
          press_pulse <= '1';
        else
          release_pulse <= '1';
        end if;
      else
        cnt <= cnt + 1;
      end if;
    end if;
  end process;

  btn_state <= state_r;

end architecture rtl;
```

### 4-4. uart_tx — USART 송신 (예제 3)

- 8N1 프레임: 시작 비트(0) → 데이터 8비트(LSB 먼저) → 정지 비트(1).
- 사용법: `busy = '0'` 일 때 `data`를 놓고 `start`를 1 클럭 동안 `'1'`로 만듭니다.
- `CPB`(Clocks Per Bit) = `CLK_HZ / BAUD`. 100 MHz, 115200 baud이면 868 클럭입니다 (정수 나눗셈 오차는 약 0.01%).

**파일:** `uart_tx.vhd`

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity uart_tx is
  generic (
    CLK_HZ : positive := 100_000_000;
    BAUD   : positive := 115_200
  );
  port (
    clk   : in  std_logic;
    rst   : in  std_logic;
    start : in  std_logic;                      -- 1 클럭 펄스: 송신 시작
    data  : in  std_logic_vector(7 downto 0);
    tx    : out std_logic;
    busy  : out std_logic
  );
end entity uart_tx;

architecture rtl of uart_tx is
  constant CPB : positive := CLK_HZ / BAUD;

  type state_t is (S_IDLE, S_START, S_DATA, S_STOP);
  signal state   : state_t := S_IDLE;
  signal clk_cnt : integer range 0 to CPB - 1 := 0;
  signal bit_idx : integer range 0 to 7 := 0;
  signal shreg   : std_logic_vector(7 downto 0) := (others => '0');
  signal tx_r    : std_logic := '1';
begin

  busy <= '0' when state = S_IDLE else '1';
  tx   <= tx_r;

  process (clk)
  begin
    if rising_edge(clk) then
      if rst = '1' then
        state   <= S_IDLE;
        tx_r    <= '1';
        clk_cnt <= 0;
        bit_idx <= 0;
      else
        case state is

          when S_IDLE =>
            tx_r    <= '1';
            clk_cnt <= 0;
            bit_idx <= 0;
            if start = '1' then
              shreg <= data;
              state <= S_START;
            end if;

          when S_START =>
            tx_r <= '0';
            if clk_cnt = CPB - 1 then
              clk_cnt <= 0;
              state   <= S_DATA;
            else
              clk_cnt <= clk_cnt + 1;
            end if;

          when S_DATA =>
            tx_r <= shreg(bit_idx);
            if clk_cnt = CPB - 1 then
              clk_cnt <= 0;
              if bit_idx = 7 then
                state <= S_STOP;
              else
                bit_idx <= bit_idx + 1;
              end if;
            else
              clk_cnt <= clk_cnt + 1;
            end if;

          when S_STOP =>
            tx_r <= '1';
            if clk_cnt = CPB - 1 then
              clk_cnt <= 0;
              state   <= S_IDLE;
            else
              clk_cnt <= clk_cnt + 1;
            end if;

        end case;
      end if;
    end if;
  end process;

end architecture rtl;
```

### 4-5. uart_rx — USART 수신 (예제 3)

- 2단 동기화 후 시작 비트(하강)를 검출합니다.
- **반 비트 지연**으로 시작 비트의 중앙에서 한 번 더 확인하고, 이후 비트마다 한 비트 시간(`CPB`)씩 기다려 **각 비트의 중앙**에서 샘플링합니다.
- 정지 비트가 `'1'`일 때만 `valid`를 1 클럭 냅니다. (프레이밍 오류면 버립니다.)

**파일:** `uart_rx.vhd`

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity uart_rx is
  generic (
    CLK_HZ : positive := 100_000_000;
    BAUD   : positive := 115_200
  );
  port (
    clk   : in  std_logic;
    rst   : in  std_logic;
    rx    : in  std_logic;
    data  : out std_logic_vector(7 downto 0);
    valid : out std_logic                       -- 1 클럭 펄스: 한 바이트 수신 완료
  );
end entity uart_rx;

architecture rtl of uart_rx is
  constant CPB  : positive := CLK_HZ / BAUD;
  constant HALF : positive := CPB / 2;

  type state_t is (S_IDLE, S_START, S_DATA, S_STOP);
  signal state   : state_t := S_IDLE;
  signal rx0, rx1 : std_logic := '1';                -- 2단 동기화
  signal clk_cnt : integer range 0 to CPB - 1 := 0;
  signal bit_idx : integer range 0 to 7 := 0;
  signal shreg   : std_logic_vector(7 downto 0) := (others => '0');
  signal data_r  : std_logic_vector(7 downto 0) := (others => '0');
  signal valid_r : std_logic := '0';
begin

  data  <= data_r;
  valid <= valid_r;

  process (clk)
  begin
    if rising_edge(clk) then
      rx0     <= rx;
      rx1     <= rx0;
      valid_r <= '0';

      if rst = '1' then
        rx0     <= '1';
        rx1     <= '1';
        state   <= S_IDLE;
        clk_cnt <= 0;
        bit_idx <= 0;
      else
        case state is

          when S_IDLE =>
            clk_cnt <= 0;
            bit_idx <= 0;
            if rx1 = '0' then                       -- 시작 비트 후보
              state <= S_START;
            end if;

          when S_START =>                           -- 시작 비트 중앙까지 대기
            if clk_cnt = HALF - 1 then
              clk_cnt <= 0;
              if rx1 = '0' then
                state <= S_DATA;                    -- 진짜 시작 비트
              else
                state <= S_IDLE;                    -- 글리치였음
              end if;
            else
              clk_cnt <= clk_cnt + 1;
            end if;

          when S_DATA =>                            -- 각 비트 중앙에서 샘플링
            if clk_cnt = CPB - 1 then
              clk_cnt <= 0;
              shreg   <= rx1 & shreg(7 downto 1);   -- LSB 먼저 들어오므로 오른쪽으로 밀기
              if bit_idx = 7 then
                state <= S_STOP;
              else
                bit_idx <= bit_idx + 1;
              end if;
            else
              clk_cnt <= clk_cnt + 1;
            end if;

          when S_STOP =>
            if clk_cnt = CPB - 1 then
              clk_cnt <= 0;
              state   <= S_IDLE;
              if rx1 = '1' then                     -- 정지 비트 확인
                data_r  <= shreg;
                valid_r <= '1';
              end if;
            else
              clk_cnt <= clk_cnt + 1;
            end if;

        end case;
      end if;
    end if;
  end process;

end architecture rtl;
```

### 4-6. pwm — 타이머 PWM (예제 4)

TIM2의 `PSC`, `ARR`, `CCR1`과 같은 구조입니다. **PWM 모드 1**: `카운터 < duty` 이면 High.

**파일:** `pwm.vhd`

```vhdl
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

entity pwm is
  generic (
    W : positive := 10
  );
  port (
    clk     : in  std_logic;
    rst     : in  std_logic;
    psc     : in  unsigned(15 downto 0);        -- 분주값: 카운터 클럭 = clk / (psc+1)
    period  : in  unsigned(W - 1 downto 0);     -- ARR: 주기 = period+1 카운트
    duty    : in  unsigned(W - 1 downto 0);     -- CCR: 비교값
    pwm_out : out std_logic
  );
end entity pwm;

architecture rtl of pwm is
  signal pcnt  : unsigned(15 downto 0)    := (others => '0');
  signal cnt   : unsigned(W - 1 downto 0) := (others => '0');
  signal out_r : std_logic := '0';
begin

  process (clk)
  begin
    if rising_edge(clk) then
      if rst = '1' then
        pcnt  <= (others => '0');
        cnt   <= (others => '0');
        out_r <= '0';
      elsif pcnt >= psc then                    -- 분주 카운터가 한 바퀴 → 카운터 한 칸 전진
        pcnt <= (others => '0');
        if cnt >= period then
          cnt <= (others => '0');
        else
          cnt <= cnt + 1;
        end if;
        if cnt < duty then
          out_r <= '1';
        else
          out_r <= '0';
        end if;
      else
        pcnt <= pcnt + 1;
      end if;
    end if;
  end process;

  pwm_out <= out_r;

end architecture rtl;
```

### 4-7. breathing — 듀티 자동 증감 (예제 4의 main 루프)

`delay_ms(1)` 마다 `duty`를 ±1 하던 루프를 `tick` 펄스로 구동합니다.

**파일:** `breathing.vhd`

```vhdl
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

entity breathing is
  generic (
    W : positive := 10
  );
  port (
    clk      : in  std_logic;
    rst      : in  std_logic;
    tick     : in  std_logic;                   -- 1 ms 틱
    max_duty : in  unsigned(W - 1 downto 0);
    duty     : out unsigned(W - 1 downto 0)
  );
end entity breathing;

architecture rtl of breathing is
  signal duty_r : unsigned(W - 1 downto 0) := (others => '0');
  signal down   : std_logic := '0';             -- '0' 증가, '1' 감소
begin

  process (clk)
  begin
    if rising_edge(clk) then
      if rst = '1' then
        duty_r <= (others => '0');
        down   <= '0';
      elsif tick = '1' then
        if down = '0' then
          if duty_r >= max_duty then
            down <= '1';
          else
            duty_r <= duty_r + 1;
          end if;
        else
          if duty_r = 0 then
            down <= '0';
          else
            duty_r <= duty_r - 1;
          end if;
        end if;
      end if;
    end if;
  end process;

  duty <= duty_r;

end architecture rtl;
```

### 4-8. top — 통합

| 출력 | 동작 | 대응하는 STM32 예제 |
|---|---|---|
| `led0` | 0.5초 간격 깜박임 | 예제 1 |
| `led1` | 버튼을 누를 때마다 토글 | 예제 2 / 6 |
| `led2` | 숨쉬기 PWM (1 kHz, 2초 왕복) | 예제 4 |
| UART | 받은 문자를 그대로 돌려보냄 (에코) | 예제 3 |

버튼 극성은 보드마다 다릅니다. Basys 3의 버튼은 누르면 High라서 `ACTIVE_LOW => false`로 두었습니다.

**파일:** `top.vhd`

```vhdl
library ieee;
use ieee.std_logic_1164.all;
use ieee.numeric_std.all;

entity top is
  generic (
    CLK_HZ : positive := 100_000_000
  );
  port (
    clk         : in  std_logic;
    btn         : in  std_logic;
    uart_rx_pin : in  std_logic;
    uart_tx_pin : out std_logic;
    led0        : out std_logic;
    led1        : out std_logic;
    led2        : out std_logic
  );
end entity top;

architecture rtl of top is

  -- 파워온 리셋
  signal por : unsigned(3 downto 0) := (others => '1');
  signal rst : std_logic;

  signal tick        : std_logic;
  signal press       : std_logic;
  signal release_p   : std_logic;
  signal btn_state   : std_logic;
  signal led1_r      : std_logic := '0';

  signal rx_data     : std_logic_vector(7 downto 0);
  signal rx_valid    : std_logic;
  signal tx_data     : std_logic_vector(7 downto 0) := (others => '0');
  signal tx_start    : std_logic := '0';
  signal tx_busy     : std_logic;

  signal duty        : unsigned(9 downto 0);
  signal psc         : unsigned(15 downto 0);

begin

  ------------------------------------------------------------------
  -- 파워온 리셋: 전원 인가 후 15 클럭 동안 rst = '1'
  ------------------------------------------------------------------
  process (clk)
  begin
    if rising_edge(clk) then
      if por /= 0 then
        por <= por - 1;
      end if;
    end if;
  end process;
  rst <= '1' when por /= 0 else '0';

  ------------------------------------------------------------------
  -- 예제 1: LED 깜박임
  ------------------------------------------------------------------
  u_blink : entity work.blink
    generic map (CLK_HZ => CLK_HZ)
    port map (clk => clk, rst => rst, led => led0);

  ------------------------------------------------------------------
  -- SysTick 대응: 1 ms 틱
  ------------------------------------------------------------------
  u_tick : entity work.tick_gen
    generic map (CLK_HZ => CLK_HZ, TICK_HZ => 1000)
    port map (clk => clk, rst => rst, tick => tick, ms_count => open);

  ------------------------------------------------------------------
  -- 예제 2/6: 버튼 → LED1 토글
  ------------------------------------------------------------------
  u_db : entity work.debounce
    generic map (CLK_HZ => CLK_HZ, STABLE_MS => 20, ACTIVE_LOW => false)
    port map (clk => clk, rst => rst, btn_in => btn,
              btn_state => btn_state,
              press_pulse => press, release_pulse => release_p);

  process (clk)
  begin
    if rising_edge(clk) then
      if rst = '1' then
        led1_r <= '0';
      elsif press = '1' then
        led1_r <= not led1_r;
      end if;
    end if;
  end process;
  led1 <= led1_r;

  ------------------------------------------------------------------
  -- 예제 4: 숨쉬기 PWM (1 MHz 카운터, 주기 1000 → 1 kHz)
  ------------------------------------------------------------------
  psc <= to_unsigned(CLK_HZ / 1_000_000 - 1, 16);

  u_breath : entity work.breathing
    generic map (W => 10)
    port map (clk => clk, rst => rst, tick => tick,
              max_duty => to_unsigned(999, 10), duty => duty);

  u_pwm : entity work.pwm
    generic map (W => 10)
    port map (clk => clk, rst => rst, psc => psc,
              period => to_unsigned(999, 10), duty => duty, pwm_out => led2);

  ------------------------------------------------------------------
  -- 예제 3: UART 에코
  ------------------------------------------------------------------
  u_rx : entity work.uart_rx
    generic map (CLK_HZ => CLK_HZ, BAUD => 115_200)
    port map (clk => clk, rst => rst, rx => uart_rx_pin,
              data => rx_data, valid => rx_valid);

  u_tx : entity work.uart_tx
    generic map (CLK_HZ => CLK_HZ, BAUD => 115_200)
    port map (clk => clk, rst => rst, start => tx_start,
              data => tx_data, tx => uart_tx_pin, busy => tx_busy);

  process (clk)
  begin
    if rising_edge(clk) then
      tx_start <= '0';
      if rx_valid = '1' and tx_busy = '0' then
        tx_data  <= rx_data;
        tx_start <= '1';
      end if;
    end if;
  end process;

end architecture rtl;
```

---

## 5. 테스트벤치

테스트벤치는 클럭 주기를 10 ns(100 MHz)로 두되, 시뮬레이션이 길어지지 않도록 **generic을 작게 줄여서** 검증합니다. `done` 신호로 클럭을 멈춰 시뮬레이션이 자연스럽게 끝나게 했습니다.

### 5-1. tb_blink

**파일:** `tb_blink.vhd`

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity tb_blink is
end entity tb_blink;

architecture sim of tb_blink is
  signal clk  : std_logic := '0';
  signal rst  : std_logic := '1';
  signal led  : std_logic;
  signal done : boolean := false;
begin

  clk <= not clk after 5 ns when not done;

  -- CLK_HZ = 1000 → 반주기 500 클럭 = 5 us (시뮬레이션용 축소)
  dut : entity work.blink
    generic map (CLK_HZ => 1000)
    port map (clk => clk, rst => rst, led => led);

  stim : process
  begin
    wait for 50 ns;
    rst <= '0';

    wait for 5.2 us;
    assert led = '1' report "tb_blink: led must be '1' after the first toggle" severity error;

    wait for 5 us;
    assert led = '0' report "tb_blink: led must be '0' after the second toggle" severity error;

    report "tb_blink: finished (no assertion errors above = PASS)" severity note;
    done <= true;
    wait;
  end process;

end architecture sim;
```

### 5-2. tb_debounce

바운싱(짧은 떨림)은 무시하고, 안정된 눌림/뗌은 각각 정확히 한 번씩만 펄스로 나오는지 확인합니다.

**파일:** `tb_debounce.vhd`

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity tb_debounce is
end entity tb_debounce;

architecture sim of tb_debounce is
  signal clk     : std_logic := '0';
  signal rst     : std_logic := '1';
  signal btn     : std_logic := '1';              -- 액티브 로우: 평소 High
  signal state   : std_logic;
  signal pp, rp  : std_logic;
  signal done    : boolean := false;
  signal presses, releases : integer := 0;
begin

  clk <= not clk after 5 ns when not done;

  -- CLK_HZ=1000, STABLE_MS=20 → 안정 시간 = 20 클럭 = 200 ns
  dut : entity work.debounce
    generic map (CLK_HZ => 1000, STABLE_MS => 20, ACTIVE_LOW => true)
    port map (clk => clk, rst => rst, btn_in => btn,
              btn_state => state, press_pulse => pp, release_pulse => rp);

  counter : process (clk)
  begin
    if rising_edge(clk) then
      if pp = '1' then presses  <= presses  + 1; end if;
      if rp = '1' then releases <= releases + 1; end if;
    end if;
  end process;

  stim : process
  begin
    wait for 40 ns;
    rst <= '0';

    -- 눌림 쪽 바운싱: 30 ns 짜리 떨림 3번 (안정 시간보다 훨씬 짧음)
    for i in 1 to 3 loop
      wait for 30 ns;  btn <= '0';
      wait for 30 ns;  btn <= '1';
    end loop;

    wait for 50 ns;  btn <= '0';                  -- 실제 눌림
    wait for 400 ns;                              -- 안정 시간(200 ns)보다 길게 유지

    -- 뗌 쪽 바운싱
    for i in 1 to 3 loop
      wait for 30 ns;  btn <= '1';
      wait for 30 ns;  btn <= '0';
    end loop;

    wait for 50 ns;  btn <= '1';                  -- 실제 뗌
    wait for 400 ns;

    assert presses  = 1 report "tb_debounce: press pulse must occur exactly once" severity error;
    assert releases = 1 report "tb_debounce: release pulse must occur exactly once" severity error;
    assert state    = '0' report "tb_debounce: final state must be released" severity error;

    report "tb_debounce: finished (no assertion errors above = PASS)" severity note;
    done <= true;
    wait;
  end process;

end architecture sim;
```

### 5-3. tb_uart_loop — TX → RX 루프백

`uart_tx`의 출력을 `uart_rx`의 입력에 직접 연결해 여러 바이트가 정확히 돌아오는지 검사합니다. 100 MHz 클럭, `BAUD = 10_000_000` 으로 한 비트를 10 클럭으로 줄여 빠르게 시뮬레이션합니다.

**파일:** `tb_uart_loop.vhd`

```vhdl
library ieee;
use ieee.std_logic_1164.all;

entity tb_uart_loop is
end entity tb_uart_loop;

architecture sim of tb_uart_loop is
  constant CLK_HZ : positive := 100_000_000;
  constant BAUD   : positive := 10_000_000;       -- 한 비트 = 10 클럭

  signal clk    : std_logic := '0';
  signal rst    : std_logic := '1';
  signal start  : std_logic := '0';
  signal din    : std_logic_vector(7 downto 0) := (others => '0');
  signal dout   : std_logic_vector(7 downto 0);
  signal line_s : std_logic;
  signal busy   : std_logic;
  signal valid  : std_logic;
  signal done   : boolean := false;
begin

  clk <= not clk after 5 ns when not done;

  u_tx : entity work.uart_tx
    generic map (CLK_HZ => CLK_HZ, BAUD => BAUD)
    port map (clk => clk, rst => rst, start => start,
              data => din, tx => line_s, busy => busy);

  u_rx : entity work.uart_rx
    generic map (CLK_HZ => CLK_HZ, BAUD => BAUD)
    port map (clk => clk, rst => rst, rx => line_s,
              data => dout, valid => valid);

  stim : process
    variable errors : integer := 0;

    procedure send_and_check (b : in std_logic_vector(7 downto 0)) is
    begin
      wait until rising_edge(clk);
      din   <= b;
      start <= '1';
      wait until rising_edge(clk);
      start <= '0';

      wait until valid = '1';                     -- 수신 완료 펄스 대기
      wait for 1 ns;
      if dout /= b then
        errors := errors + 1;
        report "tb_uart_loop: data mismatch" severity error;
      end if;

      wait until busy = '0';                      -- 송신이 완전히 끝날 때까지
    end procedure;

  begin
    wait for 100 ns;
    rst <= '0';
    wait for 50 ns;

    send_and_check(x"55");
    send_and_check(x"A3");
    send_and_check(x"00");
    send_and_check(x"FF");
    send_and_check(x"3C");

    if errors = 0 then
      report "tb_uart_loop: PASS" severity note;
    else
      report "tb_uart_loop: FAIL" severity error;
    end if;

    done <= true;
    wait;
  end process;

end architecture sim;
```

---

## 6. 시뮬레이션 방법

### 6-1. GHDL + GTKWave (오픈소스, 클라우드/로컬 모두 가능)

```bash
# 예: UART 루프백
ghdl -a --std=08 uart_tx.vhd uart_rx.vhd tb_uart_loop.vhd
ghdl -e --std=08 tb_uart_loop
ghdl -r --std=08 tb_uart_loop --wave=uart.ghw
gtkwave uart.ghw            # 로컬에서 파형 확인

# 디바운스
ghdl -a --std=08 debounce.vhd tb_debounce.vhd
ghdl -e --std=08 tb_debounce
ghdl -r --std=08 tb_debounce

# 블링크
ghdl -a --std=08 blink.vhd tb_blink.vhd
ghdl -e --std=08 tb_blink
ghdl -r --std=08 tb_blink
```

`PASS`(uart) 또는 `finished`(blink, debounce) 메시지가 나오고 `error`가 없으면 통과입니다.

> **VHDL 문자열 주의**: VHDL의 문자열 리터럴(`report "..."`)에는 한글 같은 비ASCII 문자를 쓸 수 없어 테스트벤치 메시지는 영어로 작성했습니다. 주석의 한글은 `--std=08`에서는 문제없지만, 도구에 따라 `--std=93` 등에서 오류가 나면 GHDL은 `-C` 옵션을 쓰거나 주석을 영어로 바꾸세요.

### 6-2. Vivado

1. 프로젝트 생성 → Add Sources에서 `*.vhd` 추가 (`tb_*.vhd`는 **Simulation Sources**로 추가)
2. Run Simulation → Run Behavioral Simulation
3. 테스트벤치가 `done`으로 클럭을 멈추므로 `Run All`로 끝까지 실행할 수 있습니다.

---

## 7. 합성 및 보드 적용 (Basys 3 예시)

핀 이름은 **예시이며, 사용하는 보드의 매뉴얼로 반드시 확인**하세요. UART 핀 이름은 FPGA 기준 방향입니다 (`uart_rx_pin` = FPGA로 들어오는 신호).

**파일:** `basys3.xdc`

```tcl
## 클럭 100 MHz
set_property PACKAGE_PIN W5 [get_ports clk]
create_clock -period 10.000 -name sys_clk [get_ports clk]

## LED
set_property PACKAGE_PIN U16 [get_ports led0]
set_property PACKAGE_PIN E19 [get_ports led1]
set_property PACKAGE_PIN U19 [get_ports led2]

## 버튼 (중앙)
set_property PACKAGE_PIN U18 [get_ports btn]

## USB-UART
set_property PACKAGE_PIN B18 [get_ports uart_rx_pin]
set_property PACKAGE_PIN A18 [get_ports uart_tx_pin]

set_property IOSTANDARD LVCMOS33 [get_ports {clk btn led0 led1 led2 uart_rx_pin uart_tx_pin}]
```

### 확인 순서
1. 비트스트림 생성 후 보드에 다운로드
2. `led0`이 0.5초 간격으로 깜박이는지 확인
3. 버튼을 누를 때마다 `led1`이 토글되는지 확인
4. `led2`가 2초 주기로 밝아졌다 어두워지는지 확인
5. 시리얼 터미널(115200 8N1)에서 글자를 입력하면 그대로 돌아오는지 확인

---

## 8. 설계 포인트 정리

| 주제 | 설명 |
|---|---|
| 정수 카운터 vs `unsigned` | 카운터 한계값이 generic 상수이면 `integer range`가 읽기 쉽고 합성도 잘 된다. 비트 연산이 필요하면 `unsigned` |
| 펄스 신호 설계 | 기본값을 `'0'`으로 먼저 대입하고, 조건이 맞을 때만 `'1'`을 덮어쓴다 (프로세스에서 마지막 대입이 유효) |
| 2단 동기화 | 비동기 입력은 반드시 `s0 → s1` 2단 플립플롭을 거친 `s1`만 사용 |
| 비트 중앙 샘플링 | UART 수신은 에지가 아닌 **중앙**에서 읽어야 클럭 오차에 강함 |
| 예약어 주의 | VHDL에서 `out`, `release`, `wait` 등은 예약어라 신호 이름으로 쓸 수 없다 (그래서 `pwm_out`, `release_p`) |

---

## 9. 실습 과제

1. `uart_tx`에 짝수 패리티 비트를 추가하고 `uart_rx`가 패리티 오류를 검출하도록 확장하기
2. `top`에서 문자 `'1'`을 받으면 `led1` ON, `'0'`이면 OFF가 되도록 수정하기 (STM32 예제 3의 실습 과제와 동일)
3. `breathing`의 증가 곡선을 제곱 곡선으로 바꾸기 (곱셈기 사용)
4. `pwm`에 **PWM 모드 2**(CNT ≥ CCR 이면 High) 선택 입력을 추가하기
5. Nucleo 보드의 UART 속도 115200과 FPGA의 `CPB = CLK_HZ / BAUD` 정수 나눗셈이 만드는 **보레이트 오차**를 직접 계산해 보기

---

## 10. 한계와 주의

- **검증 범위**: GHDL 4.1(`--std=08`)에서 `tb_blink`, `tb_debounce`, `tb_uart_loop`가 모두 통과하고, 부록의 `top` 통합 테스트(UART 에코 2바이트, 버튼 토글, blink 4회, PWM)도 통과합니다.
- **검증하지 않은 범위**: Vivado 합성·구현, 타이밍 클로저, 실제 보드 동작. 핀 배치와 클럭 제약은 사용하는 도구·보드에서 직접 확인해야 합니다.
- 보드의 클럭 주파수와 버튼 극성이 다르면 `CLK_HZ`, `ACTIVE_LOW`를 바꾸세요.
- 클럭이 100 MHz가 아닐 때 PWM의 `psc` 계산(`CLK_HZ / 1_000_000 - 1`)은 클럭이 1 MHz의 배수여야 정확합니다.

이전 문서: **02_stm32f411_examples.md** / Verilog 버전: **04_verilog_implementation.md**

---

## 부록. top 통합 테스트벤치

`CLK_HZ`를 11,520,000으로 낮춰 한 비트를 100 클럭, 1 ms를 11,520 클럭으로 만든 뒤 UART 에코, 버튼 토글, blink, PWM을 한 번에 확인합니다. 설계상 약 2.3초 분량이라 실행에 수십 초가 걸릴 수 있습니다. VHDL에는 `fork`가 없으므로 UART 송신(`stim`)과 수신 해석(`mon`)을 별도 프로세스로 나눴습니다.

**파일:** `tb_top_check.vhd`

```vhdl
library ieee; use ieee.std_logic_1164.all;
entity tb_top_check is end entity;
architecture sim of tb_top_check is
  constant CLK_HZ : positive := 11_520_000;   -- 100 clk per bit at 115200, 1 ms = 11520 clk
  signal clk : std_logic := '0'; signal btn : std_logic := '0'; signal rxp : std_logic := '1';
  signal txp, l0, l1, l2 : std_logic; signal done : boolean := false;
  signal got1, got2 : std_logic_vector(7 downto 0) := (others => '0');
  signal sent_all : boolean := false;
begin
  clk <= not clk after 5 ns when not done;
  dut : entity work.top generic map (CLK_HZ => CLK_HZ)
    port map (clk=>clk, btn=>btn, uart_rx_pin=>rxp, uart_tx_pin=>txp, led0=>l0, led1=>l1, led2=>l2);

  -- monitor: decode two bytes from the TX pin
  mon : process
    procedure wait_clks(n : integer) is begin for c in 1 to n loop wait until rising_edge(clk); end loop; end procedure;
    variable r : std_logic_vector(7 downto 0);
  begin
    for n in 1 to 2 loop
      wait until falling_edge(txp);
      wait_clks(150);
      for k in 0 to 7 loop r(k) := txp; wait_clks(100); end loop;
      if n = 1 then got1 <= r; else got2 <= r; end if;
    end loop;
    wait;
  end process;

  stim : process
    variable errors : integer := 0;
    variable l0_tog : integer := 0;
    variable l0_prev : std_logic := '0';
    variable hi, lo : integer := 0;
    procedure wait_clks(n : integer) is begin for c in 1 to n loop wait until rising_edge(clk); end loop; end procedure;
    procedure send(b : std_logic_vector(7 downto 0)) is begin
      rxp <= '0'; wait_clks(100);
      for k in 0 to 7 loop rxp <= b(k); wait_clks(100); end loop;
      rxp <= '1'; wait_clks(100);
    end procedure;
  begin
    wait_clks(50);
    send(x"41"); wait_clks(1500);
    send(x"7E"); wait_clks(1500);
    if got1 /= x"41" then errors := errors + 1; report "echo 1 FAIL" severity error; else report "echo 1 OK"; end if;
    if got2 /= x"7E" then errors := errors + 1; report "echo 2 FAIL" severity error; else report "echo 2 OK"; end if;
    btn <= '1'; wait_clks(11520*40);
    if l1 /= '1' then errors := errors + 1; report "led1 not toggled on press" severity error; else report "led1 toggled OK"; end if;
    btn <= '0'; wait_clks(11520*40);
    if l1 /= '1' then errors := errors + 1; report "led1 changed on release" severity error; else report "led1 stays on release OK"; end if;
    -- 2.2 design-seconds: count led0 toggles and led2 high/low
    l0_prev := l0;
    for c in 1 to 11520*2200 loop
      wait until rising_edge(clk);
      if l0 /= l0_prev then l0_tog := l0_tog + 1; l0_prev := l0; end if;
      if l2 = '1' then hi := hi + 1; else lo := lo + 1; end if;
    end loop;
    report "led0 toggles=" & integer'image(l0_tog) & " (expect ~4), led2 hi=" & integer'image(hi) & " lo=" & integer'image(lo);
    if l0_tog < 3 or l0_tog > 5 then errors := errors + 1; end if;
    if hi = 0 or lo = 0 then errors := errors + 1; end if;
    if errors = 0 then report "TOP CHECK: PASS"; else report "TOP CHECK: FAIL" severity error; end if;
    done <= true; wait;
  end process;
end architecture;
```

```bash
ghdl -a --std=08 blink.vhd tick_gen.vhd debounce.vhd uart_tx.vhd uart_rx.vhd pwm.vhd breathing.vhd top.vhd tb_top_check.vhd
ghdl -e --std=08 tb_top_check
ghdl -r --std=08 tb_top_check
```

`TOP CHECK: PASS`가 나오면 통과입니다.
