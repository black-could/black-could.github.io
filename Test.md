---
layout: default
title: Test
toc: true
---

# 使用 NanoSim 進行功耗模擬

{:toc}

在專案中我們使用 NanoSim 搭配 Perl script 對 ADC、PLL、Clock switch 進行行為模擬，以下是主要步驟與經驗整理：

## 建立功耗模型
模擬過程中使用 .vcd 匯出模擬結果並觀察 switching 行為。

## 撰寫模擬條件 script
利用 Perl script 控制不同切換狀態與模組 enable 條件。

## 分析 waveform 結果
結合 Verdi 與 NanoSim waveform，比對實測與模擬電流變化趨勢。

> 工具：NanoSim, Verdi, Perl
