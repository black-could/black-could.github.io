---
layout: default
title: UART TX
toc: true
---

# 此專案目的在於自行走過設計規格至STA流程 .
# 因為是第一個小專案，所以花了不少時間在建立環境上 .
# 在此紀錄RTL code , testbench , monitor , simulation waveform , yosys synthesis report , openSTA timming report .
# script不紀錄 .

{:toc}
###
IC design flow      : spec. --> RTL --> simulation --> synthesis --> STA

compiler       tool : lcarus verilog (iverilog)
simulation     tool : vvp
synthesis      tool : Yosys
STA            tool : OpenSTA
waveform check tool : gtkwave
library             : sky130_fd_sc_hd__tt_025C_1v80.lib (sky130 , open source pdk)

###
\[Specification\]
Data Width         : 8 bit 
Start Bit          : 1 bit (low)
Stop Bit           : 1 bit (high)
Parity             : NA
Transmission Order : LSB first
Baud Rate          : 115200 or 9600 
Main clk           : 50M Hz

\[Interface\]
| Singnal Name | Direction | Width | Note |
| :----------: | :-------: | :---: | :--- |
| clk | input | 1 bit | NA |
| reset_n | input | 1 bit | NA |
| tx_start | input | 1 bit | NA |
| tx_data_input | input | 8 bit | NA |
| baud_div | input | 16 bit | enter 5208 baud rate is 9600 , enter 434 baud rate is 115200 |
| tx_ready | output | 1 bit | NA |
| tx_busy | output | 1 bit | NA |
| tx_data_output | output | 1 bit | NA |

\[Action\]
Set Baud Rate --> Set tx_data_input --> Set tx_start high --> Wait tx_busy low --
                         ^                                                       |
                         |                                                       |
                          -------------------------------------------------------

If want to change , please check tx_busy was low and tx_ready was high .

