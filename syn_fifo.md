---

title: syn_FIFO
toc: true
---
## \[Preface\]
#### 此專案目的在於自行走過設計規格至STA流程 .
#### 原本想說驗證部分就開始用SystemVerilog練習 , 結果折騰了一個下午證實了一件悲慘的事 .
#### lcarus verilog只支援SystemVerilog部分語法 , 且不支援其驗證架構 . 
#### 也就是說如果當我把驗證環境建構成interface , tb , scoreboard , driver在編譯中會有問題 .
#### 所以作罷 , 找個時間會去EDA playground 試試 . (下面結構樹的圖 , env 資料夾請忽略)
#### 在此紀錄RTL code , testbench , monitor , simulation waveform , yosys synthesis report , openSTA timming report .
#### script不紀錄 .

{:toc}
###
## \[Info.\]
IC design flow      : spec. --> RTL --> simulation --> synthesis --> STA

compiler       tool : lcarus verilog (iverilog)

simulation     tool : vvp

synthesis      tool : Yosys

STA            tool : OpenSTA

waveform check tool : gtkwave

library             : sky130_fd_sc_hd__tt_025C_1v80.lib (sky130 , open source pdk)

## \[Specification\]
FIFO Data Deep     : 16 

FIFO Data Width    : 8 bit

Almost full/empty  : 1

Main clk           : 50M Hz

## \[Interface\]
SPI Master

| Singnal Name       | Direction | Width  | Note                                                         |
|--------------------|-----------|--------|--------------------------------------------------------------|
| clk                | input     | 1 bit  | NA                                                           |
| rst_n              | input     | 1 bit  | NA                                                           |
| w_en_i             | input     | 1 bit  | NA                                                           |
| r_en_i             | input     | 1 bit  | NA                                                           |
| d_i                | input     | 8 bit  | NA                                                           |
| d_o                | output    | 8 bit  | NA                                                           |
| full_f_o           | output    | 1 bit  | NA                                                           |
| empty_f_o          | output    | 1 bit  | NA                                                           |
| al_full_f_o        | output    | 1 bit  | NA                                                           |
| al_empty_f_o       | output    | 1 bit  | NA                                                           |

## \[Action\]
when write : set w_en_i high (1 clk), set r_en_i low (1 clk), d_i input your data
when read  : set w_en_i low  (1 clk), set r_en_i low (1 clk), d_o output data

## \[Architecture_Blocks\]
![syn_fifo_block](/images/syn_fifo/syn_fifo_blocks.png)


## \[Pattern_List\]

| Define Name | Pattern Name       | Test Data                                 | Note                                                                |
|-------------|--------------------|-------------------------------------------|---------------------------------------------------------------------|
| PATTERN01   | pattern01.v        | random data                               | Write data continuously , 15 times and then read data from syn_fifo |
| PATTERN02   | pattern02.v        | random data                               | Random read/write , 50 times                                        |
| NA          | default_pattern.v  | 8'h00 -> 8'h55 -> 8'hAA -> 8'hFF -> 8'h00 | NA                                                                  |

## \[Environment_Architecture\]
![syn_fifo_tree](/images/syn_fifo/syn_fifo_tree.png)

## \[RTL code\]
Testbench :
```verilog
`timescale 1ns/10ps

module syn_fifo_tb;

// Monitor
`include "../monitor/syn_fifo_monitor.v"

//Parameter
parameter D_W = 8, FIFO_DEEP = 16, AL_DIFF = 1;

//Clock Parameter
parameter CLK_50M_HALF_PIR = 10 ;

//Dump
initial begin
 $dumpfile("dump.vcd");
 $dumpvars(0, syn_fifo_tb);
end

//Output
 wire [ 7: 0] d_o;
 wire         full_f_o;
 wire         empty_f_o;
 wire         al_full_f_o;
 wire         al_empty_f_o;
 
//Input
 reg  [ 7: 0] d_i;
 reg          w_en_i;
 reg          r_en_i;

//Connect
 wire         clk;

//Clock
 reg          clk_50M;

initial clk_50M = 1'b0;
always  #CLK_50M_HALF_PIR clk_50M = ~clk_50M;

//Reset
 reg          reset_n;


initial begin
  reset_n  = 1'b1;
  #10;
  reset_n  = 1'b0;
  #CLK_50M_HALF_PIR;
  #CLK_50M_HALF_PIR;
  reset_n = 1'b1;
  #CLK_50M_HALF_PIR;
  #CLK_50M_HALF_PIR;
end

// Circuit
assign clk = clk_50M;

SYN_FIFO  #(
 .D_W     (D_W),
 .FIFO_D  (FIFO_DEEP),
 .AL_DIFF (AL_DIFF)) 
syn_fifo (
 .clk          (clk),
 .rst_n        (reset_n),
 .w_en_i       (w_en_i),
 .r_en_i       (r_en_i),
 .d_i          (d_i),
 .full_f_o     (full_f_o),
 .empty_f_o    (empty_f_o),
 .al_full_f_o  (al_full_f_o),
 .al_empty_f_o (al_empty_f_o),
 .d_o          (d_o)
);

// Initial Value
initial begin
  w_en_i = 0;
  r_en_i = 0;
  d_i    = 0;
end

// Pattern List
`ifdef PATTERN01
  `include "../pattern/pattern01.v"
`elsif PATTERN02
  `include "../pattern/pattern02.v"
`else
  `include "../pattern/default_pattern.v"
`endif

