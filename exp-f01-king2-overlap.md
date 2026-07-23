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

**EXP-F01 設計即紀律**：**零前瞻報酬**——只量特徵幾何（特徵 vs 完整 king2 各分量＋size＋liquidity 的 100 月頻錨日橫斷面 Spearman 平均），不碰 fwd/excess/Sharpe，金庫無涉（考卷以使用模式驗證）。分類門檻事前定：max|corr| vs 任一分量 ≥0.5→`REDUNDANT_SKIN`；0.3–0.5→`PARTIAL_OVERLAP`；<0.3→`LOW_OBSERVED_OVERLAP`（F01.1 改名,原 CANDIDATE_INCREMENTAL 屬語義過度）；|corr vs size|≥0.5 附 `+SIZE_PROXY`。

## ⚠ F01.1 語義修正（owner 揪出四類問題，全部已修）

初版有與「不顯著≠independent」同類的語義病：**低相關≠增量**。修正（`wm/exp_f01_1.py`＋考卷 11 題含合成對抗測試）：

1. **改名**：`CANDIDATE_INCREMENTAL` → **`LOW_OBSERVED_OVERLAP`**；`candidate_incremental_to_king2` → **`low_monotonic_overlap_with`**。低相關只證明「目前樣本/宇宙/錨/Spearman 口徑下未觀察到高度單調重疊」，不證明增量預測力／可解釋殘差／獨立機制／非線性不換皮。
2. **宣稱降級**：「法人流是 king2 沒用到的軸」→「**法人流是值得進入殘差辨識的低重疊候選軸**」。`INCREMENTAL_RESIDUAL_INFORMATION` 要 F02 過後才配；`INCREMENTAL_TRADABLE_ALPHA` 要 F03 成本後改善 king2 才配。
3. **SCORE ≠ POLICY**：本實驗量的是 `KING2_SCORE_NODE`（0.65rev+0.35tilt）分量重疊——股票池／品質閘／Top-12／月頻錨定／提前退出／regime 覆蓋**全未入圖**，不得宣稱 `FULL_KING2_POLICY_OVERLAP`。
4. **重疊檢查補強**：mean 之外補 median／**mean_abs**／p25/p75／**符號一致率**（抓正負抵銷）／分年／regime 分段；新增**時間分組非線性 OOF 冗餘檢定**（king2 分量 decile 條件均值 train→test R²，抓 U 型/門檻換皮）。合成對抗測試證明機制有效：正負抵銷案例（mean=−0.002、mean_abs=0.86）被 `SIGN_FLIP_OVERLAP` 旗標抓到；U 型案例（Spearman=−0.02、OOF R²=0.93）被 `NONLINEAR_OVERLAP_SUSPECT` 抓到。
5. **定義圖／證據圖分表**：計算語義（uses_data/applies_transform/normalizes_by，60 邊）入 `def_edge`；實驗結果（帶 period/universe/metric/**run_hash/data_snapshot_id** 的 provisional 邊）入 `evi_edge`；F01 誤寫入 `qual_edge` 的 11 邊以 `edge_supersession` 記錄遷移（append-only 舊列保留）。
6. **FeatureSpec 正式化**：11 特徵登記成 B/X/W/R/O＋available_at 宣告式規格＋純碼解譯器——**解譯器重算 vs 手寫實作逐格零差**（規格=實作證明）；系統從此能沿 window/normalization/reference 合法變異鄰近特徵。
7. **可重現性**：資料快照凍結（14 個來源 feather 的 sha256 → `data_snapshot_id`）＋run_hash＋[完整證物包 evidence/F01/](https://github.com/Baboonbrother/alpha-evolution-wiki/tree/main/evidence/F01)（run/source manifest、overlap matrix、graph edges、test 輸出）——GitHub 讀者可指認「哪版程式×哪版資料→哪個結果」（程式與 FinLab 資料本體因授權/真錢鄰接不公開，manifest 提供版本指認）。

## 結果（F01.1 修正口徑）：法人流＝值得進入殘差辨識的低重疊候選軸

| 特徵 | 對 score | 對 chip | 對 size | 分類 |
|---|---|---|---|---|
| inst_turn_60 | 0.167 | **0.324** | 0.015 | **PARTIAL_OVERLAP** |
| inst_mcap_20 | 0.107 | 0.298 | −0.001 | LOW_OBSERVED_OVERLAP |
| inst_turn_20 | 0.102 | 0.227 | 0.000 | LOW_OBSERVED_OVERLAP |
| foreign_turn_20 | 0.095 | 0.196 | 0.001 | LOW_OBSERVED_OVERLAP |
| inst_z | 0.036 | 0.162 | −0.010 | LOW_OBSERVED_OVERLAP |
| （其餘 6 個） | <0.06 | <0.13 | ≈0 | LOW_OBSERVED_OVERLAP |

三個乾淨的讀數：

1. **10/11 為 LOW_OBSERVED_OVERLAP、1 個 PARTIAL_OVERLAP、零換皮**——且 F01.1 補強後：符號一致率 0.87–1.0（無正負抵銷隱藏）、**非線性 OOF R² 最大僅 0.090**（遠低於 0.25 門檻＝低重疊不是 U 型/門檻換皮藏著）。
2. **重疊的方向完全符合經濟直覺**：11 個特徵裡 7 個的最大重疊分量是 **chip**（法人買 ↔ 大戶累積），且窗越長重疊越高（turn_60 > turn_20 > turn_5）——與 chip 的 26 週累積視角一致。**圖現在知道「這個特徵測量什麼、與 king2 哪個分量最近」**。
3. **零 SIZE_PROXY**（size 相關全 ≈0）——標準化（÷成交金額）成功移除規模效應。

**圖落地（F01.1 修正後）**：定義圖 60 邊入 `def_edge`（uses_data/applies_transform/normalizes_by——特徵怎麼算）；證據圖 11 邊入 `evi_edge`（`feature:X --low_monotonic_overlap_with--> king2_score:分量`，帶 metric＋valid_under＋run_hash＋data_snapshot_id 的 provisional 邊）；F01 初版誤寫入 `qual_edge` 的 11 邊已由 `edge_supersession` 記錄遷移。圖從「哪個實驗做過」走向「這個特徵怎麼算、與 king2 score 哪個分量最近、證據在什麼條件下成立」。

## 誠實邊界（不得省略）

- **F01/F01.1 沒有任何 Alpha 宣稱**：LOW_OBSERVED_OVERLAP 只代表「目前口徑下未觀察到高度單調重疊」。**F02 設計約束（owner 凍結）**：殘差標籤的分位門檻只能由每個訓練窗生成（validation 不得參與門檻設定）、以換股月份 grouped split、同持有窗股票不拆 fold；核心裁決＝**base（完整 king2）vs base＋法人特徵的樣本外增量**（Δlog loss／ΔPR-AUC／假陽性 top-risk precision／漏網 top-opportunity precision／年度符號穩定），不得以單特徵顯著性取代 incremental test；先不看 Sharpe。
- **相關的 t 是 100 個月頻錨日的樸素 t**（近不重疊，仍非顯著性宣稱）。
- **市值單位保守處理**：`inst_mcap_20` 的市值單位未逐一核對（僅用於橫斷面排名，量級不影響 rank）。
- **q_quality/B 閘等 king2 過濾器分量未入重疊圖**（本輪只對五個排序分量）；殘差集內 2017 前 chip=中性 0.5 段的相關被稀釋——分段重跑列入 F02 設計。

延伸：king2 精確規則與殘差四格見 [[champion-challenger|現任冠軍制度]]；主線改向前的通用線成果見 [[exp-009-expectation-layer|EXP-009]]（暫停擴張、成果保留）；資料契約狀態機見 [[blockers|難點 B1]]；下一輪 EXP-F02（殘差辨識）與 EXP-F03（五臂）的設計要求見 [[exp-index|實驗索引]]。
