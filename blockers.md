---
title: 難點與卡關：這條線現在真正卡在哪（v2，B0 置頂）
group: 方法論
---

# 難點與卡關：這條線現在真正卡在哪（v2）

這一頁不講成果，只誠實記錄阻擋這台引擎往「自主發現可交易 Alpha」再走一步的難點。**v2 改版說明**：初版把難點按「能否靠寫程式解」分類，被 owner 指出兩個問題——①**漏掉了最嚴重的 B0**（系統自己可修的證據語義錯誤，反而把外部限制列得很誠實）；②**把幾個「尚未做」寫成了「外部現實無法解」**（B1 資料其實存在只是未接入、B2 只有最終確認要等、B4 大半可靠統計設計解）。本版逐條更正，並記錄哪些已修。

## B0【已修】證據語義錯誤：「未拒絕交互」曾被寫成「證明獨立」

**這是初版漏掉的最嚴重難點，而且是系統自己可修的。** graph-native 曾把 n=8、n=17、t 不顯著的交互檢定寫成 `independent` 並封閉圖缺口——但同一台機器的 `min_n=20` 在信念結算會判 `HOLD_PRIOR`。「沒測出交互」被寫成「證明沒有交互」，圖裡因此累積 6 條假獨立邊，系統誤以為那些問題已解決。

| 原況 | 正確語義（已實作，`wm/evidence.py`） |
|---|---|
| n=8、t 不顯著 → independent | `inconclusive(LOW_N)`——樣本不足什麼都不能宣稱，含獨立 |
| n=25、t=0.11 → independent | `inconclusive(UNDERPOWERED)`——CI [−0.034,+0.038] 遠超 ±0.002 band |
| 未拒絕交互 | ≠ 證明獨立 |
| 寫入邊即封閉格子 | 證據不足**不封閉**（AWAITING_MORE_DATA ≠ CLOSED） |

**修法（全部落地、考卷驗證）**：①`independent` 從此要**等效性檢定**才配——效應的 95% CI **整段**落在事前指定的無經濟意義區間內（報酬單位 ±0.002≈半個來回成本；無因次 ±0.01），「有足夠證據說效應小到不值錢」而非「沒找到效應」。②缺口格三態 `OPEN / AWAITING_MORE_DATA / CLOSED_CONCLUSIVE`，inconclusive 不封閉、也不同資料重測（重測必同結果），重新可測的觸發＝LIVE_FORWARD 新資料。③6 條假獨立邊以 `edge_supersession` 更正——**舊列不刪、新列接手、關係入帳**（appendix-only 下的正確更正法）；重算結果：4 條 LOW_N、2 條 UNDERPOWERED、**0 條真獨立**。④schema 遷移擴 `relation` CHECK 加入 `inconclusive`（逐列 sha 前後比對）。修正後真跑三輪：新裁決全部 `inconclusive(LOW_N)`、格進 AWAITING、OPEN 8→7→6 遞減不迴圈。

**教訓**：在 B0 修好前繼續擴圖，會把大量「不知道」寫成「已知獨立」——這比圖很小更危險。難點排序必須把「系統自己可修的語義錯誤」放在「外部限制」之前。

## B1 世界鏈前半段未接入——是「資料取得專案」，不是「資料不存在」

**初版說「無真資料源」是過度分類。** BDI/SCFI 運價、新聞、供應鏈資料並非不存在，而是未偵察／未接入／PIT 品質未知。正確的表述是每個源有狀態：

| 狀態 | 意義 | 目前歸屬 |
|---|---|---|
| `SOURCE_UNKNOWN` | 尚未偵察 | BDI/SCFI 運價、公開供應鏈關係 |
| `SOURCE_EXISTS_UNLICENSED` | 有資料但未取得權限 | （待偵察後歸類） |
| `SOURCE_NON_PIT` | 有歷史但存在事後修訂/回填 | （待偵察後歸類） |
| `SOURCE_TOO_SHORT` | 歷史不足 | mcm 新聞管線（真歷史僅 15 天） |
| `SOURCE_READY` | 可進資料金庫 | 產業月營收（已接，世界鏈第一跳） |

它是**可推進的資料取得專案**：偵察→授權→PIT 驗證→實體歸戶→入庫。世界鏈現在接到「產業營收世界狀態」（真 finlab），下一跳的前置是把上表的 UNKNOWN 變成明確狀態。

**【已升級第一塊】資料源偵察器帳本層落地**（`wm/source_scout.py`，append-only＋latest-wins＋無證據列非法）：owner 點名「系統會探索特徵、還不會自主探索資料源」——偵察器是補這個缺口的迴路（圖譜缺口→搜源→評估 PIT/歷史/授權/成本→connector→特徵→檢驗→回寫）。第一個走完「偵察→評估→記錄」的源＝Polymarket（[[exp-009-expectation-layer|EXP-009 預備]]：三族群可行、191 檔含完整裁決規則、真 PIT 時戳；狀態 `SOURCE_BLOCKED_LOCAL`＝台灣 DNS 攔截、合規判斷歸 owner）；worklist 已種入其餘八源（IV/skew、Trends、AIS、夜光、招聘、流量、海關、信用隱含）。**迴路的自動化（自主搜源與評估）仍未做**。

## B2 只有「最終確認」必須等未來——研究不必停等

