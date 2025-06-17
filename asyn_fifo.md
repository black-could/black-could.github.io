---

title: asyn_FIFO
toc: true
---
## \[Preface\]
#### 此專案目的在於自行走過設計規格至STA流程 .
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

Write clk          : basis pattern

Read clk           : basis pattern

## \[Interface\]
SPI Master

| Singnal Name       | Direction | Width  | Note                                                         |
|--------------------|-----------|--------|--------------------------------------------------------------|
| rd_clk             | input     | 1 bit  | NA                                                           |
| wr_clk             | input     | 1 bit  | NA                                                           |
| rd_rst_n           | input     | 1 bit  | NA                                                           |
| wr_rst_n           | input     | 1 bit  | NA                                                           |
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
![asyn_fifo_block](/images/asyn_fifo/asyn_fifo_block.png)


## \[Pattern_List\]

| Define Name | Pattern Name       | Test Data                                 | Test Frequen                       | Note                                         |
|-------------|--------------------|-------------------------------------------|------------------------------------|---------------------------------------------|
| PATTERN01   | pattern01.v        | random data                               | wr_clk =  50M Hz , rd_clk = 500M Hz | Write data continuously , 15 times and then read data from syn_fifo |
| PATTERN02   | pattern02.v        | random data                               | wr_clk = 500M Hz , rd_clk =  50M Hz | Write data continuously , 15 times and then read data from syn_fifo |
| PATTERN03   | pattern03.v        | random data                               | wr_clk = 200M Hz , rd_clk = 142M Hz | Write data continuously , 15 times and then read data from syn_fifo |
| PATTERN04   | pattern04.v        | random data                               | wr_clk =  50M Hz , rd_clk = 500M Hz | Random read/write , 50 times                                        |
| PATTERN05   | pattern05.v        | random data                               | wr_clk = 500M Hz , rd_clk =  50M Hz | Random read/write , 50 times                                        |
| PATTERN06   | pattern06.v        | random data                               | wr_clk = 200M Hz , rd_clk = 142M Hz | Random read/write , 50 times                                        |
| NA          | default_pattern.v  | 8'h00 -> 8'h55 -> 8'hAA -> 8'hFF -> 8'h00 | wr_clk = 142M Hz , rd_clk = 200M Hz | NA                                           |

## \[Environment_Architecture\]
![asyn_fifo_tree](/images/asyn_fifo/asyn_fifo_tree.png)

## \[RTL code\]
Testbench :
```verilog
`timescale 1ns/10ps

module asyn_fifo_tb;

// Monitor
`include "../monitor/asyn_fifo_monitor.v"

//Parameter
parameter D_W = 8, FIFO_DEEP = 16, AL_DIFF = 1;

//Clock Parameter
parameter CLK_50M_HALF_PIR  = 10 ;
parameter CLK_500M_HALF_PIR = 1  ;
parameter CLK_200M_HALF_PIR = 2.5;
parameter CLK_142M_HALF_PIR = 3.5;

//WDT Parameter
parameter TB_WDT = 5_000_000;

//Dump
initial begin
 $dumpfile("dump.vcd");
 $dumpvars(0, asyn_fifo_tb);
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
 wire         wr_clk;
 wire         rd_clk;

//Clock
 reg          clk_50M;
 reg          clk_500M;
 reg          clk_200M;
 reg          clk_142M;

initial clk_50M = 1'b0;
always  #CLK_50M_HALF_PIR clk_50M = ~clk_50M;

initial clk_500M = 1'b0;
always  #CLK_500M_HALF_PIR clk_500M = ~clk_500M;

initial clk_200M = 1'b0;
always  #CLK_200M_HALF_PIR clk_200M = ~clk_200M;

initial clk_142M = 1'b0;
always  #CLK_142M_HALF_PIR clk_142M = ~clk_142M;

`ifdef PATTERN01
  assign wr_clk = clk_50M;
  assign rd_clk = clk_500M;