// Task
task control_syn_fifo (
	input  [D_W-1:0] w_data,
	input            w_en, r_en
);
begin
  @(posedge clk);
  d_i                <= w_data;
  w_en_i             <= w_en;
  r_en_i             <= r_en;
  @(posedge clk);
  w_en_i             <= 1'b0;
  r_en_i             <= 1'b0;
end
endtask

endmodule

```

SYN FIFO :
```verilog
module SYN_FIFO #(
parameter D_W      = 8,
parameter FIFO_D   = 16,
parameter AL_DIFF  = 1) 
(
input              clk,
input              rst_n,
input              w_en_i,
input              r_en_i,
input  [D_W-1: 0]  d_i,
output	           full_f_o,
output             empty_f_o,
output	           al_full_f_o,
output             al_empty_f_o,
output [D_W-1: 0]  d_o
);
  
parameter FIFO_D_LOG2 = $clog2(FIFO_D);


reg  [FIFO_D_LOG2     : 0] w_count;
reg  [FIFO_D_LOG2     : 0] r_count;
wire [FIFO_D_LOG2     : 0] w_count_next;
wire [FIFO_D_LOG2     : 0] r_count_next;
reg  [FIFO_D - 1      : 0] w_en;

reg  [D_W - 1 : 0]         d_reg      [FIFO_D - 1 : 0];
wire [D_W - 1 : 0]         d_reg_next [FIFO_D - 1 : 0];

reg  [D_W - 1 : 0]         d_reg_o;