**初版說「時間換不來」只對一半。** 對的部分：Alpha 的**最終 live confirmation** 只能等 LIVE_FORWARD 累積（歷史段全 EXPOSED，這是金庫紀律的代價與價值）。錯的部分：等待期間研究**不必停**，可以：

- 建 paper／shadow forward book（候選先進模擬帳排序，如既有 paper-account）
- 巢狀 walk-forward、跨市場／資產類別的可遷移性測試（用未使用期間）
- 估容量、成本、訊號衰減速度（不需新資料）
- 收集未來事件但**延遲揭露結果**（accumulate-blind：事件先記錄、結果晚點才准看）

正確表述：**Alpha 最終升級等 LIVE_FORWARD；研究方法與候選排序不停等。**

## B3【已修一半】圖不需要等 confirmed 才長大——證據層級已實作

**初版把「能進圖」與「能解鎖 C 臂」混在一起。** 已修：圖節點現在帶證據層級 `PROPOSED → EXPLORATORY → SUPPORTED → INCONCLUSIVE → REFUTED → CONFIRMED`（`wm/evidence.py`）——`EXPLORATORY` 與 `INCONCLUSIVE` 的邊**一樣是合法圖內容**（現在圖上 6 條更正邊全是 INCONCLUSIVE、新邊也如實標），只有 `CONFIRMED` 能進 king2 挑戰者或下單。圖從此不因 confirmed=0 而長不大。仍待做：主效應軸仍只 1 條（B-RES-001）、可算家族仍只 2 個——擴大靠更多信念跑過 settle（部分受 B2）＋家族軸接更寬特徵（B5）。

## B4【已修一半】交互檢定的統計設計——2×2 中位切丟太多訊息

**初版說「單一產業湊不齊 2×2 是資料量問題」不準確——大半可靠統計設計解。** 已修第一步：新增 `continuous_interaction` form（事件內連續交互回歸，標準化 y~zA+zB+zA·zB 的交互係數逐事件收集再跨事件檢定）——不做中位切、不丟組內連續訊息。仍待做（owner 開的清單）：跨事件 pooling＋event fixed effect（把 n 從事件數升到列數，需 cluster SE）、產業階層模型／partial pooling（讓單一產業借力全體）、時序 lead-lag、保留事件結構的 permutation test。**B4 應列為「統計設計待改」，不是資料量的死路。**

**測量層實錘（exp-008 稽核揪出、已修一半）**：日頻 IC 的 naive t 因 20 日重疊窗自相關（lag-1≈0.94）**系統性高估 3.5~4 倍**——曾造成 exp-008 兩個假 dev pass。已加 `harness.nw_tstat`（Newey-West）並在 world_alpha 的 dev/val 閘採用；歷史頁面的 naive t（如開放探索 low_vol t=15）讀時要打折。仍待做：整個評分器（`evaluate`/`judge`）換 NW 屬 evaluator_version 升版——**在那之前，任何用 naive t 的 dev 篩選都會系統性放行重疊灌水的候選**。

## B5 超邊函數形式仍少——純工程，可做

form 從 2 種擴到 3 種（median_split／rank_product／continuous_interaction），但離「HyperedgeSpec → FeatureDSL → ExperimentSpec」的完整 FeatureDSL 還遠（門控、多軸、非線性）。接[[fw-feature-algebra|特徵代數]]的 DSL 是明確可執行的工程。

## B6 還沒找到可交易 Alpha——誠實結果，不是卡關

到目前全部真跑結果：殘差假說 NARROW_SCOPE、自主輪全 HOLD_PRIOR、開放探索 0/4 過確認閘、graph-native 交互修正後**全 inconclusive**（B0 前曾誤標 independent）。系統很會拒絕自己，還沒發現扣成本、出樣本外仍可交易的增量。這不是 bug——紀律正確地擋下弱訊號。寫程式改善的是搜尋效率與誠實度，不保證找到 Alpha。

## B7【已修一半】append-only 與迭代的摩擦——supersession 已落地，三帳分離未做

**已修**：①`edge_supersession` 表（append-only）——更正不刪列，舊列永存、新列接手、關係入帳，B0 的 6 條更正就是首例；②`spec_hash`（假說身分）與 `run_hash`（資料＋程式＋本次執行指紋）已拆開並都入帳。**仍待做**（owner 開的規格）：dev／staging／canonical 三帳分離、每次執行帶 run_id、canonical 只接受一次 promotion——目前開發期仍用「scratch 副本＋正式帳一次性清 dev 產物」的過渡做法，那正是這條要根治的。

## 一句話收斂（v2 更正版）

初版的分類錯了兩處，這版承認並修正：**最該先做的從來不是資料偵察，是系統自己可修的證據語義（B0，已修）**；而「外部現實」清單裡，B1 是可推進的資料取得專案、B2 只有最終確認要等、B4 大半是統計設計。真正只能等的只有「Alpha 最終確認」（LIVE_FORWARD）一件事。修正後的優先序（照 owner 的排序）：**B0 語義（✅）→ 封閉規則（✅）→ spec/run hash 拆分（✅）→ 證據層級（✅）→ 統計設計（🔶 continuous form 已加，pooling/階層待做）→ 資料源偵察與 FeatureDSL（未動）。**

延伸：B0 的修正細節與更正的 6 條邊見 [[graph-native|graph-native]]；金庫與 confirmed 慢路見 [[autonomous-research|自主研究]]；「找得到 ≠ 賺得到」見 [[discipline|誠實紀律]]。
