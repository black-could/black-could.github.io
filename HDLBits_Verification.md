---
layout: default
title: HDLBits Verificaion part
toc: true
---

# 此部分主要介紹簡單的除錯、根據波型設計電路以及testbenches寫法 !
# 這邊只挑幾題做紀錄 .

{:toc}
###
題目 : [Sim/circuit6](https://hdlbits.01xz.net/wiki/Sim/circuit6)

解說 : 由題目波型圖可知能由case語法實現 .

解法 :
```verilog
module top_module (
    input [2:0] a,
    output reg [15:0] q ); 

    always@(*) begin
        case(a)
            3'd0: q=16'h1232;
            3'd1: q=16'haee0;
            3'd2: q=16'h27d4;
            3'd3: q=16'h5a0e;
            3'd4: q=16'h2066;
            3'd5: q=16'h64ce;
            3'd6: q=16'hc526;
            3'd7: q=16'h2f19;
        endcase
    end
    
endmodule
```

###
題目 : [Sim/circuit10](https://hdlbits.01xz.net/wiki/Sim/circuit10)

解說 : 根據波型畫出真值表即可知道輸入和輸出關係 .

解法 :
```verilog
module top_module (
    input clk,
    input a,
    input b,
    output q,
    output state  );

    reg state_r;
    wire state_r_next;
    
    assign q = (state_r)? (~(a^b)) : (a^b);
    assign state = state_r;
    assign state_r_next = (b&~q) | (a&~q) | (a&b);
    
    always@(posedge clk) begin
        state_r <= state_r_next;
    end
    
endmodule
```

###
題目 : [Tb/tff](https://hdlbits.01xz.net/wiki/Tb/tff)

解說 : 根據題目提供的TFF寫Testbenches，要包含reset和toggle至high .

解法 :
```verilog
module top_module ();
    
    reg clk, reset, t;
    wire q;
    
    tff T1 (
        .clk(clk),
        .reset(reset),
        .t(t),
        .q(q));
    
    initial begin
        clk = 0;
        forever begin
            #5;
            clk = ~clk;
        end
    end
    
    initial begin
        reset = 1'b1;
        @(posedge clk);
        @(posedge clk);
        reset = 1'b0;
        @(posedge clk);
        t = 1'b1;
        @(posedge clk);
        @(posedge clk);
        t = 1'b0;
        @(posedge clk);
        @(posedge clk);
        
    end
    
endmodule
```


