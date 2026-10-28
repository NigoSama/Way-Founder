# `dex_admin.bd_data_stats` 字段含义与生成逻辑

> 工单：[#4](https://github.com/NigoSama/Way-Founder/issues/4)（地图 [#1](https://github.com/NigoSama/Way-Founder/issues/1)）
> 调查日期：2026-10-28。后端生成代码**拿不到**，结论全部来自表结构/注释 + 数据反推。
> 标注：**【已证实】** = 用数据全表或抽样核对过；**【推断】** = 结构/少量样本支持，未逐条核对。
> 只读查询；`bd_data_stats` 全表约 1 万行，可以直接扫；`user` 只按索引键（id / invited_code）查。

## 0. 一句话结论

`bd_data_stats` 是**以 KOL 为根的邀请树、按天的节点快照**：每行 = 某天里树上的一个用户（`uid`）及其直接上级（`introd_uid`）。
- 无前缀字段（`transaction_volume` / `transaction_fee` / `recharge_amount` / `rebate_volume` / `rebate_cashback`）= **本人**；
- `dir_*` 字段和人数类字段（`register_user` / `transaction_user` / `ftt` / `ftd` …）= **直属下级（一层）**；
- `team_*` 字段 = **本人 + 全部多级下级（整棵子树）**；
- `team_profit = team_fee − team_rebate_volume − team_rebate_cashback`，10,274/10,274 行成立。

## 1. 表概况【已证实】

| 项 | 值 | 证据 |
|---|---|---|
| 行数 | 10,274（截至 date=20261027） | `COUNT(*)` |
| 日期范围 | 20260102 – 20261027，299 天有数据 | `MIN/MAX(date)` |
| `is_kol=1` | 4,818 行 / 134 个 uid | `GROUP BY is_kol` |
| `is_kol=2` | 5,456 行 / 1,673 个 uid | 同上 |
| 唯一键 | `UNIQUE (date, uid, introd_uid)` | `SHOW CREATE TABLE` |
| 刷新时间 | 每天 01:50（Asia/Shanghai）写入**前一天**的 `date` | `DATE_FORMAT(MAX(created_at))` for date=20261027 → `2026-10-28 01:50:15`；DB `@@session.time_zone=Asia/Shanghai` |
| `date` 的日界 | 北京时间自然日 | `register_user` 与 `user.ctime`（CST 存储）按北京日计数完全一致，见 §3 |
| 1–4 月的旧数据 | 2026-02-05 首次回填，2026-04-10 前后重跑过一次（`updated_at`） | `MIN(created_at)`、早期行 `updated_at` |

**每天行数很少，而且在下降**：1–4 月每天 30–57 行，5 月起 ~25 行，10 月下旬 3–11 行。原因见 §2：**只有当天某个字段非零的节点才写行**（全零行只有 6 行）。所以 PRD 说的"is_kol=2 每天只有几十行"并不是 is_kol=2 特有——整张表每天都只有几十行，is_kol=2 只是"当天有动静的非 KOL 下级"。

## 2. 行的结构：一棵以 KOL 为根的树

### 2.1 `uid` / `introd_uid`【已证实】
- `introd_uid` = 该 uid 的**真实邀请人**（`user.invited_code` 匹配到 `user.invite_code` 或 `user_invite_code.invite_code` 的那个 uid）。抽查 14 个 uid（58358→57121、92679→92468、96500→95090、100884→91462、101323→86382、93345→50593 等）全部一致。
- `introd_uid = 0` 只出现在 KOL 行，表示这个 KOL **没有邀请人**（如 57121 的 `invited_code` 为空）。
- 有邀请人的 KOL（如 58358 的上级是 KOL 57121，92679 的上级是非 KOL 92468），行里的 `introd_uid` 是那个邀请人，**不是 0**。⇒ **不能用 `introd_uid=0` 来筛 KOL 行，要用 `is_kol=1`（或 join `user_kol`）**。
- 除 4/20、4/24 的异常（§4）外，每个 uid 的 `introd_uid` 在全表中恒定。

### 2.2 谁会出现在表里【已证实 + 推断】
- 根 = KOL；行 = KOL 及其**多级**下级。例：4/1 上 92310(KOL) → 96796 → 96857，两层下级都有行；86382(KOL) → 91462(非 KOL) → 100884(KOL) → 100893…，树可以穿过非 KOL 继续往下，下面还能挂 KOL。
- `is_kol=2` 的 5,456 行里 4,696 行的上级是 KOL（=KOL 直属下级），760 行在更深层级。【已证实】
- 只有当天该行某个字段非零才写行（全表全零行只有 6 行）。所以"KOL 的直属下级"在表里**只能看到当天有动静的那部分**，不能用来数下级总人数。【已证实】
- KOL 若有非 KOL 上级，其上级不在表里（如 92468、95090、55101 从未出现为 uid），这类 KOL 行在当天没有父行。【已证实】

### 2.3 `is_kol`【已证实】
- 注释："1是kol，2否"。134 个 `is_kol=1` 的 uid 全部在 `dex_server_user.user_kol`（共 225 条，`kol_type` 全为 1）中。
- **是写行当时的快照**：20 个在 `user_kol` 里的 uid 有 `is_kol=2` 的行，其中 14 个后来变成 1，在 `user_kol.ctime` 之后变为 1。例：97547 `ctime=2026-03-24`，0209–0323 为 2、0324 起为 1；100041 同理。
- **KOL 关闭（`user_kol.status=0`）后仍是 1**：97547 于 2026-04-24 关闭，`is_kol=1` 一直持续到 0927。⇒ `is_kol=1` ≈ "当时在 user_kol 表里（不论 status）"。
- `user_kol` 现有 225 条中只有 94 条 `status=1`；134 个曾在表里出现过（其余 91 个 KOL 从未有过动静）。

## 3. 字段逐个说明

记号：**本人** = 该 uid 自己；**直属** = `introd_uid` 等于该 uid 的那一层下级；**团队** = 该 uid + 全部多级下级。

### 3.1 身份 / 维度
| 字段 | 含义 | 状态 |
|---|---|---|
| `id` | 自增主键（AUTO_INCREMENT 已到 284 万，说明历史上有大量删改/重跑） | 已证实 |
| `date` | 统计日，`YYYYMMDD` 字符串，北京时间自然日 | 已证实 |
| `introd_uid` | 真实邀请人 uid；0 = 无邀请人（仅 KOL） | 已证实 |
| `uid` | 节点用户 | 已证实 |
| `user_create_at` | 该 uid 注册时间（= `user.ctime`，100884 两边都是 `2026-04-01 14:41:56` CST） | 已证实 |
| `is_kol` | 写行时是否在 `user_kol`（1 是 / 2 否），见 §2.3 | 已证实 |

### 3.2 人数类（全部是**直属下级**的人数）
| 字段 | 含义 | 状态 / 证据 |
|---|---|---|
| `register_user` | 当天注册的直属下级人数 | **已证实**：按邀请码 + `user.ctime` 北京日核对 92310(9)、57121(2)、99870(9)、100709(104) 于 0401 全部一致 |
| `transaction_user` | 当天有交易的直属下级人数 | **已证实**：= 子行中 `transaction_volume>0` 的个数，3,485/3,493 个父行一致（8 个例外全是 §4 的异常行） |
| `recharge_user` | 当天充值的直属下级人数 | 推断：≥ 子行中 `recharge_amount>0` 的个数（2,643 等于、848 大于），多出的是没写行的下级 |
| `recharge_user_allseq` | "总钱包"口径的直属充值人次 | 推断：恒 ≥ `recharge_user`；"allseq"可能是人次而非人数 |
| `ftt` / `{chain}_ftt` | 当天首次交易的直属下级人数，及按链拆分 | 推断：`ftt ≤ transaction_user` 绝大多数成立；`ftt` = 各链之和在 10,181/10,274 行成立 |
| `ftd` | 当天首充的直属下级人数（看起来就是 SOL 口径） | 推断：`ftd = sol_ftd` 在 9,987/10,274 行成立 |
| `ftd_all` / `{chain}_ftd` | "总钱包"口径首充人数，及按链拆分 | 推断，**存疑**：259 行 `ftd > ftd_all`，两套口径关系不清，需后端确认 |

### 3.3 金额类：本人 / 直属 / 团队
| 字段 | 统计对象 | 含义 | 状态 / 证据 |
|---|---|---|---|
| `transaction_volume` | **本人** | 当天本人交易额 | **已证实**：父行 `dir_transaction_volume` = Σ 子行 `transaction_volume`，3,485/3,493 |
| `dir_transaction_volume` | **直属** | 直属下级交易额之和 | 同上 |
| `transaction_fee` | **本人** | 本人交易手续费 | **已证实**：`dir_transaction_fee` = Σ 子行 `transaction_fee`，3,485/3,493 |
| `dir_transaction_fee` | **直属** | 直属下级手续费之和 | 同上 |
| `recharge_amount` | **本人** | 本人充值额 | 推断（子行值能对上父行 `dir_recharge_amount` 的大部分） |
| `dir_recharge_amount` | **直属** | 直属下级充值额之和 | 推断：≥ Σ 子行 `recharge_amount`（2,712 相等、779 大于、2 小于）；多出的是只充值没写行的下级 |
| `recharge_amount_all` | **直属**（总钱包） | 直属下级"总钱包"充值额 | 推断：恒 ≥ `dir_recharge_amount`（0 行小于）；KOL 行里 `recharge_amount=0` 但 `recharge_amount_all>0` 很常见，说明它不是本人口径 |
| `rebate_volume` | **本人收到的** | 本人作为邀请人收到的返佣（来自下级交易，可多级） | **已证实**（样例）：91829 的交易产生 0.15322 返佣 = 上级 85105 收 0.14089 + 再上级 50012 收 0.01233 |
| `rebate_cashback` | **本人收到的** | 本人作为被邀请人拿到的交易返现 | 已证实（样例）：返现出现在有交易的叶子行 |
| `team_fee` | **团队** | 本人 + 所有多级下级的手续费 | **已证实**：`team_fee = 本人 transaction_fee + Σ 直属子行 team_fee`，3,482/3,493（例外为 §4 异常行和 86382@0406） |
| `team_rebate_cashback` | **团队** | 团队内所有人收到的返现之和 | **已证实**：`= 本人 rebate_cashback + Σ 子行 team_rebate_cashback`，3,493/3,493 |
| `team_rebate_volume` | **团队产生的** | 团队内交易产生的、付给**任何上级**（可在团队外）的返佣总额 | **已证实**（样例）：叶子 91829 的 `team_rebate_volume`=0.15322=它的两级上级收到的返佣之和；57121 的 = Σ 子行（19.07615+0.13029） |
| `team_profit` | **团队** | 团队利润 | **已证实**：`team_profit = team_fee − team_rebate_volume − team_rebate_cashback`，**10,274/10,274 行**误差 <0.0002；324 行为负 |

**注意 `team_rebate_volume` ≠ Σ 团队成员的 `rebate_volume`**：前者是"团队交易产生、付出去的返佣"（成本口径，包括付给团队之上的人），后者是"某人收到的返佣"（收入口径）。因此 `team_profit` 是"这棵子树的交易给平台带来的净手续费"。

### 3.4 分链字段（`{sol,bsc,base,xlayer,eth,monad,plasma}_*`）
| 字段 | 含义 | 状态 / 证据 |
|---|---|---|
| `{chain}_transaction_volume` | **直属**交易额按链拆分（不是本人！） | **已证实**：Σ 各链 = `dir_transaction_volume` 10,173/10,274；与本人 `transaction_volume` 对不上（6,505 行不等） |
| `{chain}_transaction_fee` | **直属**手续费按链拆分 | 同上（Σ 各链 = `dir_transaction_fee` 10,198/10,274） |
| `{chain}_rebate_volume` | 本人收到的返佣按链拆分 | 推断：Σ 各链 = `rebate_volume` 10,054/10,274 |

例：20261001 的 66570 行，`transaction_volume=0`、`dir_transaction_volume=65.77`、`sol_transaction_volume=65.77`；其下级 98699 行 `transaction_volume=65.77`、`sol_transaction_volume=0`。

**缺链**：101 行（全部在 **2026-07-17 之后**）各链之和 < `dir_transaction_volume`，从不大于。说明 7 月中旬起有一条新链（推断为 Robinhood 链或其他）计入总额但**没有单独列**。

## 4. 同一 uid 同一天多行（12 组）【已证实】

- 唯一键是 `(date, uid, introd_uid)`，所以同一 uid 同天多行只能是 `introd_uid` 不同。
- 全表只有 **12 组**，全部在 **20260420（7 组）和 20260424（5 组）**，涉及 8 个 KOL（58358、92293、93345、96500、99518、99870、100901、101323）。
- 每组两行：
  - `introd_uid=0` 那行：01:50 定时任务写入（`created_at` 04-21 01:50 / 04-25 01:50 CST），**数据残缺**：`dir_*`、`transaction_user`、`register_user`、`team_fee` 基本都是 0，`team_profit` 有时算出负数（如 100901 = −5.19561）。
  - `introd_uid=真实上级` 那行：当天下午手工重跑写入（04-21 16:46 / 04-25 17:48 CST），字段完整，和这些 uid 其他日期的行一致（它们其他日期 `introd_uid` 都是真实上级）。
  - 重跑时 `introd_uid` 变了，唯一键不冲突，旧的残缺行没被覆盖 ⇒ 这是一次性故障留下的脏行。
- **按 (date, uid) 逐列取 max 不对**：多数列结果碰巧等于完整行，但会破坏等式。反例：99518@0424，残缺行 `team_profit=0`，完整行 `team_profit=−0.37405`，取 max 得 0（错）；且逐列 max 拼出来的行 `team_profit ≠ team_fee − team_rebate_volume − team_rebate_cashback`。
- **正确去重**：每个 (date, uid) 保留**整行** `id`（或 `created_at`）最大的那一行；或直接删掉 `introd_uid=0 AND date IN ('20260420','20260424')` 的这 12 行。按整行去重后，父子求和关系 100% 成立。

## 5. 对 PRD 假设的核对

| PRD 假设 | 结论 |
|---|---|
| KOL 行 `transaction_volume=0`、`dir_transaction_volume>0` | **不总成立**。`transaction_volume` 是 KOL 本人交易额，KOL 自己也交易（373 行本人、直属都 >0；56422@0401 本人 2177、直属 0）。只是大部分 KOL 本人不交易 |
| 下级行的 `transaction_volume` = KOL 行的 `dir_` 值 | **只有 KOL 当天只有一个活跃直属下级时成立**。一般规律是 KOL 行 `dir_transaction_volume = Σ 直属子行 transaction_volume` |
| 约 10,261 行、1,793 uid、20260102–20261025 | 截至 1028 为 10,274 行（新增到 20261027），去重 uid 仍为 1,793（134 个出现过 is_kol=1，1,673 个出现过 is_kol=2，14 人两边都有） |
| 每天约 01:50 刷新 | 成立，写入的是前一天（北京时间） |

## 6. 对 A（统计表）vs R（原始表）选择的影响

1. **"KOL 的直接下级"类指标，A 能直接用**：取 `is_kol=1` 行（**不要**用 `introd_uid=0` 筛）的 `dir_transaction_volume`、`dir_transaction_fee`、`register_user`、`transaction_user`、`ftt`、`recharge_user`，口径就是 PRD 的"BD → KOL → 直接下级"。逐人明细取 `introd_uid ∈ KOL` 且 `is_kol=2` 的子行。
2. **`team_*` 是多级口径**，比 PRD 的"直接下级"大；且 KOL 下面还有 KOL（1,535 行 KOL 行的上级也是 KOL），**按 BD 把多个 KOL 的 `team_*` 相加会重复计算**。若 PRD 要"利润"，要么接受多级口径并按树去重（只加最上层 KOL），要么用 R 自己算直属口径利润。
3. **直属口径的返佣/返现/利润 A 里没有现成字段**：`rebate_volume` 是 KOL 本人收到的返佣（含多级来源），`team_rebate_*` 是多级。直属利润 = Σ 直属子行（`transaction_fee` − 该下级交易产生的返佣 − 该下级的 `rebate_cashback`）只能近似，建议 R。
4. **KOL 本人的交易被算作他上级 KOL 的直属交易**（如 58358 的交易进 57121 的 `dir_`）。BD 汇总时会不会把"下级 KOL 本人交易"既算进上级 KOL 的直属、又被当成 KOL 本人的数，需要在 PRD 里定规则。
5. **`is_kol` 是历史快照**：成为 KOL 之前的日期为 2，关闭后仍为 1。看板的 KOL 名单若以 `user_kol` 当前状态为准，应 join `user_kol`（`bd_name`）而不是信 `is_kol`。
6. **只写有动静的行**：下级总人数、累计活跃人数这类指标不能从 A 数，要走 R（`user.invited_code`）。
7. **充值类口径不清**：`recharge_amount` / `dir_recharge_amount` / `recharge_amount_all`、`ftd` / `ftd_all` 两套口径关系没法从数据完全确定（259 行 `ftd > ftd_all`），建议 R 或找后端确认。
8. **分链字段是直属口径，且 2026-07-17 起缺一条链**，若 PRD 要分链拆分需要注意或走 R。
9. **去重规则**：(date, uid) 取 `id` 最大的整行（只影响 12 组）。
10. **历史**：A 只到 20260102，之前要 R 补算（与地图里已有的 fog 一致）。

## 7. 主要查询（可复现）

```sql
-- team_profit 公式（10274/10274）
SELECT SUM(ABS(team_profit-(team_fee-team_rebate_volume-team_rebate_cashback))<0.0002) FROM dex_admin.bd_data_stats;

-- 父子关系：dir_ = Σ子行本人；team_fee = 本人 + Σ子行 team_fee
SELECT COUNT(*),
  SUM(ABS(p.dir_transaction_volume-c.s_tv)<0.01),
  SUM(ABS(p.team_fee-(p.transaction_fee+c.s_team_fee))<0.001),
  SUM(ABS(p.team_rebate_cashback-(p.rebate_cashback+c.s_trc))<0.001),
  SUM(p.transaction_user=c.n_trader)
FROM dex_admin.bd_data_stats p JOIN (
  SELECT date, introd_uid, SUM(transaction_volume) s_tv, SUM(team_fee) s_team_fee,
         SUM(team_rebate_cashback) s_trc, SUM(transaction_volume>0) n_trader
  FROM dex_admin.bd_data_stats WHERE introd_uid<>0 GROUP BY date, introd_uid) c
ON c.date=p.date AND c.introd_uid=p.uid;
-- 结果：3493 / 3485 / 3482 / 3493 / 3485

-- 重复组
SELECT date, uid, COUNT(*) FROM dex_admin.bd_data_stats GROUP BY date, uid HAVING COUNT(*)>1;

-- register_user 核对（user 库，按索引 invited_code）
SELECT p.pid, COUNT(u.id) FROM (
  SELECT id pid, invite_code code FROM dex_server_user.user WHERE id IN (57121,100709,99870)
  UNION SELECT uid, invite_code FROM dex_server_user.user_invite_code WHERE uid IN (57121,100709,99870)) p
JOIN dex_server_user.user u ON u.invited_code=p.code
 AND u.ctime>='2026-04-01 00:00:00' AND u.ctime<'2026-04-02 00:00:00'
GROUP BY p.pid;  -- 2 / 104 / 9，与 bd_data_stats 0401 的 register_user 一致
```

## 8. 仍待确认（需要后端）
- `recharge_amount` vs `recharge_amount_all`、`ftd` vs `ftd_all` 中"总钱包"的确切定义。
- 2026-07-17 起计入 `dir_transaction_volume` 但没有分链列的是哪条链。
- 写行的触发条件（推断为"任一字段非零"）。
- 交易额/手续费的计价单位（推断为 USD）以及与 order/finance 原始表的对账（本工单未做）。