`elsif PATTERN02
  assign wr_clk = clk_500M;
  assign rd_clk = clk_50M;
`elsif PATTERN03
  assign wr_clk = clk_200M;
  assign rd_clk = clk_142M;
`elsif PATTERN04
  assign wr_clk = clk_50M;
  assign rd_clk = clk_500M;
`elsif PATTERN05
  assign wr_clk = clk_500M;
  assign rd_clk = clk_50M;
`elsif PATTERN06
  assign wr_clk = clk_200M;
  assign rd_clk = clk_142M;
`else
  assign wr_clk = clk_142M;
  assign rd_clk = clk_200M;
`endif

//Reset
 reg          wr_reset_n;
 reg          rd_reset_n;

 initial begin
   wr_reset_n  = 1'b1;
   #2;
   wr_reset_n  = 1'b0;
   @(posedge wr_clk);
   wr_reset_n = 1'b1;
   @(posedge wr_clk);
   @(posedge wr_clk);
 end
 
 initial begin
   rd_reset_n  = 1'b1;
   #2;
   rd_reset_n = 1'b0;
   @(posedge rd_clk);
   rd_reset_n = 1'b1;
   @(posedge rd_clk);
   @(posedge rd_clk);
 end


// Circuit

ASYN_FIFO  #(
 .D_W     (D_W),
 .FIFO_D  (FIFO_DEEP),
 .AL_DIFF (AL_DIFF)) 
asyn_fifo (
 .wr_clk       (wr_clk),
 .rd_clk       (rd_clk),
 .wr_rst_n     (wr_reset_n),
 .rd_rst_n     (rd_reset_n),
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
`elsif PATTERN03
  `include "../pattern/pattern03.v"
`elsif PATTERN04
  `include "../pattern/pattern04.v"
`elsif PATTERN05
  `include "../pattern/pattern05.v"
`elsif PATTERN06
  `include "../pattern/pattern06.v"
`else
  `include "../pattern/default_pattern.v"
`endif

// Task
task write_fifo (
	input  [D_W-1:0] w_data,
	input            w_en
);
begin
  @(posedge wr_clk);
  d_i                <= w_data;
  w_en_i             <= w_en;
  @(posedge wr_clk);
  w_en_i             <= 1'b0;
end
endtask

task read_fifo (
	input            r_en
);
begin
  @(posedge rd_clk);
  r_en_i             <= r_en;
  @(posedge rd_clk);
  r_en_i             <= 1'b0;
end
endtask

// WDT
initial begin
 #TB_WDT;
 $display("[TB] Simulation Timming over !!!");
 $finish;
end

endmodule

