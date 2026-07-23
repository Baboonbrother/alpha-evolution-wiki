---
title: 實驗 F01：完整 king2 重疊圖——法人家族第一輪，不追 Sharpe 只清換皮
group: 實驗
exp: F01
---

# 實驗 F01：完整 king2 重疊圖——法人家族第一輪

**這一頁同時記錄主線改向。** owner 裁決：研究資源過度流向「通用 Research OS」（Polymarket n=88 INCONCLUSIVE、未直接改善 king2），真正的優勢在「台股既有資料 × 人類王牌策略」——**王牌不只是冠軍基準，是整個探索空間的母體**。即日起暫停：Polymarket 擴張、本地模型 operator 擴張、通用世界事件、新 UI/載具架構。接下來 3-5 輪全部集中：**FinLab Feature Graph × frozen king2 residuals**。

owner 同時做了一個關鍵核對：**king2 的 35% tilt 本來就含集保**（小股東 <5張/<10張 下降＋大戶 >1000張 上升、週頻 ffill）——「把集保加進 king2」不是新 Alpha。正確方向＝變化型特徵＋去換皮＋殘差條件交互。

## 第 1 條：DataContract（PIT 從契約開始）

13 個 FinLab 資料集實測入帳（頻率/覆蓋/缺值由 feather 實算，非手填）：

| 資料集 | 頻率 | 起始 | 可用時點（保守） |
|---|---|---|---|
| 法人三大（外資/投信/自營） | 日 | 2012-05 | T+1 開盤前（盤後公布→shift(1)） |
| 融資券餘額 | 日 | 2009-01 | T+1 開盤前 |
| **集保三級距**（>1000張/<10張/<5張） | 週 | **2016-11** | **統計日＋約3交易日** |
| 月營收 YoY | 月 | 2005-02 | index=申報截止日，as-of 即 PIT |
| 價量/市值/基本面 | 日/季 | 2007+/2013+ | EOD／財報揭露日 |

兩個契約級誠實發現：①集保 2016-11 起——**這正是 king2「2017 前籌碼中性 0.5」設計的資料根源**；②owner 警示落實：**集保統計日≠可用日**（TDCC 統計至週五、隔週初公布）——king2 真錢線自己的 `dly()` 從統計日 ffill，此 nuance 誠實記錄於契約（真錢線唯讀不動），後續研究對齊一律加 lag。

## 第 2 條：完整 king2 成為可計算節點（不再用 rev_rank 代理）

照真錢源碼逐字重建（研究鏡像；真錢線唯讀零寫入）：

```
score = 0.65·RK(月營收YoY) + 0.35·tilt,   tilt = (fp + chip)/2
fp   = (低券資比 + 低波動60 + 貼120日高 + 剛轉強5日 + 短期加速)/5     ← 五指紋
chip = (散戶10張26週流出 + 8週流出 + 大戶千張26週進貨 + 散戶5張水位低)/4  ← 四集保分量
```

**對帳驗證**：對凍結殘差 parquet 抽樣 2,500 格逐格比對（rev_rank/fp/chip/tilt/score 五分量）——**0 格超容差**。容差＝排名位次級（5e-4≈1.4/2764）：parquet 建於 07-22、finlab_db 每日鏡像更新，之後的資料修訂微移橫斷面排名分母（實測 max 2.6e-4、p99 9.2e-5）＝**公式逐字一致、差異純屬快照日更漂移**（誠實記錄非掩蓋）。從此「對 king2 正交/重疊」以完整 score 與各分量為準——先前開放探索只控 rev_rank（65% 主軸）的缺陷已補。

## 第 3-4 條：法人家族十一變化型特徵 × 重疊圖

第一輪只跑法人（owner 排程：法人→集保變化→法人×集保，禁全排列）。11 個變化型特徵，全部 PIT shift(1)、與 king2 同錨日同宇宙：`inst_turn_5/20/60`（淨買金額÷成交金額）、`foreign/trust_turn_20`、`inst_mcap_20`、`inst_z`（歷史 z）、`trust_streak`（連買天數）、`inst_accel`（加速度）、`consensus_20`（三法人共識）、`diverge_mom20`（價漲法人不買背離）。

