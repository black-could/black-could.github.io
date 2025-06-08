---

title: SPI
toc: true
---
## \[Preface\]
#### 此專案目的在於自行走過設計規格至STA流程 .
#### 驗證模擬部分為master和slave對接驗證 .
#### 合成部分有分個別合成與整合合成 .
#### 會做兩次原因是分開合成先確定合出來的電路是否有latch , 整合合成才能做接下來的STA .
#### 此專案使用Clock Gate Cell本來有使用latch , 但是由於此lib沒有latch所以改由DFF代替 .
#### Sky130其他lib有定義latch , 但是引用後發現.fucntion裡面有yosys無法辨識的語法(其他商用軟體可以) , 所以作罷 .
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
Transmission Mode  : Mode 0 

                     CPHA = 0  (Data sample at rising edge , Data change at falling edge)

                     CPOL = 0  (SCLK keep low when SPI in IDEAL state)

Transmission Order : LSB first

Main clk           : 50M Hz

## \[Interface\]
SPI Master

| Singnal Name       | Direction | Width  | Note                                                         |
|--------------------|-----------|--------|--------------------------------------------------------------|
| clk                | input     | 1 bit  | NA                                                           |
| reset_n            | input     | 1 bit  | NA                                                           |
| spi_m_start_i      | input     | 1 bit  | NA                                                           |
| spi_m_mosi_d_i     | input     | 8 bit  | NA                                                           |
| spi_m_miso_i       | input     | 1 bit  | NA                                                           |
| spi_m_miso_valid_o | output    | 1 bit  | NA                                                           |
| spi_m_miso_d_o     | output    | 8 bit  | NA                                                           |
| spi_m_sclk_o       | output    | 1 bit  | NA                                                           |
| spi_m_ss_n_o       | output    | 1 bit  | NA                                                           |
| spi_m_mosi_o       | output    | 1 bit  | NA                                                           |
| spi_m_busy_o       | output    | 1 bit  | NA                                                           |

SPI Slave

| Singnal Name        | Direction | Width  | Note                                                         |
|---------------------|-----------|--------|--------------------------------------------------------------|
| s_clk               | input     | 1 bit  | NA                                                           |
| s_rst_n             | input     | 1 bit  | NA                                                           |
| spi_s_sclk_i        | input     | 1 bit  | NA                                                           |
| spi_s_ss_n_i        | input     | 1 bit  | NA                                                           |
| spi_s_mosi_i        | input     | 1 bit  | NA                                                           |
| spi_s_reply_valid_i | input     | 1 bit  | NA                                                           |
| spi_s_reply_d_i     | input     | 8 bit  | NA                                                           |
| spi_s_miso_o        | output    | 1 bit  | NA                                                           |
| spi_s_received_d_o  | output    | 8 bit  | NA                                                           |


## \[Action\]
Set spi_s_reply_d_i and spi_m_mosi_d_i --> Set spi_m_start_i (1 clk) --> Wait spi_m_busy_o low --> check spi_m_miso_i and spi_s_received_d --> return step one

## \[Architecture_Blocks\]
![spi_block](/images/spi/spi_blocks.png)


## \[Pattern_List\]

| Define Name | Pattern Name       | Master Data                       | Slave Data                       | Note                                                     |
|-------------|--------------------|-----------------------------------|----------------------------------|----------------------------------------------------------|
| PATTERN01   | pattern01.v        | 8'hAA -> 8'h55 -> 8'h00 -> 8'hFF  | 8'h55 -> 8'hAA -> 8'hFF -> 8'h00 |                                                          |
| PATTERN02   | pattern02.v        | 8'h77                             | 8'h66                            | rst_n active when SPI is working at first time .         |
| PATTERN03   | pattern03.v        | 8'hAA -> 8'h55                    | 8'hAA -> 8'h55                   | monitor will check data change time point at this pattern|
| NA          | default_pattern.v  | 8'hC3                             | 8'h97                            |                                                          |

## \[Environment_Architecture\]
![spi_tree](/images/spi/spi_tree.png)

## \[RTL code\]
Testbench :
```verilog
`timescale 1ns/10ps
`define SPI_M_TOP    spi_tb.spi_m
`define SPI_S_TOP    spi_tb.spi_s
`define TB           spi_tb

module spi_tb;

// Monitor
`include "../monitor/spi_monitor.v"

