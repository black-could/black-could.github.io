---
layout: default
title: HDLBits CS450 part
toc: true
---

# 都為計算機架構電路設計 .
# 最後一題還須找資料研究 .

{:toc}
###
題目 : [Cs450/timer](https://hdlbits.01xz.net/wiki/Cs450/timer)

練習目標 : 當load為high時輸入計數器數值，當load為low時開始下數至0 . 計數器為0後tc為high .

解法 :
<details>
<summary>點擊展開程式碼</summary>
```verilog
module top_module(
	input clk, 
	input load, 
	input [9:0] data, 
	output tc
);
    
    reg [9:0] cnt;
    wire [9:0] cnt_next;
    
    assign cnt_next = (load)? (data) : 
        (cnt != 10'd0)? (cnt - 10'd1) : (10'd0);
    
    assign tc = (cnt == 10'd0);
    
    always@(posedge clk) begin
        cnt <= cnt_next;
    end
    
endmodule
</details>

###
題目 : [Cs450/counter 2bc](https://hdlbits.01xz.net/wiki/Cs450/counter_2bc)

練習目標 : 此題狀態圖為分支方向預測器動作，即是動態預測中的2-bit Saturating Counter .

解法 :
<details>
<summary>點擊展開程式碼</summary>
```verilog
module top_module(
    input clk,
    input areset,
    input train_valid,
    input train_taken,
    output [1:0] state
);
    
    parameter SNT=0, WNT=1, WT=2, ST=3;
    
    reg [1:0] state_r, next_state_r;

    assign state = state_r;
    
    always@(posedge clk, posedge areset) begin
        if(areset)
            state_r <= WNT;
        else
            state_r <= next_state_r;
    end
    
    always@(*) begin
        case(state_r)
            SNT:begin
                next_state_r = (~train_valid)? (SNT) :
                (train_taken)? (WNT) : (SNT);
            end
            WNT:begin
                next_state_r = (~train_valid)? (WNT) :
                (train_taken)? (WT) : (SNT);
            end
            WT:begin
                next_state_r = (~train_valid)? (WT) :
                (train_taken)? (ST) : (WNT);
            end
            ST:begin
                next_state_r = (~train_valid)? (ST) :
                (train_taken)? (ST) : (WT);
            end
        endcase
    end

endmodule
</details>

###
題目 : [Cs450/history shift](https://hdlbits.01xz.net/wiki/Cs450/history_shift)

練習目標 : 此為歷史移位暫存器用來幫助增強分支預測精準度 . 會將最近N次分支結果位移至此暫存器，若預測錯誤會載入更新值 .

解法 :
<details>
<summary>點擊展開程式碼</summary>
```verilog
module top_module(
    input clk,
    input areset,

    input predict_valid,
    input predict_taken,
    output [31:0] predict_history,

    input train_mispredicted,
    input train_taken,
    input [31:0] train_history
);
    
    reg  [31:0] sf_r;
    wire [31:0] sf_r_next;
    
    assign predict_history = sf_r;
    
    assign sf_r_next = (train_mispredicted)? ({train_history[30:0],train_taken}) : 
        (predict_valid)? ({sf_r[30:0],predict_taken}) : 
        (sf_r);
    
    
    always@(posedge clk, posedge areset) begin
        if(areset)
            sf_r <= 0;
        else
            sf_r <= sf_r_next;
    end
    
endmodule
</details>

###
題目 : [Cs450/history shift](https://hdlbits.01xz.net/wiki/Cs450/gshare)

練習目標 : 這題光看題目其實不知道它在說什麼 , 此解法為參考網路上大神的code . 
          自行研究後發現是由上面兩題組合起來的完整功能 . 
          分支預測器的動作則是根據PC堆疊和train_hisotry做互斥或產生強預測Taken、弱預測Taken、強預測Not Taken、弱預測Not Taken結果 .
          歷史移位暫存器在預測成功後會將此次分支預測器結果輸入暫存，若預測錯誤則會載入train_history數值 .
          
解法 :
<details>
<summary>點擊展開程式碼</summary>
```verilog
module top_module(
    input clk,
    input areset,

    input  predict_valid,
    input  [6:0] predict_pc,
    output predict_taken,
    output [6:0] predict_history,

    input train_valid,
    input train_taken,
    input train_mispredicted,
    input [6:0] train_history,
    input [6:0] train_pc
);
    
    reg [1:0] PHT[127:0];
    integer i;
    always @(posedge clk, posedge areset) begin
        if (areset) begin
            predict_history <= 0;
            for (i=0; i<128; i=i+1) PHT[i] <= 2'b01;
        end
        else begin
            if (train_valid && train_mispredicted)
                   predict_history <= {train_history[6:0], train_taken};
            else if (predict_valid)
                predict_history <= {predict_history[6:0], predict_taken};
            
            if (train_valid) begin
                if (train_taken)
                    PHT[train_history ^ train_pc] <= (PHT[train_history ^ train_pc] == 2'b11) ? 2'b11 : (PHT[train_history ^ train_pc] + 1);
            else
                    PHT[train_history ^ train_pc] <= (PHT[train_history ^ train_pc] == 2'b00) ? 2'b00 : (PHT[train_history ^ train_pc] - 1);
            end
        end
    end
    assign predict_taken = PHT[predict_history ^ predict_pc][1];

endmodule
</details>