```

ASYN FIFO :
```verilog
module ASYN_FIFO #(
parameter D_W      = 8,
parameter FIFO_D   = 16,
parameter AL_DIFF  = 1) 
(
input              wr_clk,
input              rd_clk,
input              wr_rst_n,
input              rd_rst_n,
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

wire [      1 : 0]         cdc_syn_rd_clk_next [FIFO_D_LOG2 : 0];
wire [      1 : 0]         cdc_syn_wr_clk_next [FIFO_D_LOG2 : 0];

reg  [      1 : 0]         cdc_syn_rd_clk [FIFO_D_LOG2 : 0];
reg  [      1 : 0]         cdc_syn_wr_clk [FIFO_D_LOG2 : 0];


wire [FIFO_D_LOG2 : 0]         w_count_gray_code;
wire [FIFO_D_LOG2 : 0]         w_count_gray_code_syn_rd_clk;
wire [FIFO_D_LOG2 : 0]         w_count_syn_rd_clk;
wire [FIFO_D_LOG2 : 0]         r_count_gray_code;
wire [FIFO_D_LOG2 : 0]         r_count_gray_code_syn_wr_clk;
wire [FIFO_D_LOG2 : 0]         r_count_syn_wr_clk;

integer                        iv, v, vi, vii;

assign d_o          = d_reg_o;
assign w_count_next = (w_en_i)? (w_count + 1'b1) : (w_count);
assign r_count_next = (r_en_i)? (r_count + 1'b1) : (r_count);

assign full_f_o      = (w_count[FIFO_D_LOG2]               != r_count_syn_wr_clk[FIFO_D_LOG2]) & 
	               (w_count[FIFO_D_LOG2-1:0]           == r_count_syn_wr_clk[FIFO_D_LOG2-1:0]);
assign al_full_f_o   = (w_count[FIFO_D_LOG2]               != r_count_syn_wr_clk[FIFO_D_LOG2]) & 
	               (w_count[FIFO_D_LOG2-1:0] + AL_DIFF == r_count_syn_wr_clk[FIFO_D_LOG2-1:0]);

assign empty_f_o     = (w_count_syn_rd_clk == r_count);
assign al_empty_f_o  = (w_count_syn_rd_clk == r_count + AL_DIFF);


always@(*) begin : DECODER
        w_en = 0;
        w_en[w_count[FIFO_D_LOG2 - 1 : 0]] = 1'b1;
end

always@(*) begin : MUX
        d_reg_o = d_reg[r_count[FIFO_D_LOG2 - 1 : 0]];
end

genvar i, ii, iii;

generate
	for(i=0 ;i < FIFO_D ;i=i+1) begin : DATA_REGISTER

	        assign d_reg_next[i] = (w_en[i])? (d_i) : (d_reg[i]);	

		always@(posedge wr_clk, negedge wr_rst_n) begin
		  if(~wr_rst_n)
		    d_reg[i] <= 0;
		  else
		    d_reg[i] <= d_reg_next[i];
		end
	end

	for(ii=0 ; ii <= FIFO_D_LOG2 ; ii=ii+1) begin : CDC_2_STAGE_REG
		assign cdc_syn_rd_clk_next[ii] = {cdc_syn_rd_clk[ii][0],w_count_gray_code[ii]};
		assign cdc_syn_wr_clk_next[ii] = {cdc_syn_wr_clk[ii][0],r_count_gray_code[ii]};
	end

	for(iii=0 ; iii <= FIFO_D_LOG2 ; iii=iii+1) begin : COMBAIN_CDC_AFT_WIRE
		assign w_count_gray_code_syn_rd_clk[iii] = cdc_syn_rd_clk[iii][1];
		assign r_count_gray_code_syn_wr_clk[iii] = cdc_syn_wr_clk[iii][1];
	end


endgenerate

Ten2Gray #(.D_W(FIFO_D_LOG2)) Wr_Ten2Gray (.d_i(w_count) , .d_o(w_count_gray_code)) ;
Ten2Gray #(.D_W(FIFO_D_LOG2)) Rd_Ten2Gray (.d_i(r_count) , .d_o(r_count_gray_code)) ;


Gray2Ten #(.D_W(FIFO_D_LOG2)) Wr_Gray2Ten (.d_i(r_count_gray_code_syn_wr_clk) , .d_o(r_count_syn_wr_clk)) ;
Gray2Ten #(.D_W(FIFO_D_LOG2)) Rd_Gray2Ten (.d_i(w_count_gray_code_syn_rd_clk) , .d_o(w_count_syn_rd_clk)) ;


always@(posedge wr_clk, negedge wr_rst_n) begin
	if(~wr_rst_n) begin
		w_count           <= 0;
		for(iv=0 ; iv <= FIFO_D_LOG2 ; iv = iv + 1) begin
		  cdc_syn_wr_clk[iv] <= 0;
		end
	end
	else begin
	       w_count           <= w_count_next;
	       for(v=0 ; v <= FIFO_D_LOG2 ; v = v + 1) begin
                 cdc_syn_wr_clk[v] <= cdc_syn_wr_clk_next[v];
	      end
	end
end

always@(posedge rd_clk, negedge rd_rst_n) begin
	if(~rd_rst_n) begin
		r_count           <= 0;
                for(vi=0 ; vi <= FIFO_D_LOG2 ; vi = vi + 1) begin
		  cdc_syn_rd_clk[vi] <= 0;
	        end
	end
	else begin
		r_count           <= r_count_next;
                for(vii=0 ; vii <= FIFO_D_LOG2 ; vii = vii + 1) begin
		  cdc_syn_rd_clk[vii] <= cdc_syn_rd_clk_next[vii];
	        end
	end
end

endmodule

```

Ten2Gray :
```verilog
module Ten2Gray #(parameter D_W = 8) (
 input  [D_W : 0] d_i,
 output [D_W : 0] d_o
);

 genvar i;

 assign d_o[D_W] = d_i[D_W];

 generate 
   for(i = 0 ; i < D_W ; i = i + 1) begin
	assign d_o[i] = d_i[i+1] ^ d_i[i];
   end
 endgenerate 

endmodule

```

Gray2Ten :
```veriog
module Gray2Ten #(parameter D_W = 8) (
 input  [D_W : 0] d_i,
 output [D_W : 0] d_o
);

 genvar i;

 assign d_o[D_W] = d_i[D_W];

 generate 
   for(i = D_W-1 ; i >= 0 ; i = i - 1) begin
	assign d_o[i] = d_o[i+1] ^ d_i[i];
   end
 endgenerate 

endmodule

```

Monitor :
```verilog
module asyn_fifo_monitor ();

  parameter D_W = 8, FIFO_DEEP = 16, DEEP_BIT = $clog2(FIFO_DEEP), AL_NUM = 1;
  
  reg  [D_W-1: 0]   golden_d      [FIFO_DEEP-1:0];  
  reg  [D_W-1: 0]   asyn_fifo_o_d;  
  reg               data_check_f;
  reg               full_check_f;
  reg               empty_check_f;
  reg               al_full_check_f;
  reg               al_empty_check_f;
  reg               flag_err_f;

  reg  [DEEP_BIT:0] w_pointer, r_pointer;
  reg  [DEEP_BIT:0] w_pointer_0, r_pointer_0;
  reg  [DEEP_BIT:0] w_pointer_syn_rd_clk, r_pointer_syn_wr_clk;
  event             data_check_fail, full_fail, empty_fail, al_full_fail, al_empty_fail, flag_err;


  //--- DATA Monitor ---//
  // catch real data
  always@(posedge asyn_fifo_tb.rd_clk, negedge asyn_fifo_tb.rd_reset_n) begin
    if(~asyn_fifo_tb.rd_reset_n) begin
	r_pointer    = 0;
	data_check_f = 0;
	asyn_fifo_o_d = 0;
    end
    else if (asyn_fifo_tb.r_en_i) begin
        data_check_f = 1'b1;
	asyn_fifo_o_d = asyn_fifo_tb.d_o;
    end
  end

  // catch golden data
  always@(posedge asyn_fifo_tb.wr_clk, negedge asyn_fifo_tb.wr_reset_n) begin
    if(~asyn_fifo_tb.wr_reset_n) begin
	for(w_pointer = 0 ; w_pointer < FIFO_DEEP ; w_pointer = w_pointer + 1) begin
		 golden_d[w_pointer] = 0;
	end
	w_pointer = 0;
    end
    else if (asyn_fifo_tb.w_en_i) begin
    	golden_d[w_pointer[DEEP_BIT-1:0]] = asyn_fifo_tb.d_i;
    	w_pointer           = w_pointer + 1;
    end
  end

  // compaire data
  always@(posedge data_check_f) begin
   #1;
    if(golden_d[r_pointer[DEEP_BIT-1:0]] == asyn_fifo_o_d) begin
      $display("[MON] [PASS] Data Successful !!! Golden Data = %h , FIFO read Data = %h ! " ,golden_d[r_pointer[DEEP_BIT-1:0]] ,asyn_fifo_o_d );
    end
    else begin
      -> data_check_fail;
      $display("[MON] [FAIL] Data Fail !!!  Golden Data = %h , FIFO read Data = %h ! " ,golden_d[r_pointer[DEEP_BIT-1:0]] ,asyn_fifo_o_d );
      #100;
      $finish;
    end

    r_pointer    = r_pointer + 1;
    data_check_f = 0;

  end

  //--- full signal Monitor ---//
  always@(posedge asyn_fifo_tb.wr_clk, negedge asyn_fifo_tb.wr_reset_n) begin
    if(~asyn_fifo_tb.wr_reset_n) begin
     r_pointer_0          <= 0;
     r_pointer_syn_wr_clk <= 0;
    end
    else begin
     r_pointer_0          <= r_pointer;
     r_pointer_syn_wr_clk <= r_pointer_0;
    end
  end

  always@(posedge asyn_fifo_tb.wr_clk, negedge asyn_fifo_tb.wr_reset_n) begin
	  if(~asyn_fifo_tb.wr_reset_n) begin
		full_check_f   <= 0;
	  end
	  else begin
		if((asyn_fifo_tb.full_f_o) | (((w_pointer[DEEP_BIT] != r_pointer_syn_wr_clk[DEEP_BIT]) & (w_pointer[DEEP_BIT-1:0] == r_pointer_syn_wr_clk[DEEP_BIT-1:0])) )) begin
                  if( ~((w_pointer[DEEP_BIT] != r_pointer_syn_wr_clk[DEEP_BIT]) & (w_pointer[DEEP_BIT-1:0] == r_pointer_syn_wr_clk[DEEP_BIT-1:0])) | ~asyn_fifo_tb.full_f_o ) begin
                    -> full_fail;
                    $display("[MON] [FAIL] Full signal Fail !!! NOW FIFO full signal = %d , monitor w_pointer = %d and r_pointer = %d " , asyn_fifo_tb.full_f_o, w_pointer, r_pointer_syn_wr_clk);
                    #100;
                    $finish;
                  end
		end
		else begin
	          full_check_f <= 0;
		end
	  end
  end


  //--- empty signal Monitor ---//
  always@(posedge asyn_fifo_tb.rd_clk, negedge asyn_fifo_tb.rd_reset_n) begin
    if(~asyn_fifo_tb.rd_reset_n) begin
     w_pointer_0          <= 0;
     w_pointer_syn_rd_clk <= 0;
    end
    else begin
     w_pointer_0          <= w_pointer;
     w_pointer_syn_rd_clk <= w_pointer_0;
    end
  end

  always@(posedge asyn_fifo_tb.rd_clk, negedge asyn_fifo_tb.rd_reset_n) begin
	  if(~asyn_fifo_tb.rd_reset_n) begin
		empty_check_f  <= 0;
	  end
	  else begin
		if((asyn_fifo_tb.empty_f_o) | ((w_pointer_syn_rd_clk == r_pointer) )) begin
                  if( ~((w_pointer_syn_rd_clk == r_pointer)) | ~asyn_fifo_tb.empty_f_o ) begin
                    -> empty_fail;
                    $display("[MON] [FAIL] Empty signal Fail !!! NOW FIFO empty signal = %d , monitor w_pointer = %d and r_pointer = %d " , asyn_fifo_tb.empty_f_o, w_pointer_syn_rd_clk, r_pointer);
                    #100;
                    $finish;
                  end
		end
		else begin
	          empty_check_f <= 0;
		end
	  end
  end


  //--- almost full signal Monitor ---//
  always@(posedge asyn_fifo_tb.wr_clk, negedge asyn_fifo_tb.wr_reset_n) begin
	  if(~asyn_fifo_tb.wr_reset_n) begin
		al_full_check_f   <= 0;
	  end
	  else begin
		if((asyn_fifo_tb.al_full_f_o) | (((w_pointer[DEEP_BIT] != r_pointer_syn_wr_clk[DEEP_BIT]) & (w_pointer[DEEP_BIT-1:0] + AL_NUM == r_pointer_syn_wr_clk[DEEP_BIT-1:0])) )) begin
                 if( ~((w_pointer[DEEP_BIT] != r_pointer_syn_wr_clk[DEEP_BIT]) & (w_pointer[DEEP_BIT-1:0] + AL_NUM == r_pointer_syn_wr_clk[DEEP_BIT-1:0])) | ~asyn_fifo_tb.al_full_f_o ) begin
                   -> al_full_fail;
                   $display("[MON] [FAIL] Almost Full signal Fail !!! NOW FIFO almost full signal = %d , monitor w_pointer = %d and r_pointer = %d " , asyn_fifo_tb.al_full_f_o, w_pointer, r_pointer_syn_wr_clk);
                   #100;
                   $finish;
                 end
		end
		else begin
	          al_full_check_f <= 0;
		end
	  end
  end
  

  //--- almost empty signal Monitor ---//
  always@(posedge asyn_fifo_tb.rd_clk, negedge asyn_fifo_tb.rd_reset_n) begin
	  if(~asyn_fifo_tb.rd_reset_n) begin
		al_empty_check_f <= 0;
	  end
	  else begin
		if((asyn_fifo_tb.al_empty_f_o) | ((w_pointer_syn_rd_clk == r_pointer + AL_NUM) )) begin
                   if( ~((w_pointer_syn_rd_clk == r_pointer + AL_NUM)) | ~asyn_fifo_tb.al_empty_f_o ) begin
                     -> al_empty_fail;
                     $display("[MON] [FAIL] Alomst Empty signal Fail !!! NOW FIFO almost empty signal = %d , monitor w_pointer = %d and r_pointer = %d " , asyn_fifo_tb.al_empty_f_o, w_pointer_syn_rd_clk, r_pointer);
                     #100;
                     $finish;
                   end
		end
		else begin
	          al_empty_check_f <= 0;
		end
	  end
  end
  


  //--- flag Monitor ---//
  always@(*) begin
    #1;
    if(~asyn_fifo_tb.wr_reset_n & ~asyn_fifo_tb.rd_reset_n)
      flag_err_f = 0;
    else begin 
      case({asyn_fifo_tb.full_f_o,asyn_fifo_tb.empty_f_o,asyn_fifo_tb.al_full_f_o,asyn_fifo_tb.al_empty_f_o})
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
      $display("[MON] [FAIL] Flag signal Fail !!! NOW Full signal = %d ,Empty signal = %d ,Almost Full signal = %d ,Almost Empty signal = %d  " , asyn_fifo_tb.full_f_o,asyn_fifo_tb.empty_f_o,asyn_fifo_tb.al_full_f_o,asyn_fifo_tb.al_empty_f_o);
      #100;
      $finish;	
  end

endmodule

```

## \[Simulation Waveform\]
Name      : Default_PATTERN 

Test Data : 8'h00 -> 8'h55 -> 8'hAA -> 8'hFF -> 8'h00

![PAT_DE_waveform](/images/asyn_fifo/DE_PAT.png)

Name      : PATTERN01

Test Data : Random

![PAT01_waveform](/images/asyn_fifo/PAT01.png)

Name      : PATTERN02

Test Data : Random

![PAT02_waveform](/images/asyn_fifo/PAT02.png)

Name      : PATTERN04

Test Data : Random

![PAT04_waveform](/images/asyn_fifo/PAT04.png)

## \[Yosys synthesis report\]
![yosys_syn_report](/images/asyn_fifo/yosys_report.png)

## \[OpenSTA timming report\]
![OpenSTA_timming_report_01](/images/asyn_fifo/timming_report_01.png)
![OpenSTA_timming_report_02](/images/asyn_fifo/timming_report_02.png)
![OpenSTA_timming_report_03](/images/asyn_fifo/timming_report_03.png)
![OpenSTA_timming_report_04](/images/asyn_fifo/timming_report_04.png)
![OpenSTA_timming_report_05](/images/asyn_fifo/timming_report_05.png)
![OpenSTA_timming_report_06](/images/asyn_fifo/timming_report_06.png)
![OpenSTA_timming_report_07](/images/asyn_fifo/timming_report_07.png)
![OpenSTA_timming_report_08](/images/asyn_fifo/timming_report_08.png)
![OpenSTA_timming_report_09](/images/asyn_fifo/timming_report_09.png)
