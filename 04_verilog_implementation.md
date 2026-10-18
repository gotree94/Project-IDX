# 04. Verilog 구현 — STM32F411 예제를 FPGA 로직으로 옮기기

> 02번 문서의 예제(LED 깜박임, SysTick, 버튼, UART, PWM)를 **FPGA 하드웨어로 구현**합니다.
> 기능과 모듈 이름, 포트 이름은 **03_vhdl_implementation.md** 와 1:1로 대응합니다.
> 코드는 Verilog-2001 문법에 `$clog2`(Verilog-2005)만 추가로 사용했습니다.

---

## 1. 소프트웨어 ↔ 하드웨어 대응

| STM32 (02번 문서) | FPGA 모듈 | 구현 방식 |
|---|---|---|
| 예제 1: LED 토글 + `delay_ms(500)` | `blink` | 카운터가 0.5초마다 LED 반전 |
| SysTick 1 ms, `millis()` | `tick_gen` | 분주 카운터 + 1 클럭 펄스 + ms 카운터 |
| 예제 2/6: 버튼, 바운싱, EXTI | `debounce` | 2단 동기화 + 안정 시간 카운터 + 에지 펄스 |
| 예제 3: USART2 TX | `uart_tx` | 상태 머신 (IDLE/START/DATA/STOP) |
| 예제 3: USART2 RX | `uart_rx` | 시작 비트 검출 + 비트 중앙 샘플링 |
| 예제 4: TIM2 PWM | `pwm`, `breathing` | PSC/ARR/CCR과 같은 구조의 카운터·비교기 |
| 예제 5: ADC | — | 벤더 IP 필요, 제외 |
| 예제 7: PLL | — | MMCM/PLL IP 필요, 제외 |

---

## 2. 설계 규칙

1. **단일 클럭, 동기 설계**: 모든 `always` 블록은 `@(posedge clk)`.
2. **외부 입력은 2단 플립플롭으로 동기화**.
3. **클럭 가정**: 기본 `CLK_HZ = 100_000_000`. 보드에 맞게 파라미터만 변경.
4. **리셋**: 동기 리셋, 액티브 하이(`rst`).
5. **펄스 신호**(`tick`, `press_pulse`, `valid`)는 1 클럭 폭.
6. **비차단 대입(`<=`)** 만 순차 로직에 사용.
7. 포트 이름에 `out` 같은 혼동되는 이름을 피하고 `pwm_out`처럼 명확히 지정 (VHDL 예약어와도 겹치지 않게 맞춤).

---

## 3. 폴더 구성

```
verilog/
├─ blink.v
├─ tick_gen.v
├─ debounce.v
├─ uart_tx.v
├─ uart_rx.v
├─ pwm.v
├─ breathing.v
├─ top.v
├─ tb_blink.v
├─ tb_debounce.v
├─ tb_uart_loop.v
└─ basys3.xdc        # 03번 문서의 제약 파일과 동일 (보드별로 수정)
```

---

## 4. 모듈

### 4-1. blink — LED 깜박임 (예제 1)

**파일:** `blink.v`

```verilog
module blink #(
    parameter CLK_HZ = 100_000_000
)(
    input  wire clk,
    input  wire rst,
    output reg  led
);
    localparam integer HALF = CLK_HZ / 2;           // 0.5 초에 해당하는 클럭 수
    localparam integer W    = $clog2(HALF);

    reg [W-1:0] cnt;

    always @(posedge clk) begin
        if (rst) begin
            cnt <= 0;
            led <= 1'b0;
        end else if (cnt == HALF - 1) begin
            cnt <= 0;
            led <= ~led;
        end else begin
            cnt <= cnt + 1'b1;
        end
    end
endmodule
```

### 4-2. tick_gen — 1 ms 틱 (SysTick, `millis()`)

**파일:** `tick_gen.v`

