---
layout: default
title: HDLBits Verilog_Language Part
toc: true
---

# 此部份都為基礎語法，照著題目範例都能了解題意並寫出答案 !
# 在此只紀錄練習題解法 .

{:toc}
###
題目 : [Vector100r](https://hdlbits.01xz.net/wiki/Vector100r)

練習目標 : 將input反轉並直接接到output . ex: output[99] = input[0] ..... output[0] = input[99]

解法 :
```verilog
module top_module( 
    input [99:0] in,
    output [99:0] out
);
    
    genvar i;
    
    generate
        for (i=0;i<=99;i=i+1) begin : a1
            assign out[i] = in[99-i];
        end
    endgenerate

endmodule
```

###
題目 : [Adder100i](https://hdlbits.01xz.net/wiki/Adder100i)

練習目標 : 使用100個全加器做出100bit漣波加法器

解法 :
```verilog
module top_module( 
    input [99:0] a, b,
    input cin,
    output [99:0] cout,
    output [99:0] sum );
    
    addr addr0 (.a(a[0]) , .b(b[0]), .cin(cin), .cout(cout[0]), .sum(sum[0]));
    
    genvar i;
    generate
        for(i=1;i<=99;i=i+1) begin : test
            addr addr1 (.a(a[i]) , .b(b[i]), .cin(cout[i-1]), .cout(cout[i]), .sum(sum[i]));
        end
    endgenerate
endmodule

module addr(
    input a,
    input b,
    input cin,
    output cout,
    output sum);

    assign {cout,sum} = a + b + cin;
    
endmodule
```

###
題目 : [Bcdadd100](https://hdlbits.01xz.net/wiki/Bcdadd100)

練習目標 : 由題目提供的bcd_fadd組合出400bit BCD漣波加法器

解法 :
```verilog
module top_module( 
    input [399:0] a, b,
    input cin,
    output cout,
    output [399:0] sum );

    wire [399:0] cout_to_next;
    
    bcd_fadd fadd_ini  (.a(a[3:0]), .b(b[3:0]), .cin(cin), .cout(cout_to_next[0]), .sum(sum[3:0]) );
    bcd_fadd fadd_last (.a(a[399:396]), .b(b[399:396]), .cin(cout_to_next[392]), .cout(cout), .sum(sum[399:396]) );
    genvar i,y;
    generate 
        for(i=4 ; i<=392 ; i=i+4 ) begin : test
            bcd_fadd fadd (.a(a[i+3:i]), .b(b[i+3:i]), .cin(cout_to_next[i-4]), .cout(cout_to_next[i]), .sum(sum[i+3:i]) );
        end
    endgenerate

endmodule
```