**EXP-F01 設計即紀律**：**零前瞻報酬**——只量特徵幾何（特徵 vs 完整 king2 各分量＋size＋liquidity 的 100 月頻錨日橫斷面 Spearman 平均），不碰 fwd/excess/Sharpe，金庫無涉（考卷以使用模式驗證）。分類門檻事前定：max|corr| vs 任一分量 ≥0.5→`REDUNDANT_SKIN`；0.3–0.5→`PARTIAL_OVERLAP`；<0.3→`CANDIDATE_INCREMENTAL`；|corr vs size|≥0.5 附 `+SIZE_PROXY`。

## 結果：法人流不是 king2 籌碼分數的換皮

| 特徵 | 對 score | 對 chip | 對 size | 分類 |
|---|---|---|---|---|
| inst_turn_60 | 0.167 | **0.324** | 0.015 | **PARTIAL_OVERLAP** |
| inst_mcap_20 | 0.107 | 0.298 | −0.001 | CANDIDATE_INCREMENTAL |
| inst_turn_20 | 0.102 | 0.227 | 0.000 | CANDIDATE_INCREMENTAL |
| foreign_turn_20 | 0.095 | 0.196 | 0.001 | CANDIDATE_INCREMENTAL |
| inst_z | 0.036 | 0.162 | −0.010 | CANDIDATE_INCREMENTAL |
| （其餘 6 個） | <0.06 | <0.13 | ≈0 | CANDIDATE_INCREMENTAL |

三個乾淨的讀數：

1. **10/11 是真候選增量、1 個部分重疊、零換皮**——日頻法人流與週頻集保級距在概念上相近，但測量上是不同的軸（max 0.324）。
2. **重疊的方向完全符合經濟直覺**：11 個特徵裡 7 個的最大重疊分量是 **chip**（法人買 ↔ 大戶累積），且窗越長重疊越高（turn_60 > turn_20 > turn_5）——與 chip 的 26 週累積視角一致。**圖現在知道「這個特徵測量什麼、與 king2 哪個分量最近」**。
3. **零 SIZE_PROXY**（size 相關全 ≈0）——標準化（÷成交金額）成功移除規模效應。

**定義圖落地**：11 條邊寫入 `qual_edge`（`feature:X --redundant_to/overlaps_with/candidate_incremental_to_king2--> king2:分量`，evidence=實測相關值）——圖從「哪個實驗做過」開始走向「這個特徵與 king2 哪裡重複」。

## 誠實邊界（不得省略）

- **F01 沒有任何 Alpha 宣稱**：CANDIDATE_INCREMENTAL 只代表「與 king2 現有分量低相關」，是否真能解釋殘差＝EXP-F02（殘差辨識，下一輪），是否改善 king2＝EXP-F03（五臂挑戰，再下一輪）。
- **相關的 t 是 100 個月頻錨日的樸素 t**（近不重疊，仍非顯著性宣稱）。
- **市值單位保守處理**：`inst_mcap_20` 的市值單位未逐一核對（僅用於橫斷面排名，量級不影響 rank）。
- **q_quality/B 閘等 king2 過濾器分量未入重疊圖**（本輪只對五個排序分量）；殘差集內 2017 前 chip=中性 0.5 段的相關被稀釋——分段重跑列入 F02 設計。

延伸：king2 精確規則與殘差四格見 [[champion-challenger|現任冠軍制度]]；主線改向前的通用線成果見 [[exp-009-expectation-layer|EXP-009]]（暫停擴張、成果保留）；資料契約狀態機見 [[blockers|難點 B1]]；下一輪 EXP-F02（殘差辨識）與 EXP-F03（五臂）的設計要求見 [[exp-index|實驗索引]]。
