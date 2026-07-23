---
title: 實驗 F02：殘差辨識——法人特徵對 king2 錯誤無樣本外增量（誠實負結果）
group: 實驗
exp: F02
---

# 實驗 F02：殘差辨識——「低相關≠增量」的實證版

**一句話結論：F01 找到的低重疊法人軸，在本設計下對辨識 king2 的錯誤沒有樣本外增量——雙任務 NO_INCREMENTAL_EVIDENCE。** owner 在 F01.1 的警告「只證明長得不太一樣，還沒證明地下有礦」被 F02 直接驗證：這一鏟下去，沒有礦。

## 問題（owner 凍結的問法）

不是「法人特徵顯不顯著」，是：把法人變化型特徵加進**完整 king2 base model**，能否在樣本外更準地辨識 king2 的兩類錯誤？

- **T1 假陽性**：入選的 12 檔裡，哪些會大跌（excess 落入訓練窗 p10 以下）？
- **T2 漏網**：落選的 ~42 檔裡，哪些會大漲（excess_vs_pool 超過訓練窗 p90）？

## 設計（預註冊 872ebadf，跑前凍結入 `prereg_freeze`）

owner 三條凍結約束逐條落實，每條都有考卷驗證：

1. **標籤門檻只由訓練窗生成**——p10/p90 只對訓練窗事件的列計算，validation 不參與門檻設定。parquet 內建的 `cls_*` 標籤（全樣本分位數）**禁用**——那正是洩漏。考卷逐折從原始資料手算門檻比對，且證明逐折門檻各不相同（12 個門檻 12 個值）。
2. **換股月 grouped split＋embargo**——切分單位＝事件＝持有窗（同持有窗股票天然不拆 fold）；訓練窗最後 1 個事件 embargo（其持有窗與測試入場相鄰）。
3. **incremental test，非單特徵顯著性**——base（king2 九分量：score/rev/fp/chip/tilt/q_quality/adv_rank/pool_rank_pct/regime_weak）vs base＋11 法人特徵，同一模型類（L2 邏輯迴歸 λ=1.0，numpy IRLS，考卷用 scipy 獨立最小化交叉驗證係數差 <1e-4）。**先不看 Sharpe**。

另兩個結構件：**法人候選由圖查詢產生**（讀 `evi_edge` 的 F01.1 邊、排除換皮、按重疊升冪——考卷驗收「刪一條邊→候選清單改變」，圖真的在決定實驗輸入，owner Major 7 落地）；6 折 expanding walk-forward（訓練窗最少 36 事件、測試塊 12 事件，2018-12～2026-06 共 64 個樣本外事件）。

## 結果：雙 NO_INCREMENTAL_EVIDENCE

| 任務 | mean ΔPR-AUC | mean Δlog loss | ΔPR-AUC>0 折數 | 裁決 |
|---|---|---|---|---|
| T1 假陽性（入選後大跌） | **−0.003** | **+0.011**（變差） | 2/6 | NO_INCREMENTAL_EVIDENCE |
| T2 漏網（落選後大漲） | **−0.012** | −0.000 | 1/6 | NO_INCREMENTAL_EVIDENCE |

三個必要的解讀，缺一都會誤讀：

1. **這不是「殘差不可測」**。base 模型自己有辨識力：測試折 PR-AUC 0.16–0.34，對比盛行率 0.08–0.25，約 1.5–2 倍 lift——king2 自己的分量（分數低空飛過、regime、排名位置）本來就攜帶「哪些選擇比較危險」的訊息。加法人特徵之後**沒有變得更準**，T1 的 log loss 六折有五折變差＝加進去的主要是雜訊（22 維 vs 9 維，樣本外校準劣化）。
2. **這不是機器沒檢定力**。考卷含合成雙向測試：同一台機器在「真有增量訊號」的合成資料上判出 INCREMENTAL（ΔPR-AUC +0.102、5/6 折正）、在純雜訊上不誤判。負結果是資料說的，不是機器聾。
3. **這正是「低相關≠增量」的實證版**。F01 說法人軸與 king2 分數低重疊（相關 <0.33）；F02 說這個低重疊軸**不含**關於 king2 錯誤的增量資訊。兩者完全相容——與 king2 長得不一樣的特徵，多數只是不一樣的雜訊。owner 在 F01.1 強制的宣稱降級（`LOW_OBSERVED_OVERLAP` 不得寫成 incremental）現在有了實驗背書。

## 圖回寫與家族帳

- `evi_edge` ×2：`feature_family:institutional_flow_change --no_incremental_evidence--> king2_residual:{selected_crash, missed_soar}`，帶逐折 metric、valid_under（period/universe/model/labels/**PROVISIONAL_HISTORICAL_EXPOSED**）、run_hash、data_snapshot_id。負結果與正結果同格式入圖——下一輪選題時，圖知道「法人→殘差」這條路已走過且不通。
- 搜尋帳家族 `KING2_RESIDUAL_DISCRIM` 燒 m=2（T1、T2）。
- [證物包 evidence/F02/](https://github.com/Baboonbrother/alpha-evolution-wiki/tree/main/evidence/F02)：預註冊原文＋run/source manifest＋完整結果 JSON＋考卷輸出。

## 誠實邊界

- **裁決只在本設計射程內**：線性邏輯迴歸、λ=1.0、11 特徵全進（無特徵選擇）、兩個二元任務。法人特徵在**非線性模型、交互項（法人×regime、法人×集保）、或持有期減碼 overlay**（日頻，非入選辨識）中是否有用，本實驗**沒有測**——那是設計外，不是被否證。
- 次要分段（不參與裁決）：2017+（chip 活資料段）ΔPR-AUC 同樣為負（T1 −0.003、T2 −0.012）；逐年符號 T1 為 −−+−+−、T2 為 +−−−−−，無任何年度族群翻轉主結論。
- 歷史資料全 EXPOSED——本頁一切最高 PROVISIONAL，金庫無涉，零 confirmed 宣稱。
- 事後零偏離（`deviations: []`）：折邊界、λ、k、裁決門檻與預註冊逐字一致。

## 對 F03 與下一輪的含義

F03 原設計五臂裡的「B：＋法人 re-rank」臂，其動機（法人加進選股改善排名）被 F02 直接削弱——**照 owner 排程，下一輪應走集保變化家族（Round 2）與法人×集保交互超邊（Round 3）**，而不是把已知無增量的法人單軸推進 F03 成本閘。法人軸剩餘的未測空間收窄為：持有期 overlay（日頻退出訊號）與交互項。

延伸：重疊圖與 F02 約束來源見 [[exp-f01-king2-overlap]]；king2 殘差四格定義見 [[champion-challenger]]；全部實驗一覽 [[exp-index]]。