```verilog
module tick_gen #(
    parameter CLK_HZ  = 100_000_000,
    parameter TICK_HZ = 1000
)(
    input  wire        clk,
    input  wire        rst,
    output reg         tick,          // 1 클럭 폭 펄스 (SysTick 인터럽트에 해당)
    output reg  [31:0] ms_count       // millis() 에 해당
);
    localparam integer DIV = CLK_HZ / TICK_HZ;
    localparam integer W   = $clog2(DIV);

    reg [W-1:0] cnt;

    always @(posedge clk) begin
        if (rst) begin
            cnt      <= 0;
            tick     <= 1'b0;
            ms_count <= 32'd0;
        end else begin
            tick <= 1'b0;                              // 기본값: 펄스 아님
            if (cnt == DIV - 1) begin
                cnt      <= 0;
                tick     <= 1'b1;
                ms_count <= ms_count + 1'b1;
            end else begin
                cnt <= cnt + 1'b1;
            end
        end
    end
endmodule
```

### 4-3. debounce — 버튼 처리 (예제 2, 6)

**파일:** `debounce.v`

```verilog
module debounce #(
    parameter CLK_HZ     = 100_000_000,
    parameter STABLE_MS  = 20,
    parameter ACTIVE_LOW = 1             // 1: 누르면 0 (Nucleo 버튼), 0: 누르면 1
)(
    input  wire clk,
    input  wire rst,
    input  wire btn_in,
    output reg  btn_state,               // 디바운스된 상태 (눌림 = 1)
    output reg  press_pulse,             // 눌린 순간 1 클럭
    output reg  release_pulse            // 뗀 순간 1 클럭
);
    localparam integer LIMIT = (CLK_HZ / 1000) * STABLE_MS;
    localparam integer W     = $clog2(LIMIT + 1);
    localparam         IDLE  = (ACTIVE_LOW != 0) ? 1'b1 : 1'b0;

    reg         s0, s1;                  // 2단 동기화
    reg [W-1:0] cnt;

    wire raw = (ACTIVE_LOW != 0) ? ~s1 : s1;          // 눌림 = 1 로 정규화

    always @(posedge clk) begin
        s0            <= btn_in;
        s1            <= s0;
        press_pulse   <= 1'b0;
        release_pulse <= 1'b0;

        if (rst) begin
            s0        <= IDLE;
            s1        <= IDLE;
            btn_state <= 1'b0;
            cnt       <= 0;
        end else if (raw == btn_state) begin
            cnt <= 0;                                  // 변화 없음 → 카운터 초기화
        end else if (cnt == LIMIT - 1) begin
            cnt       <= 0;                            // 충분히 안정됨 → 상태 확정
            btn_state <= raw;
            if (raw) press_pulse   <= 1'b1;
            else     release_pulse <= 1'b1;
        end else begin
            cnt <= cnt + 1'b1;
        end
    end
endmodule
```

### 4-4. uart_tx — USART 송신 (예제 3)

사용법: `busy == 0` 일 때 `data`를 놓고 `start`를 1 클럭 동안 1로 만듭니다.

**파일:** `uart_tx.v`

```verilog
module uart_tx #(
    parameter CLK_HZ = 100_000_000,
    parameter BAUD   = 115_200
)(
    input  wire       clk,
    input  wire       rst,
    input  wire       start,             // 1 클럭 펄스: 송신 시작
    input  wire [7:0] data,
    output reg        tx,
    output wire       busy
);
    localparam integer CPB = CLK_HZ / BAUD;            // 비트당 클럭 수
    localparam integer CW  = $clog2(CPB);

    localparam [1:0] S_IDLE  = 2'd0,
                     S_START = 2'd1,
                     S_DATA  = 2'd2,
                     S_STOP  = 2'd3;

    reg [1:0]    state;
    reg [CW-1:0] clk_cnt;
    reg [2:0]    bit_idx;
    reg [7:0]    shreg;

    assign busy = (state != S_IDLE);

    always @(posedge clk) begin
        if (rst) begin
            state   <= S_IDLE;
            tx      <= 1'b1;
            clk_cnt <= 0;
            bit_idx <= 3'd0;
            shreg   <= 8'd0;
        end else begin
            case (state)

                S_IDLE: begin
                    tx      <= 1'b1;
                    clk_cnt <= 0;
                    bit_idx <= 3'd0;
                    if (start) begin
                        shreg <= data;
                        state <= S_START;
                    end
                end

                S_START: begin
                    tx <= 1'b0;
                    if (clk_cnt == CPB - 1) begin
                        clk_cnt <= 0;
                        state   <= S_DATA;
                    end else begin
                        clk_cnt <= clk_cnt + 1'b1;
                    end
                end

                S_DATA: begin
                    tx <= shreg[bit_idx];                  // LSB 먼저
                    if (clk_cnt == CPB - 1) begin
                        clk_cnt <= 0;
                        if (bit_idx == 3'd7) state <= S_STOP;
                        else                 bit_idx <= bit_idx + 1'b1;
                    end else begin
                        clk_cnt <= clk_cnt + 1'b1;
                    end
                end

                S_STOP: begin
                    tx <= 1'b1;
                    if (clk_cnt == CPB - 1) begin
                        clk_cnt <= 0;
                        state   <= S_IDLE;
                    end else begin
                        clk_cnt <= clk_cnt + 1'b1;
                    end
                end

                default: state <= S_IDLE;
            endcase
        end
    end
endmodule
```

