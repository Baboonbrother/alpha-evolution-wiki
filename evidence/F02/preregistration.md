# 預註冊：EXP-F02 殘差辨識——法人特徵對 king2 錯誤的樣本外增量

凍結日：2026-07-24。本文件在計算任何測試折指標**之前**寫定並 sha256 入 `prereg_freeze`。
owner 凍結約束（2026-07-24 F01.1 指令⑨⑩）逐條落實於下。

## 一句話

問：把法人變化型特徵加進**完整 king2 base model**，能不能在樣本外更準地辨識 king2 的兩類錯誤
（入選後大跌＝假陽性；落選後大漲＝漏網）？不是問單特徵顯著性，不看 Sharpe。

## 資料（凍結）

- `engine/out/king2_residuals_dataset.parquet`（2026-07-22 凍結；100 換股事件 2014-01～2026-06）。
- T1 假陽性任務：`selected==True` 列（1,200 筆，12/事件）；結果變數 `excess`（相對大盤超額）。
- T2 漏網任務：`selected==False` 列（4,209 筆，~42/事件）；結果變數 `excess_vs_pool`（相對股池超額）。
- **禁用 parquet 內建 `cls_*` 標籤**——那些用了全樣本分位數（正是 owner 警告的洩漏）。

## 標籤（owner 約束①：門檻只由訓練窗生成）

- T1 標籤＝`excess ≤ q10(訓練窗事件的 selected excess)`。
- T2 標籤＝`excess_vs_pool ≥ q90(訓練窗事件的 missed excess_vs_pool)`。
- 分位數只對**訓練窗事件的列**計算；同一門檻套到測試窗的實現結果定義測試標籤。validation 不參與門檻設定。

## 切分（owner 約束②③：換股月 grouped split、同持有窗不拆）

- 事件依時間排序做 walk-forward：expanding 訓練窗（最少 36 事件）→ 測試塊 12 事件。
- 折邊界：測試起點 {36, 48, 60, 72, 84, 96}＝6 折（最後一折 4 事件）。
- **Embargo 1 事件**：訓練窗最後 1 個事件不進訓練（其持有窗與測試窗入場相鄰）。
- 切分單位＝事件（換股月）＝持有窗，天然滿足「同持有窗股票不拆 fold」。

## 特徵（凍結）

- **base（完整 king2，9 欄）**：score、rev_rank、fp、chip、tilt、q_quality、adv_rank、
  pool_rank_pct（pool_rank/pool_size）、regime_weak。
- **base＋inst**：base 加法人特徵。法人特徵清單**由圖查詢產生**（owner 指令 Major 7）：
  `evi_edge` 中 EXP-F01.1 的邊、排除 REDUNDANT_SKIN、按 |mean overlap| 升冪——預期 11 個
  （含 PARTIAL_OVERLAP 的 inst_turn_60，帶旗標進入）。驗收：刪一條邊必須改變候選清單。
- 缺值：訓練窗中位數插補；標準化：訓練窗 z-score（std=0 剔除）。全部只用訓練窗統計量。

## 模型（凍結）

L2 邏輯迴歸（λ=1.0，截距不罰），numpy IRLS（tol 1e-8、上限 100 迭代）——確定性、零新套件；
考卷以 scipy.optimize 獨立最小化同一罰項 NLL 交叉驗證係數。base 與 base+inst 用**同一模型類**同一 λ。

## 指標（owner 約束④：incremental test，非單特徵顯著性）

每折測試窗算 base 與 base+inst 各自的：log loss、PR-AUC（average precision）、
T1 precision@2/事件（12 選 2 最高風險）、T2 precision@5/事件（~42 選 5 最高機會）。
增量 Δ＝(base+inst) − base，逐折配對。

## 裁決規則（事前定，純碼）

每任務獨立裁決：

- `INCREMENTAL_RESIDUAL_INFORMATION`（provisional）：mean ΔPR-AUC > 0 **且** ΔPR-AUC>0 的折 ≥4/6 **且** mean Δlog loss < 0。
- `NO_INCREMENTAL_EVIDENCE`：mean ΔPR-AUC ≤ 0 **或** ΔPR-AUC>0 的折 ≤2/6。
- 其餘＝`MIXED_EVIDENCE`。

次要報表（不參與裁決，只報告）：2017+ 分段（chip 活資料段）逐折 Δ、逐年 ΔPR-AUC 符號穩定表。

## 證據層級與家族帳

- 全部歷史資料＝EXPOSED——結論最高只到 **PROVISIONAL**；不觸金庫、不產 confirmed、不看 Sharpe、不建投組。
- 搜尋帳家族 `KING2_RESIDUAL_DISCRIM`，本輪燒 **m=2** 測試（T1、T2）。
- 圖回寫：每任務一條 `evi_edge`（帶逐折 metric、valid_under、run_hash、data_snapshot_id），
  裁決為何寫何（含誠實負結果）。

## 事後禁止

跑完後不得：改折邊界、改 λ、改 k、改裁決門檻、換結果變數、以分段結果替代主裁決。
若機器故障需改碼，改動與理由記入結果檔 `deviations` 欄（空陣列＝零偏離）。