//Clock Parameter
parameter CLK_50M_HALF_PIR = 10 ;

//Dump
initial begin
 $dumpfile("dump.vcd");
 $dumpvars(0, spi_tb);
end

//Output
 //Slave
 wire [ 7: 0] received_data_o;
 // Master
 wire [ 7: 0] miso_data_o;
 wire         miso_valid_o;
 wire         busy_o;
 
//Input
 // Slave
 reg  [ 7: 0] reply_data_i;
 reg          reply_data_valid_i;
 // Master
 reg          start_i;
 reg  [ 7: 0] mosi_data_i;

//Connect
 wire          clk;
 wire          sclk;
 wire          ss_n;
 wire          mosi;
 wire          miso;

//Clock
 reg           clk_50M;

initial clk_50M = 1'b0;
always  #CLK_50M_HALF_PIR clk_50M = ~clk_50M;

//Reset
 reg           reset_n;

//Monitor
 reg           mon_st_f;

initial begin
  reset_n  = 1'b1;
  mon_st_f = 1'b0;
  #10;
  reset_n  = 1'b0;
  #CLK_50M_HALF_PIR;
  #CLK_50M_HALF_PIR;
  reset_n = 1'b1;
  #CLK_50M_HALF_PIR;
  #CLK_50M_HALF_PIR;
  mon_st_f = 1'b1;
end

// Circuit
assign clk = clk_50M;

SPI_Master spi_m (
 .clk           (clk),
 .rst_n         (reset_n),
 .start_i       (start_i),
 .mosi_data_i   (mosi_data_i),
 .miso_i        (miso),
 .miso_valid_o  (miso_valid_o),
 .sclk_o        (sclk),
 .ss_n_o        (ss_n),
 .mosi_o        (mosi),
 .busy_o        (busy_o),
 .miso_data_o   (miso_data_o)
);

SPI_Slave spi_s (
 .s_clk              (clk),
 .s_rst_n            (reset_n),
 .sclk_i             (sclk),
 .ss_n_i             (ss_n),
 .mosi_i             (mosi),
 .reply_data_i       (reply_data_i),
 .reply_data_valid_i (reply_data_valid_i),
 .received_data_o    (received_data_o),
 .miso_o             (miso)
);

// Initial Value
initial begin
 // Slave
 reply_data_i       = 0;
 reply_data_valid_i = 0;
 // Master
 start_i            = 0;
 mosi_data_i        = 0;

end

// Pattern List
`ifdef PATTERN01
  `include "../pattern/pattern01.v"
`elsif PATTERN02
  `include "../pattern/pattern02.v"
`elsif PATTERN03
  `include "../pattern/pattern03.v"
`else
  `include "../pattern/default_pattern.v"
`endif

// Task
task send_data (input [7:0] m_data, s_data);
begin
  wait(~busy_o);
  @(posedge clk);
  reply_data_i       <= s_data;
  mosi_data_i        <= m_data;
  reply_data_valid_i <= 1'b1;
  @(posedge clk);
  reply_data_valid_i <= 1'b0;
  @(posedge clk);
  start_i            <= 1'b1;
  @(posedge clk);
  start_i            <= 1'b0;
  wait(busy_o);
end
endtask

endmodule

```