### 4-5. uart_rx — USART 수신 (예제 3)

**파일:** `uart_rx.v`

```verilog
module uart_rx #(
    parameter CLK_HZ = 100_000_000,
    parameter BAUD   = 115_200
)(
    input  wire       clk,
    input  wire       rst,
    input  wire       rx,
    output reg  [7:0] data,
    output reg        valid               // 1 클럭 펄스: 한 바이트 수신 완료
);
    localparam integer CPB  = CLK_HZ / BAUD;
    localparam integer HALF = CPB / 2;
    localparam integer CW   = $clog2(CPB);

    localparam [1:0] S_IDLE  = 2'd0,
                     S_START = 2'd1,
                     S_DATA  = 2'd2,
                     S_STOP  = 2'd3;

    reg          rx0, rx1;                // 2단 동기화
    reg [1:0]    state;
    reg [CW-1:0] clk_cnt;
    reg [2:0]    bit_idx;
    reg [7:0]    shreg;

    always @(posedge clk) begin
        rx0   <= rx;
        rx1   <= rx0;
        valid <= 1'b0;

        if (rst) begin
            rx0     <= 1'b1;
            rx1     <= 1'b1;
            state   <= S_IDLE;
            clk_cnt <= 0;
            bit_idx <= 3'd0;
            shreg   <= 8'd0;
            data    <= 8'd0;
        end else begin
            case (state)

                S_IDLE: begin
                    clk_cnt <= 0;
                    bit_idx <= 3'd0;
                    if (rx1 == 1'b0) state <= S_START;     // 시작 비트 후보
                end

                S_START: begin                              // 시작 비트 중앙까지 대기
                    if (clk_cnt == HALF - 1) begin
                        clk_cnt <= 0;
                        if (rx1 == 1'b0) state <= S_DATA;   // 진짜 시작 비트
                        else             state <= S_IDLE;   // 글리치
                    end else begin
                        clk_cnt <= clk_cnt + 1'b1;
                    end
                end

                S_DATA: begin                               // 각 비트 중앙에서 샘플링
                    if (clk_cnt == CPB - 1) begin
                        clk_cnt <= 0;
                        shreg   <= {rx1, shreg[7:1]};       // LSB 먼저 → 오른쪽으로 밀기
                        if (bit_idx == 3'd7) state <= S_STOP;
                        else                 bit_idx <= bit_idx + 1'b1;
                    end else begin
                        clk_cnt <= clk_cnt + 1'b1;
                    end
                end

                S_STOP: begin
                    if (clk_cnt == CPB - 1) begin
                        clk_cnt <= 0;
                        state   <= S_IDLE;
                        if (rx1 == 1'b1) begin              // 정지 비트 확인
                            data  <= shreg;
                            valid <= 1'b1;
                        end
                    end else begin
                        clk_cnt <= clk_cnt + 1'b1;
                    end
                end

                default: state <= S_IDLE;
            endcase
        end
    end
endmodule
```

### 4-6. pwm — 타이머 PWM (예제 4)

**파일:** `pwm.v`

