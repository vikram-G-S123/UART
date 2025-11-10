# UART

## Aim:

Synthesize UART design using Constraints and analyse area and Power reports.

## Tool Required:

Functional Simulation: Incisive Simulator (ncvlog, ncelab, ncsim)

Synthesis: Genus

### Step 1: Getting Started

Synthesis requires three files as follows,

◦ Liberty Files (.lib)


◦ Verilog/VHDL Files (.v or .vhdl or .vhd)

**Design Code**
`timescale 1ns / 1ps

module uart64_with_baud #(
    parameter CLK_FREQ = 50_000_000,   
    parameter BAUD_RATE = 9600         
)(
    input wire clk,                    // System clock
    input wire reset,                  // Asynchronous reset
    // Transmitter
    input wire ld_tx_data,             // Load transmit data
    input wire [63:0] tx_data,         // 64-bit transmit data
    input wire tx_enable,              // Enable transmission
    output reg tx_out,                 // Serial output line
    output reg tx_empty,               // TX ready flag
    // Receiver
    input wire uld_rx_data,            // Unload received data
    output reg [63:0] rx_data,         // 64-bit received data
    input wire rx_enable,              // Enable receiver
    input wire rx_in,                  // Serial input
    output reg rx_empty                // RX ready flag
);

    localparam integer BAUD_DIV = CLK_FREQ / BAUD_RATE;
    reg [15:0] baud_cnt = 0;
    reg baud_tick = 0;

    always @(posedge clk or posedge reset) begin
        if (reset) begin
            baud_cnt <= 0;
            baud_tick <= 0;
        end else begin
            if (baud_cnt == BAUD_DIV/2) begin
                baud_cnt <= 0;
                baud_tick <= 1;
            end else begin
                baud_cnt <= baud_cnt + 1;
                baud_tick <= 0;
            end
        end
    end

    // ----------- Transmitter -----------
    reg [63:0] tx_reg;
    reg [6:0] tx_cnt;  // Need 0-65 count (1 start + 64 data + 1 stop)
    reg tx_busy;

    always @(posedge clk or posedge reset) begin
        if (reset) begin
            tx_reg <= 0;
            tx_cnt <= 0;
            tx_empty <= 1;
            tx_out <= 1;
            tx_busy <= 0;
        end else if (baud_tick) begin
            // Load new data when TX is empty
            if (ld_tx_data && tx_empty) begin
                tx_reg <= tx_data;
                tx_empty <= 0;
                tx_busy <= 1;
                tx_cnt <= 0;
            end

            // Transmission logic
            if (tx_busy && tx_enable) begin
                tx_cnt <= tx_cnt + 1;

                if (tx_cnt == 0)
                    tx_out <= 0;  // Start bit
                else if (tx_cnt >= 1 && tx_cnt <= 64)
                    tx_out <= tx_reg[tx_cnt - 1];  // Data bits (LSB first)
                else if (tx_cnt == 65) begin
                    tx_out <= 1;  // Stop bit
                    tx_empty <= 1;
                    tx_busy <= 0;
                end
            end
        end
    end

    // ----------- Receiver -----------
    reg [63:0] rx_reg;
    reg [6:0] rx_cnt;           // up to 64 data bits
    reg [3:0] rx_sample_cnt;
    reg rx_busy;
    reg rx_d1, rx_d2;

    always @(posedge clk or posedge reset) begin
        if (reset) begin
            rx_data <= 0;
            rx_empty <= 1;
            rx_reg <= 0;
            rx_cnt <= 0;
            rx_sample_cnt <= 0;
            rx_busy <= 0;
            rx_d1 <= 1;
            rx_d2 <= 1;
        end else begin
            // Synchronize input
            rx_d1 <= rx_in;
            rx_d2 <= rx_d1;

            if (baud_tick && rx_enable) begin
                // Detect start bit
                if (!rx_busy && !rx_d2) begin
                    rx_busy <= 1;
                    rx_cnt <= 0;
                    rx_sample_cnt <= 0;
                end
                // Data sampling
                else if (rx_busy) begin
                    rx_sample_cnt <= rx_sample_cnt + 1;
                    if (rx_sample_cnt == 8) begin
                        rx_sample_cnt <= 0;
                        rx_cnt <= rx_cnt + 1;

                        if (rx_cnt >= 1 && rx_cnt <= 64)
                            rx_reg[rx_cnt - 1] <= rx_d2;

                        if (rx_cnt == 65) begin
                            rx_data <= rx_reg;
                            rx_empty <= 0;
                            rx_busy <= 0;
                        end
                    end
                end
            end

            // When data is read, mark RX empty
            if (uld_rx_data)
                rx_empty <= 1;
        end
    end

endmodule



**TestBench:**

`timescale 1ns / 1ps
module uart64_with_baud_tb;

    reg clk;
    reg reset;
    reg ld_tx_data;
    reg [63:0] tx_data;
    reg tx_enable;
    wire tx_out;
    wire tx_empty;
    reg uld_rx_data;
    wire [63:0] rx_data;
    reg rx_enable;
    wire rx_empty;
    reg rx_in;

    // Instantiate DUT
    uart64_with_baud #(
        .CLK_FREQ(50_000_000),
        .BAUD_RATE(9600)
    ) uut (
        .clk(clk),
        .reset(reset),
        .ld_tx_data(ld_tx_data),
        .tx_data(tx_data),
        .tx_enable(tx_enable),
        .tx_out(tx_out),
        .tx_empty(tx_empty),
        .uld_rx_data(uld_rx_data),
        .rx_data(rx_data),
        .rx_enable(rx_enable),
        .rx_in(rx_in),
        .rx_empty(rx_empty)
    );

    // Generate system clock (50 MHz)
    initial clk = 0;
    always #10 clk = ~clk;

    // Connect TX → RX
    always @(tx_out)
        rx_in = tx_out;

    initial begin
        $dumpfile("uart64.vcd");
        $dumpvars(0, uart64_with_baud_tb);

        reset = 1;
        ld_tx_data = 0;
        tx_enable = 1;
        rx_enable = 1;
        uld_rx_data = 0;
        rx_in = 1;

        #100 reset = 0;
        #200;

        tx_data = 64'hA5A5_1234_DEAD_BEEF;  // Example 64-bit data
        ld_tx_data = 1;
        #20 ld_tx_data = 0;

        wait (rx_empty == 0);
        $display("Received 64-bit Data: %h", rx_data);

        uld_rx_data = 1;
        #20 uld_rx_data = 0;

        #10000 $finish;
    end