SPI Master :
```verilog
module SPI_Master (
input          clk,
input          rst_n,
input          start_i,
input [ 7: 0]  mosi_data_i,
input          miso_i,
output	       miso_valid_o,
output         sclk_o,
output	       ss_n_o,
output         mosi_o,
output         busy_o,
output [ 7: 0] miso_data_o
);
  
  parameter IDEAL = 0, ST = 1, TRANS_DATA = 2;
  parameter COUNT_NUM = 6; 
  
  wire         ss_n_next;
  wire         busy_f_next;
  wire         miso_shift_en, mosi_shift_en;
  wire         miso_valid_f_next;
  wire         clk_gate_en;
  wire         count_finish, count_en;
  wire [ 2: 0] count_next;
  wire [ 7: 0] mosi_d_shift_next;
  wire [ 7: 0] miso_d_shift_next;
  
  reg  [ 2: 0] count;
  reg  [ 1: 0] state;
  reg  [ 1: 0] state_next;
  reg          ss_n;
  reg          busy_f;
  reg  [ 7: 0] mosi_d_shift;
  reg  [ 7: 0] miso_d_shift;
  reg          miso_valid_f;
  
  assign ss_n_o            = ss_n;
  assign ss_n_next         = (state == IDEAL);
  
  assign busy_o            = busy_f;
  assign busy_f_next       = ((state == ST) | (state == TRANS_DATA));
  
  assign mosi_o            = mosi_d_shift[0];
  assign mosi_shift_en     = (state == TRANS_DATA);
  assign mosi_d_shift_next = (mosi_shift_en)? (mosi_d_shift >> 1) : (mosi_data_i);
  
  assign miso_data_o       = miso_d_shift;
  assign miso_shift_en     = ((state == ST) | (state == TRANS_DATA));
  assign miso_d_shift_next = (miso_shift_en)? ({miso_i,miso_d_shift[7:1]}) : (miso_d_shift);
  
  assign miso_valid_o      = miso_valid_f;
  assign miso_valid_f_next = ((state == TRANS_DATA) & (state_next == IDEAL));
  
  assign count_en          = (state == TRANS_DATA);
  assign count_finish      = (count == COUNT_NUM);
  assign count_next        = (count_finish )? (3'd0) :
  	                     (count_en     )? (count + 1'd1) :
  		                              (count) ;
   
  assign clk_gate_en       = ((state == ST) | (state == TRANS_DATA));			    
  //ICG_CELL ICG (.clk_i(clk), .en_i(clk_gate_en), .clk_o(sclk_o));
  ICG_CELL ICG (.clk_i(clk), .rst_n_i(rst_n), .en_i(clk_gate_en), .clk_o(sclk_o));
  
  always@(posedge clk, negedge rst_n) begin
   if(~rst_n)
    state <= IDEAL;
   else
    state <= state_next;
  end
  
  always@(*) begin
   case(state)
    IDEAL      : state_next = (start_i)? (ST) : (IDEAL);
    ST         : state_next = TRANS_DATA;
    TRANS_DATA : state_next = (count_finish)? (IDEAL) : (TRANS_DATA);
    default    : state_next = IDEAL;
   endcase
  end
  
  always@(posedge clk, negedge rst_n) begin
   if(~rst_n) begin
    count        <= 0;
    busy_f       <= 0;
    miso_d_shift <= 0;
    miso_valid_f <= 0;
   end
   else begin
    count        <= count_next;
    busy_f       <= busy_f_next;
    miso_d_shift <= miso_d_shift_next;
    miso_valid_f <= miso_valid_f_next;
   end
  end
  
  always@(negedge clk, negedge rst_n) begin
   if(~rst_n) begin
    ss_n         <= 0;
    mosi_d_shift <= 0;
   end
   else begin
    ss_n         <= ss_n_next;
    mosi_d_shift <= mosi_d_shift_next;
   end
  end

endmodule

```

SPI Slave :
```verilog
module SPI_Slave (
input          s_clk,
input          s_rst_n,
input          sclk_i,
input          ss_n_i,
input          mosi_i,
input  [ 7: 0] reply_data_i,
input          reply_data_valid_i,
output [ 7: 0] received_data_o,
output         miso_o
);

 wire         valid_data_i_w;
 wire [ 7: 0] received_data_next;
 wire [ 7: 0] reply_data_next;
 reg  [ 7: 0] received_data;
 reg  [ 7: 0] reply_data;

 wire [ 2: 0] reply_count_next;
 reg  [ 2: 0] reply_count;
 reg          miso_d;

 assign miso_o             = (~ss_n_i)? (miso_d) : (1'bz);
 assign reply_data_next    = (reply_data_valid_i)? (reply_data_i) : (reply_data);

 assign valid_data_i_w     = mosi_i | ss_n_i;
 assign received_data_next = ({valid_data_i_w,received_data[7:1]});
 assign received_data_o    = received_data;

 assign reply_count_next   = reply_count + 1'b1;

 always@(*) begin
  case(reply_count)
    3'b000 : miso_d = reply_data[0];
    3'b001 : miso_d = reply_data[1];
    3'b010 : miso_d = reply_data[2];
    3'b011 : miso_d = reply_data[3];
    3'b100 : miso_d = reply_data[4];
    3'b101 : miso_d = reply_data[5];
    3'b110 : miso_d = reply_data[6];
    3'b111 : miso_d = reply_data[7];
  endcase
 end


 always@(posedge sclk_i) begin
   received_data <= received_data_next;
 end

  always@(negedge s_clk, negedge s_rst_n) begin
   if(~s_rst_n)
     reply_data <= 0;
   else
     reply_data <= reply_data_next;
 end

 always@(negedge sclk_i, negedge s_rst_n) begin
   if(~s_rst_n)
     reply_count <= 0;
   else
     reply_count <= reply_count_next;
 end

endmodule

```

