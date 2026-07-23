---
title: 實驗 F02：殘差辨識——F02 spec 下未偵測到法人增量（NOT_DETECTED_UNDER_F02_SPEC）
group: 實驗
exp: F02
---

# 實驗 F02：殘差辨識——這個鏟法沒挖到礦（不等於山裡沒有礦）

**一句話結論（F02.1 修正口徑）：11 個法人特徵在「月頻錨、全部一起放入、線性邏輯迴歸、固定 λ、絕對漲跌二元標籤」的 spec 下，未偵測到樣本外增量——雙任務 NOT_DETECTED_UNDER_F02_SPEC。** 這證明的是「這個鏟法沒挖到礦」，**不是**「法人這座山沒有礦」：無信賴區間、無等效區間，「未偵測到」不得升格為「證明不存在」（與「不顯著≠independent」同一條紀律）。

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

## 結果：雙 NOT_DETECTED_UNDER_F02_SPEC（F02.1 換名,原判 NO_INCREMENTAL_EVIDENCE 已 supersede）

| 任務 | mean ΔPR-AUC | mean Δlog loss | ΔPR-AUC>0 折數 | 裁決 |
|---|---|---|---|---|
| T1 假陽性（入選後大跌） | **−0.003** | **+0.011**（變差） | 2/6 | NOT_DETECTED_UNDER_F02_SPEC |
| T2 漏網（落選後大漲） | **−0.012** | −0.000 | 1/6 | NOT_DETECTED_UNDER_F02_SPEC |

三個必要的解讀，缺一都會誤讀：

1. **這不是「殘差不可測」**。base 模型自己有辨識力：測試折 PR-AUC 0.16–0.34，對比盛行率 0.08–0.25，約 1.5–2 倍 lift——king2 自己的分量（分數低空飛過、regime、排名位置）本來就攜帶「哪些選擇比較危險」的訊息。加法人特徵之後**沒有變得更準**，T1 的 log loss 六折有五折變差＝加進去的主要是雜訊（22 維 vs 9 維，樣本外校準劣化）。
2. **這不是機器沒檢定力**。考卷含合成雙向測試：同一台機器在「真有增量訊號」的合成資料上判出 INCREMENTAL（ΔPR-AUC +0.102、5/6 折正）、在純雜訊上不誤判。負結果是資料說的，不是機器聾。
3. **這與「低相關≠增量」一致但射程有限**。F01 說法人軸與 king2 分數低重疊（相關 <0.33）；F02 說在本 spec 下**未偵測到**這個低重疊軸攜帶 king2 錯誤的增量資訊。低相關的特徵可能只是不一樣的雜訊——但 T1 log loss 變差也可能來自 all-11 稀釋、固定 λ 對 22 維不公平、或 base 共線（score=rev+tilt、tilt=fp+chip 同時入模）——**這幾種目前分不開**，正是宣稱必須綁 spec 的原因。

## 圖回寫與家族帳