```verilog
module pwm #(
    parameter W = 10
)(
    input  wire         clk,
    input  wire         rst,
    input  wire [15:0]  psc,         // 분주값: 카운터 클럭 = clk / (psc+1)
    input  wire [W-1:0] period,      // ARR: 주기 = period+1 카운트
    input  wire [W-1:0] duty,        // CCR: 비교값
    output reg          pwm_out
);
    reg [15:0]  pcnt;
    reg [W-1:0] cnt;

    wire step = (pcnt >= psc);       // 분주 카운터가 한 바퀴 돌면 카운터 한 칸 전진

    always @(posedge clk) begin
        if (rst) begin
            pcnt    <= 16'd0;
            cnt     <= {W{1'b0}};
            pwm_out <= 1'b0;
        end else if (step) begin
            pcnt <= 16'd0;
            if (cnt >= period) cnt <= {W{1'b0}};
            else               cnt <= cnt + 1'b1;
            pwm_out <= (cnt < duty);                  // PWM 모드 1
        end else begin
            pcnt <= pcnt + 1'b1;
        end
    end
endmodule
```

### 4-7. breathing — 듀티 자동 증감 (예제 4의 main 루프)

**파일:** `breathing.v`

```verilog
module breathing #(
    parameter W = 10
)(
    input  wire         clk,
    input  wire         rst,
    input  wire         tick,           // 1 ms 틱
    input  wire [W-1:0] max_duty,
    output reg  [W-1:0] duty
);
    reg down;                           // 0: 증가, 1: 감소

    always @(posedge clk) begin
        if (rst) begin
            duty <= {W{1'b0}};
            down <= 1'b0;
        end else if (tick) begin
            if (!down) begin
                if (duty >= max_duty) down <= 1'b1;
                else                  duty <= duty + 1'b1;
            end else begin
                if (duty == {W{1'b0}}) down <= 1'b0;
                else                   duty <= duty - 1'b1;
            end
        end
    end
endmodule
```

### 4-8. top — 통합

| 출력 | 동작 | 대응하는 STM32 예제 |
|---|---|---|
| `led0` | 0.5초 간격 깜박임 | 예제 1 |
| `led1` | 버튼을 누를 때마다 토글 | 예제 2 / 6 |
| `led2` | 숨쉬기 PWM (1 kHz, 2초 왕복) | 예제 4 |
| UART | 받은 문자를 그대로 돌려보냄 (에코) | 예제 3 |

Basys 3 버튼은 누르면 High이므로 `ACTIVE_LOW(0)`으로 설정했습니다.

**파일:** `top.v`

```verilog
module top #(
    parameter CLK_HZ = 100_000_000
)(
    input  wire clk,
    input  wire btn,
    input  wire uart_rx_pin,
    output wire uart_tx_pin,
    output wire led0,
    output wire led1,
    output wire led2
);
    //--------------------------------------------------------------
    // 파워온 리셋: 전원 인가 후 15 클럭 동안 rst = 1
    //--------------------------------------------------------------
    reg [3:0] por = 4'hF;
    always @(posedge clk) begin
        if (por != 4'd0) por <= por - 1'b1;
    end
    wire rst = (por != 4'd0);

    //--------------------------------------------------------------
    // 예제 1: LED 깜박임
    //--------------------------------------------------------------
    blink #(.CLK_HZ(CLK_HZ)) u_blink (
        .clk(clk), .rst(rst), .led(led0)
    );

    //--------------------------------------------------------------
    // SysTick 대응: 1 ms 틱
    //--------------------------------------------------------------
    wire tick;
    tick_gen #(.CLK_HZ(CLK_HZ), .TICK_HZ(1000)) u_tick (
        .clk(clk), .rst(rst), .tick(tick), .ms_count()
    );

    //--------------------------------------------------------------
    // 예제 2/6: 버튼 → LED1 토글
    //--------------------------------------------------------------
    wire btn_state, press, release_p;
    debounce #(.CLK_HZ(CLK_HZ), .STABLE_MS(20), .ACTIVE_LOW(0)) u_db (
        .clk(clk), .rst(rst), .btn_in(btn),
        .btn_state(btn_state), .press_pulse(press), .release_pulse(release_p)
    );

    reg led1_r = 1'b0;
    always @(posedge clk) begin
        if (rst)        led1_r <= 1'b0;
        else if (press) led1_r <= ~led1_r;
    end
    assign led1 = led1_r;

    //--------------------------------------------------------------
    // 예제 4: 숨쉬기 PWM (1 MHz 카운터, 주기 1000 → 1 kHz)
    //--------------------------------------------------------------
    wire [9:0] duty;
    wire [15:0] psc = (CLK_HZ / 1_000_000) - 1;

    breathing #(.W(10)) u_breath (
        .clk(clk), .rst(rst), .tick(tick),
        .max_duty(10'd999), .duty(duty)
    );

    pwm #(.W(10)) u_pwm (
        .clk(clk), .rst(rst), .psc(psc),
        .period(10'd999), .duty(duty), .pwm_out(led2)
    );

    //--------------------------------------------------------------
    // 예제 3: UART 에코
    //--------------------------------------------------------------
    wire [7:0] rx_data;
    wire       rx_valid;
    wire       tx_busy;
    reg  [7:0] tx_data  = 8'd0;
    reg        tx_start = 1'b0;

    uart_rx #(.CLK_HZ(CLK_HZ), .BAUD(115_200)) u_rx (
        .clk(clk), .rst(rst), .rx(uart_rx_pin),
        .data(rx_data), .valid(rx_valid)
    );

    uart_tx #(.CLK_HZ(CLK_HZ), .BAUD(115_200)) u_tx (
        .clk(clk), .rst(rst), .start(tx_start),
        .data(tx_data), .tx(uart_tx_pin), .busy(tx_busy)
    );

    always @(posedge clk) begin
        tx_start <= 1'b0;
        if (rx_valid && !tx_busy) begin
            tx_data  <= rx_data;
            tx_start <= 1'b1;
        end
    end
endmodule
```