assign d_o          = d_reg_o;
assign w_count_next = (w_en_i)? (w_count + 1'b1) : (w_count);
assign r_count_next = (r_en_i)? (r_count + 1'b1) : (r_count);

assign full_f_o      = (w_count[FIFO_D_LOG2] != r_count[FIFO_D_LOG2]) & 
	               (w_count[FIFO_D_LOG2-1:0] == r_count[FIFO_D_LOG2-1:0]);
assign empty_f_o     = (w_count == r_count);
assign al_full_f_o   = (w_count[FIFO_D_LOG2] != r_count[FIFO_D_LOG2]) & 
	               (w_count[FIFO_D_LOG2-1:0] + AL_DIFF == r_count[FIFO_D_LOG2-1:0]);
assign al_empty_f_o  = (w_count == r_count + AL_DIFF);

always@(*) begin : DECODER
        w_en = 0;
        w_en[w_count[FIFO_D_LOG2 - 1 : 0]] = 1'b1;
end

always@(*) begin : MUX
        d_reg_o = d_reg[r_count[FIFO_D_LOG2 - 1 : 0]];
end


genvar i;

generate
	for(i=0 ;i < FIFO_D ;i=i+1) begin : DATA_REGISTER

	        assign d_reg_next[i] = (w_en[i])? (d_i) : (d_reg[i]);	

		always@(posedge clk, negedge rst_n) begin
		  if(~rst_n)
		    d_reg[i] <= 0;
		  else
		    d_reg[i] <= d_reg_next[i];
		end
	end
endgenerate

always@(posedge clk, negedge rst_n) begin
	if(~rst_n) begin
		w_count <= 0;
		r_count <= 0;
	end
	else begin
		w_count <= w_count_next;
		r_count <= r_count_next;
	end
end

endmodule

```

Monitor :
```verilog
module syn_fifo_monitor ();

  parameter D_W = 8, FIFO_DEEP = 16, DEEP_BIT = $clog2(FIFO_DEEP), AL_NUM = 1;
  
  reg  [D_W-1: 0]   golden_d      [FIFO_DEEP-1:0];  
  reg  [D_W-1: 0]   syn_fifo_o_d;  
  reg               data_check_f;
  reg               full_check_f;
  reg               empty_check_f;
  reg               al_full_check_f;
  reg               al_empty_check_f;
  reg               flag_err_f;

  reg  [DEEP_BIT:0] w_pointer, r_pointer;
  event             data_check_fail, full_fail, empty_fail, al_full_fail, al_empty_fail, flag_err;


  //--- DATA Monitor ---//
  // catch real data
  always@(posedge syn_fifo_tb.r_en_i, negedge syn_fifo_tb.reset_n) begin
    if(~syn_fifo_tb.reset_n) begin
	r_pointer    = 0;
	data_check_f = 0;
	syn_fifo_o_d = 0;
    end
    else begin
        data_check_f = 1'b1;
	syn_fifo_o_d = syn_fifo_tb.d_o;
    end
  end

  // catch golden data
  always@(posedge syn_fifo_tb.w_en_i, negedge syn_fifo_tb.reset_n) begin
    if(~syn_fifo_tb.reset_n) begin
	for(w_pointer = 0 ; w_pointer < FIFO_DEEP ; w_pointer = w_pointer + 1) begin
		 golden_d[w_pointer] = 0;
	end
	w_pointer = 0;
    end
    else begin
    	golden_d[w_pointer[DEEP_BIT-1:0]] = syn_fifo_tb.d_i;
    	w_pointer           = w_pointer + 1;
    end
  end

  // compaire data
  always@(posedge data_check_f) begin
   #5;
    if(golden_d[r_pointer[DEEP_BIT-1:0]] == syn_fifo_o_d) begin
      $display("[MON] [PASS] Data Successful !!! Golden Data = %h , FIFO read Data = %h ! " ,golden_d[r_pointer[DEEP_BIT-1:0]] ,syn_fifo_o_d );
    end
    else begin
      -> data_check_fail;
      $display("[MON] [FAIL] Data Fail !!!  Golden Data = %h , FIFO read Data = %h ! " ,golden_d[r_pointer[DEEP_BIT-1:0]] ,syn_fifo_o_d );
      #100;
      $finish;
    end

    r_pointer    = r_pointer + 1;
    data_check_f = 0;

  end

  //--- full signal Monitor ---//
  always@(posedge syn_fifo_tb.clk, negedge syn_fifo_tb.reset_n) begin
	  if(~syn_fifo_tb.reset_n) begin
		full_check_f = 0;
	  end
	  else begin
		if((syn_fifo_tb.full_f_o) | (((w_pointer[DEEP_BIT] != r_pointer[DEEP_BIT]) & (w_pointer[DEEP_BIT-1:0] == r_pointer[DEEP_BIT-1:0])) )) begin
		  full_check_f = 1;
		end
		else begin
	          full_check_f = 0;
		end
	  end
  end
  

  always@(posedge full_check_f) begin
   #5;
   if(syn_fifo_tb.reset_n) begin
    if( ~((w_pointer[DEEP_BIT] != r_pointer[DEEP_BIT]) & (w_pointer[DEEP_BIT-1:0] == r_pointer[DEEP_BIT-1:0])) | ~syn_fifo_tb.full_f_o ) begin
      -> full_fail;
      $display("[MON] [FAIL] Full signal Fail !!! NOW FIFO full signal = %d , monitor w_pointer = %d and r_pointer = %d " , syn_fifo_tb.full_f_o, w_pointer, r_pointer);
      #100;
      $finish;
    end
   end
   end

  //--- empty signal Monitor ---//
  always@(posedge syn_fifo_tb.clk, negedge syn_fifo_tb.reset_n) begin
	  if(~syn_fifo_tb.reset_n) begin
		empty_check_f = 0;
	  end
	  else begin
		if((syn_fifo_tb.empty_f_o) | ((w_pointer == r_pointer) )) begin
		  empty_check_f = 1;
		end
		else begin
	          empty_check_f = 0;
		end
	  end
  end
  

  always@(posedge empty_check_f) begin
   #5;
   if(syn_fifo_tb.reset_n) begin
    if( ~((w_pointer == r_pointer)) | ~syn_fifo_tb.empty_f_o ) begin
      -> empty_fail;
      $display("[MON] [FAIL] Empty signal Fail !!! NOW FIFO empty signal = %d , monitor w_pointer = %d and r_pointer = %d " , syn_fifo_tb.empty_f_o, w_pointer, r_pointer);
      #100;
      $finish;
    end
   end
   end

  //--- almost full signal Monitor ---//
  always@(posedge syn_fifo_tb.clk, negedge syn_fifo_tb.reset_n) begin
	  if(~syn_fifo_tb.reset_n) begin
		al_full_check_f = 0;
	  end
	  else begin
		if((syn_fifo_tb.al_full_f_o) | (((w_pointer[DEEP_BIT] != r_pointer[DEEP_BIT]) & (w_pointer[DEEP_BIT-1:0] + AL_NUM == r_pointer[DEEP_BIT-1:0])) )) begin
		  al_full_check_f = 1;
		end
		else begin
	          al_full_check_f = 0;
		end
	  end
  end
  

  always@(posedge al_full_check_f) begin
   #5;
   if(syn_fifo_tb.reset_n) begin
    if( ~((w_pointer[DEEP_BIT] != r_pointer[DEEP_BIT]) & (w_pointer[DEEP_BIT-1:0] + AL_NUM == r_pointer[DEEP_BIT-1:0])) | ~syn_fifo_tb.al_full_f_o ) begin
      -> al_full_fail;
      $display("[MON] [FAIL] Almost Full signal Fail !!! NOW FIFO almost full signal = %d , monitor w_pointer = %d and r_pointer = %d " , syn_fifo_tb.al_full_f_o, w_pointer, r_pointer);
      #100;
      $finish;
    end
   end
   end

  //--- almost empty signal Monitor ---//
  always@(posedge syn_fifo_tb.clk, negedge syn_fifo_tb.reset_n) begin
	  if(~syn_fifo_tb.reset_n) begin
		al_empty_check_f = 0;
	  end
	  else begin
		if((syn_fifo_tb.al_empty_f_o) | ((w_pointer == r_pointer + AL_NUM) )) begin
		  al_empty_check_f = 1;
		end
		else begin
	          al_empty_check_f = 0;
		end
	  end
  end
  

  always@(posedge al_empty_check_f) begin
   #5;
   if(syn_fifo_tb.reset_n) begin
    if( ~((w_pointer == r_pointer + AL_NUM)) | ~syn_fifo_tb.al_empty_f_o ) begin
      -> al_empty_fail;
      $display("[MON] [FAIL] Alomst Empty signal Fail !!! NOW FIFO almost empty signal = %d , monitor w_pointer = %d and r_pointer = %d " , syn_fifo_tb.al_empty_f_o, w_pointer, r_pointer);
      #100;
      $finish;
    end
   end
   end

  //--- flag Monitor ---//
  always@(posedge syn_fifo_tb.clk, negedge syn_fifo_tb.reset_n) begin
    #5;
    if(~syn_fifo_tb.reset_n)
      flag_err_f = 0;
    else begin 
      case({syn_fifo_tb.full_f_o,syn_fifo_tb.empty_f_o,syn_fifo_tb.al_full_f_o,syn_fifo_tb.al_empty_f_o})
	            4'b1111: flag_err_f = 1; 
		    4'b1110: flag_err_f = 1;
		    4'b1101: flag_err_f = 1;
		    4'b1100: flag_err_f = 1;
		    4'b1011: flag_err_f = 1;
		    4'b1010: flag_err_f = 1;
		    4'b1001: flag_err_f = 1;
		    4'b1000: flag_err_f = 0;
		    4'b0111: flag_err_f = 1;
		    4'b0110: flag_err_f = 1;
		    4'b0101: flag_err_f = 1;
		    4'b0100: flag_err_f = 0;
		    4'b0011: flag_err_f = 1;
		    4'b0010: flag_err_f = 0;
		    4'b0001: flag_err_f = 0;
		    4'b0000: flag_err_f = 0;
      endcase
    end
  end


  always@(posedge flag_err_f) begin
      -> flag_err;
      $display("[MON] [FAIL] Flag signal Fail !!! NOW Full signal = %d ,Empty signal = %d ,Almost Full signal = %d ,Almost Empty signal = %d  " , syn_fifo_tb.full_f_o,syn_fifo_tb.empty_f_o,syn_fifo_tb.al_full_f_o,syn_fifo_tb.al_empty_f_o);
      #100;
      $finish;	
  end

endmodule

```

## \[Simulation Waveform\]
Name      : Default_PATTERN 

Test Data : 8'h00 -> 8'h55 -> 8'hAA -> 8'hFF -> 8'h00

![PAT_DE_waveform](/images/syn_fifo/DE_PAT.png)

Name      : PATTERN02

Test Data : Random

![PAT02_waveform](/images/syn_fifo/PAT02.png)

## \[Yosys synthesis report\]
![yosys_syn_report](/images/syn_fifo/yosys_report.png)

## \[OpenSTA timming report\]
![OpenSTA_timming_report_01](/images/syn_fifo/timming_report_01.png)
![OpenSTA_timming_report_02](/images/syn_fifo/timming_report_02.png)
![OpenSTA_timming_report_03](/images/syn_fifo/timming_report_03.png)
![OpenSTA_timming_report_04](/images/syn_fifo/timming_report_04.png)
![OpenSTA_timming_report_05](/images/syn_fifo/timming_report_05.png)
![OpenSTA_timming_report_06](/images/syn_fifo/timming_report_06.png)