Clock Gate Cell :
```verilog
module ICG_CELL (
input  clk_i,
input  rst_n_i,
input  en_i,
output clk_o
);

 reg latch_en;

 assign clk_o = latch_en & clk_i ;

 /* latch
 always@(*) begin
  latch_en = (~clk_i)? (en_i) : (latch_en);
 end
*/

// DFF
always@(negedge clk_i, negedge rst_n_i) begin
 if(~rst_n_i)
   latch_en <=  0;
 else
   latch_en <= en_i;
end

endmodule
```

SPI TOP (synthesis and STA use) :
```verilog
module SPI_TOP (
 input          clk,
 input          reset_n,
 input  [ 7: 0] spi_s_reply_d_i,
 input          spi_s_reply_valid_i,
 input          spi_s_sclk_i,
 input          spi_s_ss_n_i,
 input          spi_s_mosi_i,
 input          spi_m_start_i,
 input  [ 7: 0] spi_m_mosi_d_i,
 input          spi_m_miso_i,
 output [ 7: 0] spi_s_received_d_o,
 output         spi_s_miso_o,
 output [ 7: 0] spi_m_miso_d_o,
 output         spi_m_miso_valid_o,
 output         spi_m_busy_o,
 output         spi_m_sclk_o,
 output         spi_m_ss_n_o,
 output         spi_m_mosi_o,
);



 SPI_Master spi_m (
  .clk           (clk),
  .rst_n         (reset_n),
  .start_i       (spi_m_start_i),
  .mosi_data_i   (spi_m_mosi_d_i),
  .miso_i        (spi_m_miso_i),
  .miso_valid_o  (spi_m_miso_valid_o),
  .sclk_o        (spi_m_sclk_o),
  .ss_n_o        (spi_m_ss_n_o),
  .mosi_o        (spi_m_mosi_o),
  .busy_o        (spi_m_busy_o),
  .miso_data_o   (spi_m_miso_d_o)
 );
 
 SPI_Slave spi_s (
  .s_clk              (clk),
  .s_rst_n            (reset_n),
  .sclk_i             (spi_s_sclk_i),
  .ss_n_i             (spi_s_ss_n_i),
  .mosi_i             (spi_s_mosi_i),
  .reply_data_i       (spi_s_reply_d_i),
  .reply_data_valid_i (spi_s_reply_valid_i),
  .received_data_o    (spi_s_received_d_o),
  .miso_o             (spi_s_miso_o)
 );

endmodule

```