---

## 5. 테스트벤치

시뮬레이션이 길어지지 않도록 파라미터를 작게 줄여 검증합니다. 모든 테스트벤치는 자체 점검 후 `$finish`로 종료합니다.

### 5-1. tb_blink

**파일:** `tb_blink.v`

```verilog
`timescale 1ns/1ps

module tb_blink;
    reg  clk = 1'b0;
    reg  rst = 1'b1;
    wire led;

    always #5 clk = ~clk;                       // 100 MHz

    // CLK_HZ = 1000 → 반주기 500 클럭 = 5 us (시뮬레이션용 축소)
    blink #(.CLK_HZ(1000)) dut (.clk(clk), .rst(rst), .led(led));

    integer errors = 0;

    initial begin
        #50  rst = 1'b0;

        #5200;
        if (led !== 1'b1) begin
            errors = errors + 1;
            $display("tb_blink: 첫 토글 후 led 는 1 이어야 함");
        end

        #5000;
        if (led !== 1'b0) begin
            errors = errors + 1;
            $display("tb_blink: 두 번째 토글 후 led 는 0 이어야 함");
        end

        if (errors == 0) $display("tb_blink: PASS");
        else             $display("tb_blink: FAIL (%0d)", errors);
        $finish;
    end
endmodule
```

### 5-2. tb_debounce

**파일:** `tb_debounce.v`

```verilog
`timescale 1ns/1ps

module tb_debounce;
    reg  clk = 1'b0;
    reg  rst = 1'b1;
    reg  btn = 1'b1;                            // 액티브 로우: 평소 High
    wire state, pp, rp;

    always #5 clk = ~clk;

    // CLK_HZ=1000, STABLE_MS=20 → 안정 시간 = 20 클럭 = 200 ns
    debounce #(.CLK_HZ(1000), .STABLE_MS(20), .ACTIVE_LOW(1)) dut (
        .clk(clk), .rst(rst), .btn_in(btn),
        .btn_state(state), .press_pulse(pp), .release_pulse(rp)
    );

    integer presses  = 0;
    integer releases = 0;
    integer i;
    integer errors   = 0;

    always @(posedge clk) begin
        if (pp) presses  <= presses  + 1;
        if (rp) releases <= releases + 1;
    end

    initial begin
        #40 rst = 1'b0;

        // 눌림 쪽 바운싱: 30 ns 짜리 떨림 3번
        for (i = 0; i < 3; i = i + 1) begin
            #30 btn = 1'b0;
            #30 btn = 1'b1;
        end

        #50  btn = 1'b0;                        // 실제 눌림
        #400;                                   // 안정 시간(200 ns)보다 길게 유지

        // 뗌 쪽 바운싱
        for (i = 0; i < 3; i = i + 1) begin
            #30 btn = 1'b1;
            #30 btn = 1'b0;
        end

        #50  btn = 1'b1;                        // 실제 뗌
        #400;

        if (presses  !== 1)    begin errors = errors + 1; $display("tb_debounce: 눌림 펄스는 1번이어야 함 (%0d)", presses);  end
        if (releases !== 1)    begin errors = errors + 1; $display("tb_debounce: 뗌 펄스는 1번이어야 함 (%0d)", releases); end
        if (state    !== 1'b0) begin errors = errors + 1; $display("tb_debounce: 최종 상태는 안 눌림이어야 함");            end

        if (errors == 0) $display("tb_debounce: PASS");
        else             $display("tb_debounce: FAIL");
        $finish;
    end
endmodule
```

### 5-3. tb_uart_loop — TX → RX 루프백

100 MHz 클럭, `BAUD = 10_000_000` 으로 한 비트를 10 클럭으로 줄였습니다.

**파일:** `tb_uart_loop.v`

```verilog
`timescale 1ns/1ps

module tb_uart_loop;
    localparam CLK_HZ = 100_000_000;
    localparam BAUD   = 10_000_000;             // 한 비트 = 10 클럭

    reg        clk   = 1'b0;
    reg        rst   = 1'b1;
    reg        start = 1'b0;
    reg  [7:0] din   = 8'd0;
    wire [7:0] dout;
    wire       line, busy, valid;

    always #5 clk = ~clk;

    uart_tx #(.CLK_HZ(CLK_HZ), .BAUD(BAUD)) u_tx (
        .clk(clk), .rst(rst), .start(start),
        .data(din), .tx(line), .busy(busy)
    );

    uart_rx #(.CLK_HZ(CLK_HZ), .BAUD(BAUD)) u_rx (
        .clk(clk), .rst(rst), .rx(line),
        .data(dout), .valid(valid)
    );

    integer errors = 0;

    task send_and_check(input [7:0] b);
        begin
            @(posedge clk);
            din   <= b;
            start <= 1'b1;
            @(posedge clk);
            start <= 1'b0;

            @(posedge valid);                   // 수신 완료 펄스 대기
            #1;
            if (dout !== b) begin
                errors = errors + 1;
                $display("tb_uart_loop: 불일치 보냄=%h 받음=%h", b, dout);
            end

            wait (!busy);                       // 송신이 완전히 끝날 때까지
        end
    endtask

    initial begin
        #100 rst = 1'b0;
        #50;

        send_and_check(8'h55);
        send_and_check(8'hA3);
        send_and_check(8'h00);
        send_and_check(8'hFF);
        send_and_check(8'h3C);

        if (errors == 0) $display("tb_uart_loop: PASS");
        else             $display("tb_uart_loop: FAIL (%0d)", errors);
        $finish;
    end
endmodule
```

---

## 6. 시뮬레이션 방법

### 6-1. Icarus Verilog + GTKWave (오픈소스, 클라우드/로컬 모두 가능)

```bash
# UART 루프백
iverilog -g2012 -o sim_uart tb_uart_loop.v uart_tx.v uart_rx.v
vvp sim_uart

# 디바운스
iverilog -g2012 -o sim_db tb_debounce.v debounce.v
vvp sim_db

# 블링크
iverilog -g2012 -o sim_blink tb_blink.v blink.v
vvp sim_blink
```

파형을 보려면 테스트벤치의 `initial` 블록 앞부분에 다음 두 줄을 추가하고 `gtkwave dump.vcd`로 확인합니다.

```verilog
initial begin
    $dumpfile("dump.vcd");
    $dumpvars(0, tb_uart_loop);
end
```

`PASS`가 출력되면 통과입니다.

### 6-2. Vivado

1. 프로젝트 생성 → Add Sources에서 `*.v` 추가 (`tb_*.v`는 **Simulation Sources**로 추가)
2. Run Simulation → Run Behavioral Simulation
3. 테스트벤치가 `$finish`로 끝납니다.

---

## 7. 합성 및 보드 적용

제약 파일(`basys3.xdc`)은 **03_vhdl_implementation.md의 7장**과 동일합니다. 핀 이름은 예시이므로 사용하는 보드 매뉴얼로 반드시 확인하세요.

### 확인 순서
1. 비트스트림 생성 후 보드에 다운로드
2. `led0`이 0.5초 간격으로 깜박이는지 확인
3. 버튼을 누를 때마다 `led1`이 토글되는지 확인
4. `led2`가 2초 주기로 밝아졌다 어두워지는지 확인
5. 시리얼 터미널(115200 8N1)에서 글자를 입력하면 그대로 돌아오는지 확인

---

## 8. VHDL ↔ Verilog 문법 대응표

| 항목 | VHDL | Verilog |
|---|---|---|
| 모듈 선언 | `entity` + `architecture` | `module ... endmodule` |
| 파라미터 | `generic` | `parameter` / `localparam` |
| 클럭 에지 | `rising_edge(clk)` | `@(posedge clk)` |
| 순차 대입 | `<=` (신호) | `<=` (비차단) |
| 상태 정의 | `type state_t is (...)` | `localparam` 상수 |
| 상태 분기 | `case state is when ...` | `case (state) ... endcase` |
| 비트 연결 | `rx1 & shreg(7 downto 1)` | `{rx1, shreg[7:1]}` |
| 모두 0 | `(others => '0')` | `{W{1'b0}}` |
| 부호 없는 값 | `unsigned` (numeric_std) | `reg [N-1:0]` (기본 부호 없음) |
| 로그 계산 | 직접 함수 작성 필요 | `$clog2` |
| 인스턴스 | `entity work.xxx generic map port map` | `xxx #(...) inst (...)` |
| 사용하지 않는 출력 | `open` | `.port()` 비워 두기 |

---

## 9. 설계 포인트 정리

| 주제 | 설명 |
|---|---|
| 펄스 신호 설계 | 블록 맨 앞에서 기본값 `0`을 대입하고, 조건이 맞을 때만 `1`로 덮어쓴다 (마지막 비차단 대입이 유효) |
| 2단 동기화 | 비동기 입력은 `s0 → s1`을 거친 `s1`만 사용 |
| `reg` 초기값 | `reg [3:0] por = 4'hF;` 같은 초기값은 FPGA 합성에서는 지원되지만 ASIC 설계에서는 리셋으로 대체해야 함 |
| `$clog2` 주의 | `HALF`나 `DIV`가 1이면 폭이 0이 되므로 CLK_HZ와 목표 주파수가 충분히 차이 나야 함 |
| 래치 방지 | `case`에는 `default`를 두고, 모든 분기에서 레지스터가 의도대로 갱신되는지 확인 |
| 보레이트 오차 | `CPB = CLK_HZ / BAUD` 정수 나눗셈 오차. 100 MHz / 115200 = 868.05 → 868 (약 0.006%) |

---

## 10. 실습 과제

1. `uart_tx`에 짝수 패리티 비트를 추가하고 `uart_rx`가 패리티 오류를 검출하도록 확장하기
2. `top`에서 문자 `"1"`을 받으면 `led1` ON, `"0"`이면 OFF가 되도록 수정하기
3. `uart_tx` 앞에 **FIFO**(예: 16바이트)를 붙여 `busy` 대기 없이 연속 송신하기 (STM32의 `TXE`+버퍼 구조와 비교)
4. `pwm`에 PWM 모드 2 선택 입력 추가하기
5. 같은 `top`을 VHDL 버전과 함께 합성해 **자원 사용량(LUT/FF)과 타이밍**이 어떻게 다른지 비교하기

---

## 11. 한계와 주의

- **검증 범위**: Icarus Verilog(`-g2012`)에서 `tb_blink`, `tb_debounce`, `tb_uart_loop`가 모두 PASS이고, 부록의 `top` 통합 테스트(UART 에코 2바이트, 버튼 토글, blink, PWM)도 PASS입니다. `uart_rx`의 비트 순서를 일부러 틀리게 바꾼 변형에서는 `tb_uart_loop`가 FAIL을 내는 것까지 확인했습니다.
- **검증하지 않은 범위**: Vivado 합성·구현, 타이밍 클로저, 실제 보드 동작. 핀 배치와 클럭 제약은 사용하는 도구·보드에서 직접 확인해야 합니다.
- 보드의 클럭 주파수와 버튼 극성이 다르면 `CLK_HZ`, `ACTIVE_LOW`를 바꾸세요.
- PWM의 `psc` 계산(`CLK_HZ / 1_000_000 - 1`)은 클럭이 1 MHz의 배수여야 정확합니다.

이전 문서: **02_stm32f411_examples.md** / VHDL 버전: **03_vhdl_implementation.md**

---

## 부록. top 통합 테스트벤치

`CLK_HZ`를 11,520,000으로 낮춰 한 비트를 100 클럭, 1 ms를 11,520 클럭으로 만든 뒤 UART 에코, 버튼 토글, blink, PWM을 한 번에 확인합니다. 시뮬레이션 시간이 설계상 약 2.3초 분량이라 실행에 수십 초가 걸릴 수 있습니다.

**파일:** `tb_top_check.v`

```verilog
`timescale 1ns/1ps
module tb_top_check;
  localparam CLK_HZ = 11_520_000;   // 115200 baud -> 100 clk/bit, 1ms = 11520 clk
  reg clk=0, btn=0, rxp=1; wire txp, l0,l1,l2;
  always #5 clk=~clk;
  top #(.CLK_HZ(CLK_HZ)) dut(.clk(clk),.btn(btn),.uart_rx_pin(rxp),.uart_tx_pin(txp),.led0(l0),.led1(l1),.led2(l2));
  integer i, errors=0, l0_toggles=0, l2_hi=0, l2_lo=0; reg [7:0] got;
  reg l0_prev=0;
  always @(posedge clk) begin
    if ((l0===1'b0 || l0===1'b1) && (l0_prev===1'b0 || l0_prev===1'b1) && l0!==l0_prev) l0_toggles<=l0_toggles+1;
    l0_prev<=l0;
    if (l2===1'b1) l2_hi<=l2_hi+1; else l2_lo<=l2_lo+1;
  end
  task send(input [7:0] b); integer k; begin
    rxp=0; repeat(100) @(posedge clk);
    for(k=0;k<8;k=k+1) begin rxp=b[k]; repeat(100) @(posedge clk); end
    rxp=1; repeat(100) @(posedge clk); end endtask
  task recv(output [7:0] r); integer k; begin
    @(negedge txp); repeat(150) @(posedge clk);           // middle of bit 0
    for(k=0;k<8;k=k+1) begin r[k]=txp; repeat(100) @(posedge clk); end end endtask
  initial begin
    repeat(50) @(posedge clk);
    fork send(8'h41); recv(got); join
    if (got!==8'h41) begin errors=errors+1; $display("echo FAIL got=%h",got); end else $display("echo OK 0x%h",got);
    fork send(8'h7E); recv(got); join
    if (got!==8'h7E) begin errors=errors+1; $display("echo FAIL got=%h",got); end else $display("echo OK 0x%h",got);
    // button: press 40ms, release 40ms -> led1 toggles once on press
    btn=1; repeat(11520*40) @(posedge clk); if (l1!==1'b1) begin errors=errors+1; $display("led1 not toggled on press"); end else $display("led1 toggled OK");
    btn=0; repeat(11520*40) @(posedge clk); if (l1!==1'b1) begin errors=errors+1; $display("led1 changed on release"); end else $display("led1 stays on release OK");
    // run ~2.2 s for blink + breathing
    repeat(11520*2200) @(posedge clk);
    $display("led0 toggles=%0d (expect ~4)", l0_toggles);
    $display("led2 hi=%0d lo=%0d  (duty ratio %0d%%)", l2_hi, l2_lo, 100*l2_hi/(l2_hi+l2_lo));
    if (l0_toggles<3 || l0_toggles>5) errors=errors+1;
    if (l2_hi==0 || l2_lo==0) errors=errors+1;
    if (errors==0) $display("TOP CHECK: PASS"); else $display("TOP CHECK: FAIL");
    $finish;
  end
endmodule
```

```bash
iverilog -g2012 -o sim_top tb_top_check.v top.v blink.v tick_gen.v debounce.v uart_tx.v uart_rx.v pwm.v breathing.v
vvp sim_top
```

`TOP CHECK: PASS`가 나오면 통과입니다.
