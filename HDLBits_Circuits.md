---
layout: default
title: HDLBits_Circuits Part
toc: true
---

# 此部份都為基礎電路練習，其中分為組合邏輯、循序邏輯、狀態機三大主題 . 
# 組合邏輯部份除了基礎電路能由題目就可以明顯知道題意之外，會需要真質表及卡諾圖化簡相關知識 .
# 循序邏輯部份講解了Latche和DFF差別與一些簡單的循序電路(ex:計數器、移位站存器)之外，也有介紹DFF中clock和reset正緣及負緣觸發寫法 .
# 狀態機部份介紹了莫爾機、米粒機差別與one-hot logic equation寫法 .
# 在此只紀錄Building Larger Circuits解法 .

{:toc}
###
題目 : [Exams/review2015 count1k](https://hdlbits.01xz.net/wiki/Exams/review2015_count1k)

練習目標 : 設計出由0計數至999的計數器，reset觸發與clock同步且會將此計數器歸零 .

解法 :
<details>
<summary>點擊展開程式碼</summary>
```verilog
module top_module (
    input clk,
    input reset,
    output [9:0] q);
    
    reg [9:0] cnt;
    wire [9:0] cnt_next;
    
    assign q = cnt;
    assign cnt_next = (cnt == 10'd999)? (10'd0) : (cnt + 10'd1);
    
    always@(posedge clk)begin
        if(reset)
            cnt <= 0;
        else
            cnt <= cnt_next;
    end

endmodule
</details>

###
題目 : [Exams/review2015 shiftcount](https://hdlbits.01xz.net/wiki/Exams/review2015_shiftcount)

練習目標 : 當shift_ena為high時計數器數值由data移位進入設定，當count_ena為high時即開始下數至0 . 此題不考慮兩個enable訊號同時發生期況 .

解法 :
<details>
<summary>點擊展開程式碼</summary>
```verilog
module top_module (
    input clk,
    input shift_ena,
    input count_ena,
    input data,
    output [3:0] q);
    
    reg [3:0] cnt;
    wire [3:0] cnt_next;
    
    assign q = cnt;
    assign cnt_next = (shift_ena & ~count_ena)? ({cnt[2:0],data}) : 
        (~shift_ena & count_ena)? (cnt - 4'd1) :
        (cnt);
    
    always@(posedge clk) begin
        cnt <= cnt_next;
    end
    

endmodule
</details>

###
題目 : [Exams/review2015 fsmseq](https://hdlbits.01xz.net/wiki/Exams/review2015_fsmseq)

練習目標 : 設計狀態機偵測"1101"資料輸入序列是否發生 . 

解法 :
<details>
<summary>點擊展開程式碼</summary>
```verilog
module top_module (
    input clk,
    input reset,      // Synchronous reset
    input data,
    output start_shifting);
    
    parameter IDLE=0, A=1, B=2, C=3, FIN=4;
    
    reg [2:0] state, next_state;
    
    assign start_shifting = (state == FIN);
    
    always@(posedge clk) begin
        if(reset)
            state <= IDLE;
        else
            state <= next_state;
    end
    
    always@(*) begin
        case(state)
            IDLE:begin
                next_state = (data)? (A) : (IDLE);
            end
            A:begin
                next_state = (data)? (B) : (IDLE);
            end
            B:begin
                next_state = (data)? (B) : (C);
            end
            C:begin
                next_state = (data)? (FIN) : (IDLE);
            end
            FIN:begin
                next_state = (FIN);
            end
            default:begin
                next_state = IDLE;
            end
        endcase
    end

endmodule
</details>

###
題目 : [Exams/review2015 fsmshift](https://hdlbits.01xz.net/wiki/Exams/review2015_fsmshift)

練習目標 : 按照題目波型圖設計狀態機 . 由波型可知會需要一個計數器計算shift_ena為high的時間，或是直接用狀態堆也可以 .

解法 :
<details>
<summary>點擊展開程式碼</summary>
```verilog
module top_module (
    input clk,
    input reset,      // Synchronous reset
    output shift_ena);

    parameter IDLE=0, A=1, CNT_DOWN=2, FIN=3;
    
    reg [1:0] state, next_state;
    reg [1:0] cnt;
    wire [1:0] cnt_next;
    reg out_r;
    wire out_r_next;
    
    assign cnt_next = (state == CNT_DOWN)? (cnt + 1'd1) : (cnt);
    assign shift_ena = (state == IDLE) | (state == CNT_DOWN);
    
    always@(posedge clk) begin
        if(reset)
            state <= IDLE;
        else
        	state <= next_state;
    end
    
    always@(*) begin
        case(state)
            IDLE:begin
                next_state = CNT_DOWN;
            end
            CNT_DOWN:begin
                next_state = (cnt == 2'd2)? (FIN) : (CNT_DOWN);
            end
            FIN:begin
                next_state = (reset)? (IDLE) : (FIN);
            end
        endcase
    end
    
    always@(posedge clk) begin
        if(reset)
            cnt <= 0;
        else 
            cnt <= cnt_next;
    end
    
    
endmodule
</details>

###
題目 : [Exams/review2015 fsm](https://hdlbits.01xz.net/wiki/Exams/review2015_fsm)

練習目標 : 此題是前面4題的融合體 . 在偵測輸入資料串為"1101"後，移位輸入計數器值(shift_ena為high) . 之後計數器開始動作(期間counting為high)直到收到done_counting，done為high直到收到ack回到初始狀態 .

解法 :
<details>
<summary>點擊展開程式碼</summary>
```verilog
module top_module (
    input clk,
    input reset,      // Synchronous reset
    input data,
    output shift_ena,
    output counting,
    input done_counting,
    output done,
    input ack );

    parameter A=0, B=1, C=2, D=3, SH=4, CNT=5, DONE=6;
    
    reg [3:0] state, next_state;
    wire [1:0] cnt_next;
    reg [1:0] cnt;
    
    assign cnt_next  = (state == SH)? (cnt + 1'b1) : (2'd0);
    assign shift_ena = (state == SH);
    assign counting  = (state == CNT);
    assign done      = (state == DONE);
    
    always@(posedge clk) begin
        if(reset)
            state <= A;
        else
            state <= next_state;
    end
    
    always@(*) begin
        case(state)
            A:begin
                next_state = (data)? (B) : (A);
            end
            B:begin
                next_state = (data)? (C) : (A);
            end
            C:begin
                next_state = (~data)? (D) : (C);
            end
            D:begin
                next_state = (data)? (SH) : (A);
            end
            SH:begin
                next_state = (cnt == 3)? (CNT) : (SH);
            end
            CNT:begin
                next_state = (done_counting)? (DONE) : (CNT);
            end
            DONE:begin
                next_state = (ack)? (A) : (DONE);
            end
        endcase
    end
    
    always@(posedge clk) begin
        if(reset)
            cnt <= 0;
        else
            cnt <= cnt_next;
    end
    
    
endmodule
</details>

###
題目 : [Exams/review2015 fancytimer](https://hdlbits.01xz.net/wiki/Exams/review2015_fancytimer)

練習目標 : 前一題的進階，差在計數器並非收到input訊號才停止而是下數至0才結束，且下數動作則是需要由0數至999才會觸發 .

解法 :
<details>
<summary>點擊展開程式碼</summary>
```verilog
module top_module (
    input clk,
    input reset,      // Synchronous reset
    input data,
    output [3:0] count,
    output counting,
    output done,
    input ack );
    
    parameter A=0, B=1, C=2, D=3, SH1=4, SH2=5, SH3=6, SH4=7, LOAD_CNT=8, CNT=9, DONE=10;
    parameter DELAY_TIME=999;
    
    reg [3:0] state, next_state;
    
    reg  [15:0] cnt, cnt2;
    wire [15:0] cnt_next, cnt2_next;
    reg  [3:0]  sh_d;
    wire [3:0]  sh_d_next;
    
    assign sh_d_next  = ((state == SH1) | (state == SH2) | (state == SH3) | (state == SH4))? ({sh_d[2:0],data}) : (sh_d);
    assign count      = (state == LOAD_CNT)? (sh_d) :
        				(state == CNT)?      (cnt2) : (4'd0);
    assign counting   = ((state == CNT) | (state == LOAD_CNT));
    assign done       = (state == DONE);
    assign cnt_next   = (cnt == DELAY_TIME)?   (16'd0) :
                        (state == LOAD_CNT)?   (cnt + 16'd1) :
        				(state == CNT) ?       (cnt + 16'd1) : (16'd0);
    
    assign cnt2_next  = (state == LOAD_CNT)? 					(sh_d) :
        				((state == CNT) & (cnt == DELAY_TIME))? (cnt2 - 16'd1) : (cnt2);
    
    always@(posedge clk) begin
        if(reset)
            state <= A;
        else
            state <= next_state;
    end
    
    always@(*) begin
        case(state)
            A:begin
                next_state = (data)? (B) : (A);
            end
            B:begin
                next_state = (data)? (C) : (A);
            end
            C:begin
                next_state = (~data)? (D) : (C);
            end
            D:begin
                next_state = (data)? (SH1) : (A);
            end
            SH1:begin
                next_state = SH2;
            end
            SH2:begin
                next_state = SH3;
            end
            SH3:begin
                next_state = SH4;
            end
            SH4:begin
                next_state = LOAD_CNT;
            end
            LOAD_CNT:begin
                next_state = CNT;
            end
            CNT:begin
                next_state = ((cnt2 == 16'd0) & (cnt == DELAY_TIME))? (DONE) : (CNT);
            end
            DONE:begin
                next_state = (ack)? (A) : (DONE);
            end
        endcase
    end
    
    always@(posedge clk) begin
        if(reset) begin
            cnt  <= 0;
            cnt2 <= 0;
        	sh_d <= 0;
        end
        else begin
            cnt  <= cnt_next;
            cnt2 <= cnt2_next;
            sh_d <= sh_d_next;
        end
    end
    
endmodule
</details>

###
題目 : [Exams/review2015 fsmonehot](https://hdlbits.01xz.net/wiki/Exams/review2015_fsmonehot)

練習目標 : 根據題目提供的狀態圖設計狀態機，但是寫法必須是One-hot logic equation

解法 :
<details>
<summary>點擊展開程式碼</summary>
```verilog
module top_module(
    input d,
    input done_counting,
    input ack,
    input [9:0] state,    // 10-bit one-hot current state
    output B3_next,
    output S_next,
    output S1_next,
    output Count_next,
    output Wait_next,
    output done,
    output counting,
    output shift_ena
); //

    // You may use these parameters to access state bits using e.g., state[B2] instead of state[6].
    parameter S=0, S1=1, S11=2, S110=3, B0=4, B1=5, B2=6, B3=7, Count=8, Wait=9;

    assign B3_next = state[6];
    assign S_next  = (state[0] & ~d) | (state[1] & ~d) | (state[3] & ~d) | (state[9] & ack);
    assign S1_next = state[0] & d;
    assign Count_next = (state[7]) | (state[8] & ~done_counting);
    assign Wait_next = (state[9] & ~ack) | (state[8] & done_counting);
    
    assign done = state[9];
    assign counting = state[8];
    assign shift_ena = state[7] | state[6] | state[5] | state[4];
    
endmodule

</details>