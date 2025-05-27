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

module uart (
    input reset,
    input txclk,
    input ld_tx_data,
    input [7:0] tx_data,
    input tx_enable,
    output reg tx_out,
    output reg tx_empty,
    input rxclk,
    input uld_rx_data,
    output reg [7:0] rx_data,
    input rx_enable,
    input rx_in,
    output reg rx_empty
);

// Internal Variables
reg [7:0] tx_reg;
reg tx_over_run;
reg [3:0] tx_cnt;

reg [7:0] rx_reg;
reg [3:0] rx_sample_cnt;
reg [3:0] rx_cnt;
reg rx_frame_err;
reg rx_over_run;
reg rx_d1;
reg rx_d2;
reg rx_busy;

// UART RX Logic
always @(posedge rxclk or posedge reset)
begin

    if (reset) begin
        rx_reg <= 0;
        rx_data <= 0;
        rx_sample_cnt <= 0;
        rx_cnt <= 0;
        rx_frame_err <= 0;
        rx_over_run <= 0;
        rx_empty <= 1;
        rx_d1 <= 1;
        rx_d2 <= 1;
        rx_busy <= 0;
    end 
    else begin
    
        rx_d1 <= rx_in;
        rx_d2 <= rx_d1;

        if (uld_rx_data) begin
            rx_data <= rx_reg;
            rx_empty <= 1;
        end

        if (rx_enable) begin
            if (!rx_busy && !rx_d2) begin
                rx_busy <= 1;
                rx_sample_cnt <= 1;
                rx_cnt <= 0;
            end

            if (rx_busy) begin
                rx_sample_cnt <= rx_sample_cnt + 1;

                if (rx_sample_cnt == 7) begin
                    if ((rx_d2 == 1) && (rx_cnt == 0)) begin 
                        rx_busy <= 0;
                    end 
                    else begin
                        rx_cnt <= rx_cnt + 1;

                        if (rx_cnt > 0 && rx_cnt < 9) begin 
                            rx_reg[rx_cnt - 1] <= rx_d2;
                        end

                        if (rx_cnt == 9) begin
                            rx_busy <= 0;
                            rx_empty <= 0;
                            rx_over_run <= (rx_empty) ? 0 : 1;
                        end
                    end
                end
            end
        end

        if (!rx_enable) begin
            rx_busy <= 0;
        end
    end
end

// UART TX Logic
always @(posedge txclk or posedge reset)
begin

    if (reset) begin
        tx_reg <= 0;
        tx_empty <= 1;
        tx_over_run <= 0;
        tx_out <= 1;
        tx_cnt <= 0;
    end 
    else begin
        if (ld_tx_data) begin
            if (!tx_empty) begin
                tx_over_run <= 1;
            end 
            else begin
                tx_reg <= tx_data;
                tx_empty <= 0;
            end
        end

        if (tx_enable && !tx_empty) begin
            tx_cnt <= tx_cnt + 1;
            if (tx_cnt == 0)
                tx_out <= 0; // Start bit
            else if (tx_cnt > 0 && tx_cnt < 9)
                tx_out <= tx_reg[tx_cnt - 1]; // Data bits
            else if (tx_cnt == 9) begin
                tx_out <= 1; // Stop bit
                tx_cnt <= 0;
                tx_empty <= 1;
            end
        end

        if (!tx_enable) begin
            tx_cnt <= 0;
        end
    end
end

endmodule



**TestBench:**

`timescale 1 ns / 1 ps
 module uart_tb; 
// Inputs
reg reset; 
reg txclk;
reg ld_tx_data; 
reg [7:0] tx_data; 
reg tx_enable; 
reg rxclk;
reg uld_rx_data; 
reg rx_enable; 
reg rx_in;
// Outputs
wire tx_out;
wire tx_empty;
wire [7:0] rx_data;
wire rx_empty;

uart uut (
.reset(reset),
.txclk(txclk),
.ld_tx_data(ld_tx_data),
.tx_data(tx_data),
.tx_enable(tx_enable),
.tx_out (tx_out),
.tx_empty(tx_empty), 
.rxclk(rxclk),
.uld_rx_data(uld_rx_data),
.rx_data(rx_data),
.rx_enable(rx_enable),
.rx_in(rx_in),
.rx_empty(rx_empty) );

reg clk;

initial clk=0;
always #10 clk = ~clk; 
reg [3:0] counter;
initial begin
rxclk=0;
txclk=0;
counter=0;
end
always @(posedge clk) 
begin

counter<=counter+1;
if (counter == 15) 

txclk <= ~txclk;
rxclk<= ~rxclk;
end

always@ (tx_out)
 rx_in=tx_out;
initial begin
reset = 1;
ld_tx_data = 0;
tx_data = 0;
tx_enable = 1;
uld_rx_data = 0;
rx_enable = 1;
rx_in = 1;
#500;
reset = 0;
tx_data=8'b0111_1111;
#500;
wait (tx_empty==1); //make sure data can be sent
ld_tx_data = 1; //load data to send
wait (tx_empty==0); //wait until data loaded for send
$display("Data loaded for send");
ld_tx_data = 0;
wait (tx_empty==1); //wait for flag of data to finish sending 
$display ("Data sent");
wait (rx_empty==0); //wait for
$display("RX Byte Ready");
uld_rx_data = 1;
wait (rx_empty==1);
$display("RX Byte Unloaded: %b", rx_data);
#100;
$finish;
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
