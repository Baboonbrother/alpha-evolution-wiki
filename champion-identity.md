---
title: 冠軍身分對帳：王牌其實是兩條策略＋一台共用的時間狀態機
group: 制度
---

# 冠軍身分對帳（2026-07-24，owner 第一工項）

owner 貼出「#王牌策略」原始碼後的第一個問題：**這跟 wiki 凍結研究的 king2 是同一條策略嗎？** 答案：**不是。它們是同一血統的兩條策略，而且真錢自動線跑的是 king2_deploy、owner 貼的是王牌本尊 king1。** 三路取證（AARO 鏡像稽核／部署鏈稽核／helper 語義稽核）＋對抗查證，證據如下。

## 三個身分，各是什麼

| 身分 | 定義處 | 選股 | 現況 |
|---|---|---|---|
| **STR_king1（王牌本尊）** | `s993_str_for_import.py`（owner 貼文＝此檔逐字） | 純 `RK(REVYOY)>0.93`（b1-b4 閘內，**可變檔數**）；b1＝純 ROE | 手動路徑引用（s1_trade fallback、_app_live）；**血統祖先** |
| **STR_king2_deploy（自動線現任）** | `s994_str_enhanced.py:171`＝`STR_king2(adv_floor=0.3, top_k=12, harden_gates=True)` | score＝0.65rev＋0.35(指紋＋集保籌碼)/2，gate-passers **固定 top-12**＋流動性地板；b1＝三維品質複合 | **真錢自動線在跑**：king-order.service（下單 5577）→`launch_app(strategy='king2_deploy')`；king-dashboard.service（7795）預設同源 |
| **AARO 研究鏡像** | `engine/residuals.py` | king2_deploy 選股層**逐字忠實**（top-12／地板／三維閘／harden 全對上） | 持有層簡化：月錨→下錨前5日單段毛報酬；**S_IND 狀態機／5日冷卻／skip_holiday／RP 成本全不入鏡**（deviation 自報） |

**兩條策略共用同一台時間狀態機**（差異只在選股分數與閘）：月營收公告錨凍結名單 → 下錨前 5 日先出 → 領先股群體 S_IND 感測器 → 熄火後 5 日冷卻 → 假日調整 → close 執行。owner 說「王牌的靈魂在整套階層政策」——**這台機器兩條王牌都有，而 F01/F02 只研究了選股分數節點**。

## 王牌的四層（owner 解讀,對帳後確認屬實）

```
L1 個股閘   b1 ROE>30%位 × b2 200日位置前10% × b3 10日位置前25% × b4 15日動能前40%
L2 條件選股  全市場 RK(REVYOY) 先算 → 套閘遮罩 → >0.93（順序釘死;非遮罩後重排名）
L3 群體聚合  s1&s2&s3 選領先股 → 5日報酬取平均 → td2 廣播 → >1.005 ＝內生市場感測器
L4 狀態機   熄火後 extend_false_forward 5日冷卻（hysteresis）
L5 時間政策  月營收週期凍結名單＋下錨前5日先出＋假日調整
L6 執行     RP＝finlab sim, trade_at='close', fee 1.425‰/3
```

## 逐項稽核結果（owner 六個必查問題）

1. **wiki 冠軍是否凍結錯策略？** 部分對、部分錯。champion_registry 早已凍結 `CHAMP-king2-v1`＝king2_deploy 並註明「真部署」——**自動線身分沒凍錯**；錯的是敘事：wiki 一直把 king2 稱作「王牌」，而 owner 心中的王牌是 king1，且鏡像只忠實選股層。修正：`CHAMP-king1-original` 已另冊註冊（lineage `king1-line`，STR_king1 逐字＋17 個 helper 全 hash 凍結），**兩冊分開，禁止混稱**。
2. **trade_at='close' 的 same-bar 風險？** RP＝finlab `sim(trade_at_price='close')` 逐字凍結在案；finlab sim 的執行語義（訊號日 t 用 t 收盤還是 t+1）＝**待實測確認**——列為消融矩陣第一批的執行軸（同日close／shift(1)隔日open／shift(1)隔日close 三臂），不從記憶斷言。
3. **09-25～11-10 季節窗？** **全目錄死碼**。`pos2 = mask_date_range(pos,...)` 在 s993 算完即丟（`return_pos` 與 `RP` 都用 `pos`），s991 同樣；從未餵給任何回測、下單或監控。**live 王牌全年運作**。季節窗若要啟用＝新策略變體，需先過日期擾動＋安慰劑窗＋leave-one-year-out（owner 指定的 seasonal overfit 稽核），不是直接打開。
4. **count_flt / b5？** 死碼（buy 未用 b5）。危險點確認：`count_flt` 用 `.iloc[-1]`（最後一日全市場檔數套全歷史）——若未來啟用 b5 必須改逐日 count series，否則未來資訊污染。
5. **`A & mask_intersection(A,B)` 冗餘？** 讀實作＋實測：`mask_intersection` 只在 index/columns **交集**格寫入 A&B、交集外保持 True 且**恆保 A 的形狀**（原生 `&` 會對齊成聯集形狀）。實測 A 有 2 個索引日不在 B 的交易日曆（月營收截止日落非交易日）→ mask 版在那裡保留 A 的值，**與原生 A&B 差 24 格**（22,140 vs 22,116 True）——**近冗餘但非嚴格等價**，第三層作用＝形狀保持與非交集寬鬆。矩陣鏡像必須逐字重現此語義，不得用 `A&B` 替代。
6. **領先股群體為空時？** 讀實作＋實測：`td2` 是**純廣播、無 ffill**→空群體日 cohort 平均＝NaN→`NaN > 1.005 = False`→當日判「不持有」並觸發 5 日冷卻延伸。**實測 4,732 個交易日中有 199 天（4.2%）是空群體 NaN 日**——王牌約 4% 的持有決策由 pandas NaN 比較**沉默決定**（方向 fail-closed，但未明文）。鏡像與矩陣實作必須顯式寫出這條規則，消融矩陣應加「空群體＝沿用前一狀態」對照臂看它是保護還是傷害。