Monitor :
```verilog
module spi_monitor ();

  parameter IDEAL = 0, ST = 1, TRANS_DATA = 2;
  
  reg  [ 7: 0]  spi_m_golden_d;  
  reg  [ 7: 0]  spi_m_receive_d;  
  reg  [ 7: 0]  spi_s_golden_d;  
  reg  [ 7: 0]  spi_s_receive_d;  

  reg           data_check_f;
  event         data_check_fail, ss_n_check_fail, sclk_check_fail;


  //--- DATA Monitor ---//
  initial begin
    data_check_f = 0;
  end
  // catch real data
  always@(posedge `TB.miso_valid_o) begin
    spi_m_receive_d = `TB.miso_data_o;
    spi_s_receive_d = `TB.received_data_o;
    data_check_f    = 1;
  end

  // catch golden data
  always@(posedge `TB.busy_o) begin
    spi_m_golden_d = `TB.mosi_data_i;
    spi_s_golden_d = `TB.reply_data_i;
  end

  // compaire data
  always@(posedge data_check_f) begin

   if(spi_s_receive_d == spi_m_golden_d) begin
      $display("[MON] [PASS] Data Successful !!! SPI Master TX Data = %h , SPI Slave RX Data = %h ! " ,spi_m_golden_d ,spi_s_receive_d );
    end
    else begin
      -> data_check_fail;
      $display("[MON] [FAIL] Data Fail !!! SPI Master TX Data = %h , SPI Slave RX Data = %h ! " ,spi_m_golden_d ,spi_s_receive_d );
      #100;
      $finish;
    end


    if(spi_m_receive_d == spi_s_golden_d) begin
      $display("[MON] [PASS] Data Successful !!! SPI Slave TX Data = %h , SPI Master RX Data = %h ! " ,spi_s_golden_d ,spi_m_receive_d );
    end
    else begin
      -> data_check_fail;
      $display("[MON] [FAIL] Data Fail !!! SPI Slave TX Data = %h , SPI Master RX Data = %h ! " ,spi_s_golden_d ,spi_m_receive_d );
      #100;
      $finish;
    end

    data_check_f = 0;

  end

  //--- ss_n Monitor ---//
  always@(negedge `TB.clk) begin
   #10;
   if(`TB.mon_st_f) begin
    if( ((`SPI_M_TOP.state == IDEAL) & ~`TB.ss_n & `TB.reset_n) | ((`SPI_M_TOP.state != IDEAL) & `TB.ss_n & `TB.reset_n)) begin
      -> ss_n_check_fail;
      $display("[MON] [FAIL] ss_n Fail !!! Now SPI Master State is IDEAL , ss_n should keep high . But now ss_n is %h ! " , `TB.ss_n);
      #100;
      $finish;
    end
   end
   end

  //--- sclk Monitor ---//
  always@(posedge `TB.clk) begin
    #10;
    if((`SPI_M_TOP.state == IDEAL) & `TB.sclk & `TB.mon_st_f & ~`TB.busy_o & `TB.reset_n) begin
      -> sclk_check_fail;
      $display("[MON] [FAIL] sclk Fail !!! Now SPI Master State is IDEAL , sclk should keep low . But now sclk is activeing ! ");
      #100;
      $finish;
    end
   end

  //--- Data change Monitor ---//
  `ifdef PATTERN03
    reg   m_d_bef, m_d_aft;
    reg   s_d_bef, s_d_aft;
    event m_data_change_fail, s_data_change_fail;
 
    always@(posedge `TB.sclk) begin
     m_d_bef = `TB.mosi;
     s_d_bef = `TB.miso;
    end

    always@(negedge `TB.sclk) begin
     #10
     m_d_aft = `TB.mosi;
     s_d_aft = `TB.miso;

     if(`SPI_M_TOP.state != IDEAL) begin
       if(m_d_bef == m_d_aft) begin
         -> m_data_change_fail;
         $display("[MON] [FAIL] Master data change time point Fail !!! ");
         #100;
         $finish;
       end
        
       if(s_d_bef == s_d_aft) begin
         -> s_data_change_fail;
         $display("[MON] [FAIL] Slave data change time point Fail !!! ");
         #100;
         $finish;
       end
     end
     
    end

  `endif
   
endmodule

```

## \[Simulation Waveform\]
Name      : PATTERN01 
Master Test Data : 8'hAA -> 8'h55 -> 8'h00 -> 8'hFF
Slave  Test Data : 8'h55 -> 8'hAA -> 8'hFF -> 8'h00
![PAT01_waveform](/images/spi/PAT01.png)

Name      : PATTERN02
Master Test Data : 8'h77
Slave  Test Data : 8'h66
![PAT02_waveform](/images/spi/PAT02.png)

## \[Yosys synthesis report\]
![yosys_syn_report](/images/spi/yosys_report.png)

## \[OpenSTA timming report\]
![OpenSTA_timming_report_01](/images/spi/timming_report_01.png)
![OpenSTA_timming_report_02](/images/spi/timming_report_02.png)
![OpenSTA_timming_report_03](/images/spi/timming_report_03.png)
![OpenSTA_timming_report_04](/images/spi/timming_report_04.png)
![OpenSTA_timming_report_05](/images/spi/timming_report_05.png)
![OpenSTA_timming_report_06](/images/spi/timming_report_06.png)