endmodule



### Step 2 : Performing Synthesis

The Liberty files are present in the library path,

![Screenshot 2025-05-27 112728](https://github.com/user-attachments/assets/84434a7b-0187-42c6-bb06-9283dc524879)

![Screenshot 2025-05-27 112851](https://github.com/user-attachments/assets/5b370bdf-7ea0-4b6b-aa5e-0f2be30f9794)


• The Available technology nodes are 180nm ,90nm and 45nm.

• In the terminal, initialise the tools with the following commands if a new terminal is being
used.

◦ csh
◦ source /cadence/install/cshrc

![Screenshot 2025-05-25 212758](https://github.com/user-attachments/assets/a419fccc-1012-4056-8ac5-f6e8ece6d7b8)


• The tool used for Synthesis is “Genus”. Hence, type “genus -gui” to open the tool.

![Screenshot 2025-05-25 205017](https://github.com/user-attachments/assets/566d282c-3ef2-4360-a0a1-3fc07e88b224)


• Genus Script file with .tcl file Extension commands are executed one by one to synthesize the netlist.

![Screenshot 2025-05-27 113001](https://github.com/user-attachments/assets/06471485-a69c-4924-9a37-57d2f2535151)


![Screenshot 2025-05-27 114508](https://github.com/user-attachments/assets/9300b983-886a-415a-8d25-d82cd50ac121)


#### Synthesis RTL Schematic :

![Screenshot 2025-05-27 104506](https://github.com/user-attachments/assets/69c2348d-0ca1-449a-a349-ed6a04dae576)

#### Area report:

![Screenshot 2025-05-27 113021](https://github.com/user-attachments/assets/5170e6e5-bd02-4714-91a9-4cedfbc6bf4f)

#### Power Report:

![Screenshot 2025-05-27 112926](https://github.com/user-attachments/assets/195a1420-d1b2-4ba6-be15-fe68e684eec4)

#### Result: 

![Screenshot 2025-05-27 110723](https://github.com/user-attachments/assets/7d4bfa72-9e45-4469-a5d5-6515d809099a)

The generic netlist of 32 bit ALU  has been created, and area, power reports have been tabulated and generated using Genus.