## ⚠ 對抗查證挖出的部署發現（歸 owner 拍板,研究線不動真錢）

1. **兩條產單路徑用不同策略變體**：互動下單 app（5577）pin 死 `king2_deploy`（top-12＋地板＋harden，log 實證「策略=king2_deploy｜凍結 12 檔」）；但**無頭日終線**（crontab 20:00 `run_daily_deploy.sh` → `deploy_today.py` → `edge_supervisor.current_holdings()`）用的是 `STR_king2()` **基礎版**（0.93 可變檔數、無流動性地板）——top-K 截斷與地板會改名單，**兩路產出的持股清單可能分歧**。哪條是權威、要不要統一＝owner 決定；研究線只入帳不修改。
2. **同型病曾發生並被修過**：`launch_order.py` 註解「修 king2 錯配」＋log 歷史顯示，下單 app 曾一度跑 king2 基礎版與作戰台顯示不一致，後被 pin 死 king2_deploy——上面那條日終線分歧很可能是同一次修正的漏網面。
3. **對抗查證的誠實紀錄**：本輪初稿曾誤判「live＝king1」（被 s1_trade fallback 誤導），被對抗查證員以 systemd→launch_order→log 三級證據反駁後修正——fallback 階梯的精確語義是：給字串→直接用；無字串無預跑 report→'king2'；**有預跑 report→'king1'**（＝_app_live 的手動形狀，king1 只在這條舊手動路徑上活著）。

## 對研究路線的含義

- **F01/F02 的宣稱進一步縮小**：base＝king2_deploy 鏡像的**選股分數節點**——不含 S_IND 狀態機、不含冷卻、不含 sellforward 動態。「king2 的錯誤」精確說是「鏡像分數節點的錯誤」。
- **法人/TDCC 特徵的未測空間比想像大**：對 S_IND 市場狀態、冷卻天數、提前退出、持有監控的作用全部沒測過——這正是 owner 說「F02 只測第一種就不能代表法人無用」的機器版證據。
- **下一工項＝王牌完整消融**（owner 指令）：先懂王牌靠哪個結構賺錢，再談加特徵。分區塊：A 選股結構（REV only／Price gates only／REV×gates／逐閘）、B 市場狀態（無 S_IND／有／有＋冷卻／大盤 regime 對照／隨機 regime 安慰劑）、C 時間政策（提前 0-7 日／季節窗有無＋日期擾動）、D 完整超邊四臂。執行軸含 same-bar 三臂稽核。**凍結骨架＋一次一部件**，不做全排列挑最高 Sharpe。
- 在消融完成前，**法人／TDCC 矩陣結果不得宣稱為「改善或否決王牌」**（owner 指令 13，已入紀律）。

## 凍結證物

`CHAMP-king1-original`：STR_king1 逐字（sha `bb75e2709ab2d10c`）＋s993/s0_basedata 全檔 sha＋17 個 helper 逐函式 sha，入 `champion_registry`（lineage `king1-line`）與 `engine/out/champion_king1_freeze.json`。真錢線全程唯讀零寫入。

延伸：king2_deploy 的完整凍結規則見 [[champion-challenger]]；F02 受本對帳影響的宣稱降級見 [[exp-f02-residual-discrimination]]。