- `evi_edge` ×2：`feature_family:institutional_flow_change --no_incremental_evidence--> king2_residual:{selected_crash, missed_soar}`，帶逐折 metric、valid_under（period/universe/model/labels/**PROVISIONAL_HISTORICAL_EXPOSED**）、run_hash、data_snapshot_id。負結果與正結果同格式入圖——但邊的語義（F02.1 修正）是 `not_detected_under_f02_spec`，valid_under 綁定 spec 六欄（linear_logistic／λ=1 固定／all-11-joint／absolute-tail-binary／monthly-anchor／selection layer）：**封的是這個鏟法，不是 institutional_flow_change 家族**。
- 搜尋帳家族 `KING2_RESIDUAL_DISCRIM` 燒 m=2（T1、T2）。
- [證物包 evidence/F02/](https://github.com/Baboonbrother/alpha-evolution-wiki/tree/main/evidence/F02)：預註冊原文＋run/source manifest＋完整結果 JSON＋考卷輸出。

## 誠實邊界

- **裁決只在本設計射程內**：線性邏輯迴歸、λ=1.0、11 特徵全進（無特徵選擇）、兩個二元任務。法人特徵在**非線性模型、交互項（法人×regime、法人×集保）、或持有期減碼 overlay**（日頻，非入選辨識）中是否有用，本實驗**沒有測**——那是設計外，不是被否證。
- ~~次要分段：2017+~~ **F02.1 標記 VACUOUS_SEGMENT_CHECK**：所有測試折本來就從 2018-12 起，「2017+ 分段」＝主結果自身，不是 robustness 證據。真正的敏感性＝訓練窗也限 2017+ 的 retraining，收進矩陣引擎欄位。逐年符號（T1 −−+−+−、T2 +−−−−−）仍如實保留。
- 歷史資料全 EXPOSED——本頁一切最高 PROVISIONAL，金庫無涉，零 confirmed 宣稱。
- 事後零偏離（`deviations: []`）：折邊界、λ、k、裁決門檻與預註冊逐字一致。

## ⚠ F02.1 修正（owner 揪出六類問題,2026-07-24）

1. **宣稱過強**：NO_INCREMENTAL_EVIDENCE 讀起來像「證明沒有增量」——但 F02 無 CI、無事件 block bootstrap、無等效區間，只能說「未偵測到」。裁決換名 `NOT_DETECTED_UNDER_F02_SPEC`，圖邊 supersede 為 `not_detected_under_f02_spec` 並綁 spec 六欄。
2. **標籤混合兩種問題**：絕對門檻（訓練期 p10/p90）讓模型可以靠「認出壞月份」得分（regime_weak 在同事件內全體相同），不必認出「12 檔裡哪檔最危險」——Event Risk 與 Within-Event Ranking 未拆開。「base 能辨識哪些選擇危險」可能主要是「哪些月份危險」。
3. **λ 公平性**：9 維 vs 20 維共用固定 λ=1.0 不代表相同有效複雜度；base 高度共線。log loss 變差有四種成因（真雜訊/稀釋/λ 失配/共線干擾）分不開。應 nested train-only 選 λ 或 base-residual 正交測試。
4. **all-11 整包**：檢驗的是「11 特徵聯合」，不是「法人家族內有沒有某個機制有用」——投信/外資/自營/共識/加速反轉應按經濟角色分 block 預註冊。
5. **折權重**：末折只有 4 事件卻與 12 事件折同權；缺 pooled OOF、事件數加權、排除末折敏感性。
6. **2017+ 分段空洞**：見誠實邊界的 VACUOUS_SEGMENT_CHECK。

以上統計補強（bootstrap CI／等效區間／pooled OOF／事件加權／任務拆分／nested λ／經濟 block）**不再開 F02.2 小輪，照 owner 裁決收進矩陣研究引擎的欄位**。F02 保留為矩陣第一格：`institutional_all × monthly_selection × absolute_tail_binary × linear_logistic × all_11_joint → NOT_DETECTED`。

## ⚠ 冠軍身分警示（2026-07-24 對帳中）

F02 的 base model 是 AARO 凍結的「king2 研究鏡像」。同日冠軍身分對帳發現：**真錢 live 下單線預設策略是 STR_king1（王牌本尊：純 REVYOY 條件選股＋硬閘＋領先股群體狀態機＋時間政策）**，s994 的 STR_king2 是強化版候選，而 AARO 鏡像又是 king2 的 score 節點簡化版（Top-12 月錨，非 0.93 cutoff 可變檔數、無 S_IND 狀態機）。**因此本頁「king2 的錯誤」精確地說是「鏡像分數節點的錯誤」**——在冠軍身分對帳與王牌消融完成前，F02 結論不得宣稱為「改善或否決王牌」。見 [[champion-identity]]。

## 對下一輪的含義

矩陣引擎（owner 新指令）取代單點實驗節奏；法人單軸**暫不**降級——要等矩陣把 within-event ranking／nested λ／經濟 block／holding overlay 格子跑完仍為負，才把法人從 selection 層降去 holding/exit overlay。集保變化家族與法人×集保時序超邊照原排程。

延伸：重疊圖與 F02 約束來源見 [[exp-f01-king2-overlap]]；king2 殘差四格定義見 [[champion-challenger]]；全部實驗一覽 [[exp-index]]。