\[Architecture_Blocks\]
![uart_block](https://github.com/black-could/black-could.github.io/blob/main/images/uart_blocks.png)


\[Pattern_List\]
| Define Name | Pattern Name | Baud Rate | Test Data |
| :---------: | :----------: | :-------: | :-------: |
| PATTERN01   | pattern01.v  | 9600 | 8'hFF -> 8'h00 -> 8'hFF |
| PATTERN02   | pattern02.v  | 9600 | 8'h55 -> 8'hAA -> 8'h55 |
| PATTERN03   | pattern03.v  | 115200 | 8'hFF -> 8'h00 -> 8'hFF |
| PATTERN04   | pattern04.v  | 115200 | 8'h55 -> 8'hAA -> 8'h55 |
| NA   | default_pattern.v  | 9600 | 8'h58 |

\[Environment_Architecture\]
![uart_tree](https://github.com/black-could/black-could.github.io/blob/main/images/uart_tree.png)

###
\[RTL code\]
Testbench :
```verilog
`timescale 1ns/10ps
`define TOP      uart_tx_tb.uart_top
`define TB       uart_tx_tb

module uart_tx_tb;

// Monitor
`include "../monitor/uart_monitor.v"

//Clock Parameter
parameter CLK_50M_HALF_PIR = 10 ;

//Dump
initial begin
 $dumpfile("dump.vcd");
 $dumpvars(0, uart_tx_tb);
end

//Output
 wire         tx_data_output;
 wire         tx_ready;
 wire         tx_busy;

//Input
 reg  [15: 0]  baud_div;
 reg           tx_start;
 reg  [ 7: 0]  tx_data_input;

//Connect
 wire          clk;

//Clock
 reg           clk_50M;

initial clk_50M = 1'b0;
always  #CLK_50M_HALF_PIR clk_50M = ~clk_50M;

//Reset
 reg           reset_n;

initial begin
  reset_n = 1'b1;
  #10;
  reset_n = 1'b0;
  #CLK_50M_HALF_PIR;
  #CLK_50M_HALF_PIR;
  reset_n = 1'b1;
end

// Circuit
assign clk = clk_50M;

uart_top uart_top (
 .clk            (clk),
 .reset_n        (reset_n),
 .baud_div       (baud_div),
 .tx_start       (tx_start),
 .tx_data_input  (tx_data_input),
 .tx_data_output (tx_data_output),
 .tx_ready       (tx_ready),
 .tx_busy        (tx_busy)
);

// Initial Value
initial begin
 baud_div      = 16'd1;
 tx_start      = 1'd0;
 tx_data_input = 1'd0;
end

// Pattern List
`ifdef PATTERN01
  `include "../pattern/pattern01.v"
`elsif PATTERN02
  `include "../pattern/pattern02.v"
`elsif PATTERN03
  `include "../pattern/pattern03.v"
`elsif PATTERN04
  `include "../pattern/pattern04.v"
`else
  `include "../pattern/default_pattern.v"
`endif

// Task
task send_data (input [7:0] data);
begin
  @(posedge clk);
  wait(tx_ready);
  @(posedge clk);
  tx_data_input <= data;
  tx_start      <= 1'b1;
  @(posedge clk);
  tx_start      <= 1'b0;
end
endtask


endmodule
```

Uart TOP :
```verilog
module uart_top (
 input          clk,
 input          reset_n,
 input  [15: 0] baud_div,
 input          tx_start,
 input  [ 7: 0] tx_data_input,
 output         tx_data_output,
 output         tx_ready,
 output         tx_busy
);

wire  baud_tick;

baud_rate_gen baud_rate_gen (
 .clk         (clk),
 .reset_n     (reset_n),
 .tx_busy_i   (tx_busy),
 .baud_div_i  (baud_div),
 .baud_tick_o (baud_tick)

);

uart_tx uart_tx (
 .clk          (clk),
 .reset_n      (reset_n),
 .baud_tick_i  (baud_tick),
 .tx_start_i   (tx_start),
 .tx_data_i    (tx_data_input),
 .tx_data_o    (tx_data_output),
 .tx_ready_o   (tx_ready),
 .tx_busy_o    (tx_busy)
);

endmodule
```

Uart TX :
```verilog
module uart_tx (
 input         clk,
 input         reset_n,
 input         baud_tick_i,
 input         tx_start_i,
 input [ 7: 0] tx_data_i,
 output        tx_data_o,
 output        tx_ready_o,
 output        tx_busy_o
);

  parameter IDLE = 2'b00,
            ST   = 2'b01,
            DATA = 2'b10,
            DONE = 2'b11;

 wire [ 7: 0] tx_d_shf_reg_next;
 wire [ 2: 0] tx_d_count_next;
 wire         tx_d_shf_en, tx_d_fin;
 
 reg  [ 1: 0] state_next;
 reg  [ 1: 0] state;
 reg  [ 7: 0] tx_d_shf_reg;
 reg  [ 2: 0] tx_d_count;

  assign tx_ready_o        = (state == IDLE);
  assign tx_busy_o         = ~tx_ready_o;
  assign tx_data_o         = (state == ST  )? (1'b0) :
	                     (state == DATA)? (tx_d_shf_reg[0]) :
			     (1'b1);

  assign tx_d_shf_en       = ((baud_tick_i) & (state == DATA));
  assign tx_d_fin          = ((baud_tick_i) & (state == DATA) & (tx_d_count == 3'd7));

  assign tx_d_count_next   = (tx_ready_o)?  (3'd0) : 
	                     (tx_d_shf_en)? (tx_d_count + 1'b1) : 
			     (tx_d_count);

  assign tx_d_shf_reg_next = (tx_ready_o)?  (tx_data_i) : 
	                     (tx_d_shf_en)? (tx_d_shf_reg >> 1) : 
			     (tx_d_shf_reg) ;

  always@(posedge clk, negedge reset_n) begin
    if(~reset_n)
      state <= IDLE;
    else
      state <= state_next;
  end

  always@(*) begin
    case(state)
     IDLE : state_next = (tx_start_i )?  (ST)   : (IDLE);
     ST   : state_next = (baud_tick_i)?  (DATA) : (ST);
     DATA : state_next = (tx_d_fin   )?  (DONE) : (DATA);
     DONE : state_next = (baud_tick_i)?  (IDLE) : (DONE);
    endcase
  end

  always@(posedge clk, negedge reset_n) begin
    if(~reset_n) begin
       tx_d_shf_reg <= 0;
       tx_d_count   <= 0;
    end
    else begin
       tx_d_shf_reg <= tx_d_shf_reg_next;
       tx_d_count   <= tx_d_count_next;
    end
  end

endmodule
```

Baud Rate Gen :
```verilog
module baud_rate_gen (
 input          clk,
 input          reset_n,
 input          tx_busy_i,
 input  [15: 0] baud_div_i,
 output         baud_tick_o
);

wire [15: 0] baud_count_next;
wire         baud_tick_next;
reg  [15: 0] baud_count;
reg          baud_tick;

assign baud_tick_o     =  baud_tick;
assign baud_tick_next  = (baud_count == baud_div_i - 16'd3);
assign baud_count_next = (baud_tick_next)? (16'd0) : 
	                 (tx_busy_i     )? (baud_count + 1'b1) :
			 (16'd0);


always@(posedge clk , negedge reset_n) begin
 if(~reset_n) begin
   baud_count   <= 0;
   baud_tick    <= 0;
 end
 else begin
   baud_count   <= baud_count_next;
   baud_tick    <= baud_tick_next;
 end
end 

endmodule
```

Monitor :
```verilog
module uart_monitor ();

  parameter ERROR_VALUE1 = 104160 * 0.02 ; // baud rate = 9600 , +-2% 
  parameter ERROR_VALUE2 = 8680 * 0.02   ; // baud rate = 112500 , +-2% 

  reg [ 9: 0]  rx_data;
  reg [ 7: 0]  golden_data;
  time         t_f1, t_f2; 
  integer      t_diff;
  reg [ 1: 0]  t_count;

  wire         baud_tick;
  wire         shf_en;

  event        event_1, event_2;

  assign baud_tick = `TOP.baud_tick;
  assign shf_en    = `TOP.uart_tx.tx_d_shf_en;

  //--- DATA Monitor ---//
  // catch tx data
  always@(posedge baud_tick) begin
    rx_data    = {`TB.tx_data_output , rx_data[9:1]};
  end

  // catch golden data
  always@(posedge `TB.tx_start) begin
    golden_data = `TB.tx_data_input;
  end

  // compaire data
  always@(posedge `TB.tx_busy) begin
    wait(~`TB.tx_busy);
    wait(`TB.tx_ready);
    if(golden_data == rx_data[8:1]) begin
      $display("[MON] [PASS] Data TX Successful !!! Golden Data = %h , UART RX Data = %h ! " ,golden_data ,rx_data[8:1] );
    end
    else begin
      $display("[MON] [FAIL] Data TX Fail !!! Golden Data = %h , UART RX Data = %h ! " ,golden_data ,rx_data[8:1] );
      #100;
      $finish;
    end
  end


  // Baud Rate Monitor

  initial begin
   t_f1    = 0;
   t_f2    = 0;
   t_count = 0;
  end

  always@(posedge shf_en) begin
    t_f1 <= $time;
    t_f2 <= t_f1;
    -> event_1;
  end

  always@(posedge shf_en) begin
    # 10;
    if(t_count == 2'd1) begin
      -> event_2;
      t_diff = t_f1 - t_f2;

      if(`TOP.baud_div == 16'd5208) begin // 9600
        if( (t_diff <= 104160 + ERROR_VALUE1) & (t_diff >= 104160 - ERROR_VALUE1))
	  $display("[MON] [PASS] Baud Rate OK ! Golden Baud Rate =   9600 , Error Value +-2 pct. , UART Baud Rate = %d ! " , (1_000_000_000/t_diff));
        else begin
	  $display("[MON] [FAIL] Baud Rate Fail ! Golden Baud Rate =   9600 , Error Value +-2 pct. ,UART Baud Rate = %d ! ", (1_000_000_000/t_diff) );
          #100;
          $finish;
        end
      end

      if(`TOP.baud_div == 16'd434) begin // 115200
        if( (t_diff <= 8680 + ERROR_VALUE2) & (t_diff >= 8680 - ERROR_VALUE2))
	  $display("[MON] [PASS] Baud Rate OK ! Golden Baud Rate =   115200 , Error Value +-2 pct. , UART Baud Rate = %d ! " , (1_000_000_000/t_diff));
        else begin
	  $display("[MON] [FAIL] Baud Rate Fail ! Golden Baud Rate =  115200 , Error Value +-2 pct. ,UART Baud Rate = %d ! ", (1_000_000_000/t_diff) );
          #100;
          $finish;
        end
      end

      t_count <= 0;
    end
    else
      t_count <= t_count + 1;
  end

endmodule
```

###
\[Simulation Waveform\]
Name      : PATTERN01 
Baud Rate : 9600 
Test Data : 8'hFF -> 8'h00 -> 8'hFF
![PAT1_waveform](https://github.com/black-could/black-could.github.io/blob/main/images/PAT1.png)

Name      : PATTERN04 
Baud Rate : 115200 
Test Data : 8'h55 -> 8'hAA -> 8'h55
![PAT4_waveform](https://github.com/black-could/black-could.github.io/blob/main/images/PAT4.png)

###
\[Yosys synthesis report\]
![yosys_syn_report](https://github.com/black-could/black-could.github.io/blob/main/images/Yosys_report.png)

###
\[OpenSTA timming report\]
![OpenSTA_timming_report1](https://github.com/black-could/black-could.github.io/blob/main/images/timming_report1.png)
![OpenSTA_timming_report2](https://github.com/black-could/black-could.github.io/blob/main/images/timming_report2.png)
![OpenSTA_timming_report3](https://github.com/black-could/black-could.github.io/blob/main/images/timming_report3.png)
