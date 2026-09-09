# SGS 项目维护 Skill（v49a final）

## v49a 最终成功状态

- v49a 是当前最终成功版本。
- 决斗误触发破军 bug 已成功修复。
- 界徐盛已完美实现。
- 破军只在【杀】触发，一次【杀】内只确认一次。
- 官方 PoJunNew 二层面板可扣置手牌/装备，破军标记正常，当前回合结束返还。
- 白银狮子/同类减伤 pipeline 正常。
- 普通整局稳定，无莫名弃牌或回合错乱。

## 后续维护原则

- 以后以 v49a 为最终成功基线。
- 不要重新引入 v42a/v43a 中造成整局结算错乱的全局风险写法。
- 不要把 parent Sha +0x13f 当安全字段盲写。
- 不要恢复 Lua 全局规则桥接。
- 每次只做小步补丁，并先验证普通局稳定。

---

# 三国杀手游 iOS 单机武将移植 Skill

## 适用场景

当用户要求继续《三国杀手游 / 三国杀单机版 iOS IPA》武将、技能、资源、`game.lib`、Mach-O native patch、Codex/Ghidra handoff、重签前打包等任务时使用本 Skill。

本 Skill 特别适用于当前项目：在稳定 4.4.2.4 iOS 单机 IPA 中加入曹冲 174 与界徐盛 491，并继续实现界徐盛新破军。

## 最高优先级原则

1. 不要假装完成 native 规则接入。
2. 不要跨版本直接等同技能 ID。
3. 不要把 303/402/414 当作曹冲/界徐盛承载入口。
4. 不要把普通徐盛旧破军 414 伪装成界徐盛新破军。
5. 不要在 Lua 全局伤害、濒死、酒杀、动画链路做边缘桥接。
6. 每次生成 IPA 前必须说明实际写入内容、未实现边界、静态校验结果。
7. 用户能接受多次安装签名测试，因此可以按 v10a/v10b/v11 分阶段尝试，但每一版都要尽量保持可回滚。

## 当前项目状态

### 目标角色

- 曹冲：角色 ID `174`
  - 称象：`441`
  - 仁心：`442`
- 界徐盛：角色 ID `491`
  - 新破军 / PoJunNew：`11146`

### 当前推荐基线

总回滚安全基线仍为：

```text
sgs_jiexusheng_v13_rollback_safe.ipa
```

该文件内容与 `sgs_jiexusheng_v12b_selection_stable.ipa` 一致。

界徐盛新破军继续开发的当前**已验证工作基线**为：

```text
sgs_jiexusheng_prompt_parent_sha_filter_v31a.ipa
```

用户已真机验证 v31a：

- 只有界徐盛本人使用【杀】时出现“新破军”确认/取消 action 提示。
- 其他武将使用【杀】不会误出现该提示。
- 界徐盛选择“取消”后，原【杀】能够正常继续询问【闪】并正常结算。
- 第一层 trigger/candidate/action/prompt 生命周期已打通，可冻结。
- 点击“确定”后仍会卡住，说明第二层 11146 effect / 目标牌选择 native 链尚未接通。
- v10 条件伤害 +1 仍保留，并已验证可与酒叠加。
- 曹冲技能尚未 native 接入。

## 已知有效结论

### game.lib / Lua 客户端层

`game.lib` 解密后确认为有效 ZIP，目标模块为 LuaJIT bytecode：

- `app.gamelogic.spell.sanjiang.ChengXiang`，ID `441`
- `app.gamelogic.spell.sanjiang.RenXin`，ID `442`
- `app.gamelogic.spell.yijiang.PoJunNew`，ID `11146`

三者均通过 `PubGsCUseSpell` 发送请求。

`PoJunNew` 目标不是在请求表中重新携带，而是从当前 action 的 `targetSeatId` 读取。因此仅发送 `PubGsCUseSpell(11146)` 不足以触发 UI；必须让客户端进入 `11146` 的 trigger/effect action 状态。

### PubGsCUseSpell 单机编码

单机分支序列化字段：

```text
srcSeatId: UByte
destCount: UByte
useCardCount: UByte
user_param[1]: UInt
user_param[2]: UInt
spellId: UShort
data[1..destCount+useCardCount]: UInt
```

单机分支不写入：

```text
seatId
chrId
spellIndex
```

### Android native 参考能力

已确认语义参考：

- `CSpell::DisCardFromRole`
- `CRole::ToggleTurnOver`
- `CMoveCardAction::MoveCardsFromRole`
- `CSpellMgr::PreventReciveDamageUseSpellEffect`
- `CSpellMgr::AddReciveDamageHpUseSpellEffect`

这些只作为语义/样本参考，不能直接把 Android 地址当 iOS 地址。

### iOS native 已知地址

```text
Sha::Resolve = 0x10016effc
ask-shan 前候选点 = 0x10016f26c 前
CDamageAction::Resolve = 0x1003059a4
AddDamage-like chain = 0x10030630c
AddDamage 调用点 = 0x100305fd8
```

字段：

```text
CDamageAction +0x88 = source
CDamageAction +0x90 = target
CDamageAction +0x98 = 当前 damage
CDamageAction +0x9c = final/副本 damage
```

### v10 native damage patch

v10 修改点：

```text
Patch point VA: 0x100306388
Patch point file offset: 0x306388
Stub VA: 0x100002a00
Stub file offset: 0x2a00
```

逻辑：

```text
if CDamageAction 来自 Sha action
and source 拥有 11146
and target 两组牌区数量 <= source 对应两组牌区数量:
    ++*(uint32_t *)(CDamageAction + 0x98)
return to original chain
```

用户实测结果：

- 普通结算正常。
- 其他武将正常。
- 界徐盛满足条件时伤害 +1。
- 可与酒杀叠加。

后续不要再动这个加伤 hook，除非有明确 bug。

## 已失败路线，禁止重复

### 414 母体路线

不能用 `414`。当前目标 IPA 里普通徐盛 `PoJun 414` 实际效果为：杀造成伤害后，目标翻面并摸 X 张。该逻辑不是界徐盛新破军。

### 11146 请求层路线

单纯发送：

```text
PubGsCUseSpell(11146)
PubGsCMoveCard(11146)
```

不会生效，因为单机 native 没有完整 11146 handler。

### Lua 全局桥接路线

不要在这些位置模拟规则：

- `GameAnimLogic`
- 濒死桃数计算
- 酒杀/伤害动画链
- 全局伤害结果处理

曾导致：酒杀只伤 1、2 血攻击特效异常、濒死显示 0 桃。

### Lua 前置 UI hook 路线

不要在 `app.gamelogic.card.normal.Sha` 或启动/初始化期 Lua chunk 中挂全局前置 UI continuation。v13 曾导致游卡 logo 初始动画反复弹出、不推进。

## 继续实现界徐盛的推荐流程

当前界徐盛已完成：条件伤害 +1。

剩余目标：

```text
出杀指定目标后、询问闪前
→ 弹出新破军确认/目标牌选择 UI
→ 选择目标手牌/装备，数量 <= 目标当前体力
→ 扣置这些牌，使本回合目标视为没有
→ 回合结束返还原区域
```

推荐分阶段：

### v14a：找到正确原生技能提示 action

目标不是移动牌，而是找到已有“杀指定目标后、询问闪前”的技能提示创建机制。

优先分析：

- 雌雄双股剑
- 铁骑
- 烈弓
- 其他杀指定目标后、ask-shan 前触发的技能/装备

要找的信息：

```text
1. native 如何创建 trigger/effect action
2. 客户端如何被告知某技能正处于可发动状态
3. action 中 targetSeatId 如何设置
4. 如何让 GameTable:isSpellTriggerOrEffect(11146) 为真
5. 如何恢复原 Sha::Resolve 流程
```

不要只构造 `PubGsCUseSpell(11146)`。

### v14b：只弹 UI 验证版

目标：界徐盛使用杀指定单一目标后，在 ask-shan 前弹出 PoJunNew UI。确认/取消都必须能回到正常杀/闪流程。不得移动牌，不得改伤害。

### v15：扣置牌

目标：根据 UI 选择结果，移动目标手牌/装备到临时区或武将牌旁。必须记录：

```text
cardId
原 role/seat
原 zone：手牌 or 装备
原 position/index，如需要
当前界徐盛回合标记
```

优先参考：

- `ProcessMoveCard`
- `CMoveCardAction::MoveCardsFromRole`
- 过河拆桥/顺手牵羊目标牌选择与移动流程

### v16：回合结束返还

目标：界徐盛当前回合结束后，把扣置牌还给原目标的原区域。

必须先找到可靠的回合结束 hook，不要在全局动画/桌面刷新中猜。

### v17：整合回归验证

验证列表：

- 普通杀正常。
- 酒杀正常。
- 濒死桃数正常。
- 其他武将正常。
- 界徐盛条件伤害 +1 正常。
- 酒 + 界徐盛加伤叠加。
- 新破军 UI 出现在 ask-shan 前。
- 取消新破军后杀流程继续。
- 确认新破军后目标牌被扣置。
- 扣置牌本回合不在目标手牌/装备区。
- 回合结束返还到正确角色和区域。

## 处理 IPA/game.lib 的注意事项

1. 修改 `game.lib` 后必须重新加密，并更新 `flist1` 中 `hotfix2/game.lib` 的 MD5/size。
2. 打包 IPA 优先使用 Python `zipfile`，保持中文 app 路径、metadata、文件权限和顺序，避免 shell zip 造成中文路径问题。
3. 修改 Mach-O 后移除旧 `_CodeSignature`，用户安装前会自行重签。
4. 每个输出 IPA 必须附报告，说明：
   - 基线
   - 改动文件
   - patch VA/file offset
   - 静态校验
   - 已知边界
   - 推荐实测点

## Codex/Ghidra 最小 prompt 模板

当需要 Codex 继续 native 定位时，不要让它重扫 APK/Lua/XML/资源。使用：

```text
不要查资源、XML、game.lib、PoJunNew 是否存在。那些已经确认。

当前基线：sgs_jiexusheng_v13_rollback_safe.ipa。
已验证：v10 native damage patch 成功，界徐盛条件伤害 +1 可与酒叠加。
不要改 v10 damage hook。
不要改 414。
不要做 Lua 全局桥接。

现在只查：已有杀指定目标后、ask-shan 前的原生技能提示 action 创建机制。
优先对象：雌雄双股剑、铁骑、烈弓、其他杀后 ask-shan 前技能。

输出：
1. 函数 VA / file offset
2. 调用链
3. 关键伪代码
4. 如何设置 skill trigger/effect 状态
5. action.targetSeatId 如何设置
6. 如何让客户端 PoJunNew 读取到当前目标
7. 最小 patch 点
8. 覆盖指令、trampoline、跳回地址
9. 风险
```

## 回答用户时的风格

- 简洁直接。
- 不要反复解释“为什么不能”。
- 如果能做，就生成 IPA；如果不能做，说明缺的精确信息。
- 用户接受失败版，但不接受假完成。
- 每次失败要保留可回滚安全基线。

## 最新维护记录：v14 UI 触发重新判断

用户已验证 `sgs_jiexusheng_v13_rollback_safe.ipa` / 等价 v12b 回滚安全版一切正常；继续以它作为安全基线。

重新检查 `game.lib` 后，v11/v12 UI 不弹的更可能原因是：`PoJunNew` 的目标牌选择 UI 不是由 `PubGsCUseSpell(11146)` 或普通 `GsCTriggerSpellNew` 驱动，而是由 `GsCRoleOptTargetNtf` 进入 `OnRoleOptTargetLogic` 的 `OPT_SKILL_FLAG1 = 28` 分支打开。

关键客户端链路：

```text
GsCRoleOptTargetNtf
  optType = OPT_SKILL_FLAG1 (28)
  spellId = SKILL_CHARACTER_PO_JUN_NEW (11146)
  optSeatId = 界徐盛 seat
  targetSeatId = 当前杀目标 seat
  spellCasterSeat = 界徐盛 seat
    -> OnRoleOptTargetLogic
       -> pushActionToSeat(TYPE_SPELL_EFFECT, spellId=11146, targetSeatId=...)
       -> addDialogToSeat(..., UISelCardDlg.new(rep))
       -> UISelCardDlg 对 PO_JUN_NEW 开启 hand/equip 选择，数量上限为目标 currHp
       -> 确认后发送 PubGsCMoveCard(spellId=action.spellId, data=selected cardIds)
```

因此下一步 v14a 应尝试构造/发送 `GsCRoleOptTargetNtf` 语义通知，而不是继续构造 `PubGsCUseSpell(11146)`。v14a 目标只验证 UI 出现与取消/确认接续，不移动牌、不改伤害。

需要 Codex/Ghidra 聚焦查：

1. iOS native 中 `GsCRoleOptTargetNtf` 或等价 opt-target 通知的构造/发送函数。
2. `OPT_SKILL_FLAG1 = 28` 的现有 native 样本。
3. `PoJun / ChiYan / TaoMie / YanZhu / WeiKui` 等进入 `OnRoleOptTargetLogic -> UISelCardDlg` 的样本。
4. 这些样本如何填写 `timeOut / optSeatId / targetSeatId / spellCasterSeat / spellId / optType / param / param2 / dataCnt`。
5. 如何在 `Sha::Resolve` ask-shan 前调用同一个通知发送函数。

禁止继续重复：直接发 `PubGsCUseSpell(11146)`、Lua 启动期 hook、414 母体、全局伤害/濒死桥接。

## 最新实测澄清：没有任何新破军提示

用户最新澄清：当前可用回滚安全版中，界徐盛使用【杀】后没有出现任何新破军相关 UI；不仅没有目标牌选择界面，也没有技能提示的确认/取消弹窗。实际生效的只有 v10 native damage hook：满足条件时【杀】伤害 +1，并可与酒叠加。

这会影响后续排查顺序：

```text
不是：确认/取消弹出成功 → 选牌/MoveCard 失败
而是：Sha 指定目标后 → 11146 trigger prompt/action 根本没有创建
```

因此继续实现时，第一步应定位 iOS native 中已有“杀指定目标后、询问闪前”的技能/装备提示 action 创建路径，例如雌雄双股剑、铁骑、烈弓等，而不是直接尝试 `UISelCardDlg` 或 `PubGsCMoveCard`。

更新后的推荐 v14/v15 方向：

1. 先找出 native 如何创建“可发动技能确认/取消” action。
2. 确认这个 action 如何让客户端认为 `GameTable:isSpellTriggered(spellId)` 或 `isSpellTriggerOrEffect(spellId)` 为真。
3. 对 11146 注入等价 trigger action，先只验证确认/取消提示出现。
4. 只有确认/取消提示稳定后，再接入 `GsCRoleOptTargetNtf / OPT_SKILL_FLAG1=28` 或 `UISelCardDlg` 选牌流程。
5. 最后处理 native `PubGsCMoveCard(11146)`、扣置与回合结束返还。


## 2026-09-02 术语修正：不要混淆两层 UI

用户再次澄清：此前多次说“没有 UI”，准确含义是：

```text
界徐盛使用【杀】指定目标后，连“新破军是否发动”的确认/取消提示都没有出现。
没有进入确认阶段，也没有进入目标手牌/装备选择阶段。
当前只有 v10 native 被动加伤生效，且可与酒叠加。
```

因此后续所有分析必须区分：

```text
第一层 UI：技能触发确认/取消提示
  - 目标：让客户端认为 11146 处于 triggered / trigger action 状态。
  - 这是当前真正缺失的部分。

第二层 UI：目标手牌/装备选择窗口
  - 目标：确认发动后，进入 UISelCardDlg / opt-target 选牌流程。
  - 这一步不能提前当成当前问题的根因。
```

此前对“没有 UI”的记录若产生歧义，一律按最新澄清修正为：**没有第一层确认/取消提示**，而不是“确认后没有选牌窗口”。

### 对 v11/v12/v13 失败的重新解释

- v11a / v12a 失败点：没有让 11146 的 trigger prompt/action 成立；不是证明选牌 UI 入口错误。
- v13 失败点：Lua 前置 UI continuation 污染启动流程，导致游卡 logo/初始动画循环；不是证明 native 触发不可行。
- `PubGsCUseSpell(11146)` 更像确认后的请求；在 trigger/effect action 尚未存在时单独发送，不会让确认提示出现。
- `GsCRoleOptTargetNtf / OPT_SKILL_FLAG1 / UISelCardDlg` 更接近第二层“选牌 UI”路径；不能把它当成已经验证过的第一层确认提示入口。

### 当前修复顺序

```text
v16a：prompt-only
  只尝试在 Sha 指定目标后、ask-shan 前弹出“是否发动新破军”的确认/取消提示。
  不打开选牌窗口，不扣牌，不改伤害。

v16b：confirm -> target card UI
  只有在 v16a 确认/取消提示成功后，再让确认分支进入目标手牌/装备选择。

v16c/v17：扣置与返还
  在选牌 UI 稳定后，再接 MoveCard / 临时区 / 回合结束返还。
```

### native 下一步重点

优先定位或复用 iOS native 中已有“杀指定目标后、询问闪前”的 trigger action 创建机制，尤其是雌雄双股剑、铁骑、烈弓、猛进/谋溃等技能。

当前重点怀疑对象：

```text
Sha::Resolve 中调用 0x1002359e8 的分支
例如 optType = 0x1c / 28 的 action 创建路径
```

但必须解决 `11146` 与该 action 的绑定问题。也就是说，下一步不是继续手写 `PubGsCUseSpell(11146)`，而是确认：

```text
1. optType/spellId 映射表是否包含 11146；
2. 如果不包含，是否能安全把 11146 注入到 optType=28 或相近 trigger prompt 类型；
3. Sha::Resolve 何处应该创建该 action 并设置 Sha 等待状态；
4. 确认/取消后原 Sha ask-shan 流程如何恢复。
```


## v16a prompt-first diagnostic attempt

Generated artifact:

```text
sgs_jiexusheng_prompt_first_v16a.ipa
sgs_jiexusheng_prompt_first_v16a_report.txt
```

Purpose: test only the first-level confirm/cancel prompt.

Patches:

```text
1. Sha::Resolve early state0 hook
   VA 0x10016f06c / file offset 0x16f06c
   Branch to header-cave stub at 0x100002b00.

2. Header-cave stub
   Checks CRole::HasSpellId(source, 11146).
   If false, returns to original flow 0x10016f070.
   If true, branches to existing Sha::Resolve optType=0x1c/28 action creation at 0x10016f278.

3. optType=0x1c mapping diagnostic injection
   VA 0x100235534 / file offset 0x235534
   Replaces mapping skillId 0x25c with 11146 / 0x2b8a.
```

Limits:

- Diagnostic only.
- Does not implement hand/equip selection.
- Does not move, hide, or return cards.
- Does not alter v10 AddDamage hook.
- May fail if optType=28 is not the first-level confirm prompt action or if reused Sha state 0xb is wrong.
- If it opens a selection window directly, record that as “phase-2 path triggered,” not prompt success.


## 2026-09-04 关键维护记录：v28 → v31 第一层 prompt 真正打通

### 已验证里程碑

`v28a` 首次让 `11146` 成功进入原生 `optType=0x1c / 28` 的 candidate/action/prompt 生命周期：

- 界徐盛出【杀】时出现“新破军”确认/取消。
- 选择取消后原【杀】继续正常。
- 但只要场上存在界徐盛，其他武将出【杀】也会错误出现该提示。
- 点击确定会卡住，因为 11146 第二层 native effect/Resolve 尚未实现。

`v29a` / `v30a` 为修 ownership filter 先后失败：普通武将提示消失，但界徐盛自己的提示也消失。不要重复这两种错误。

`v31a` 最终修复：

```text
sgs_jiexusheng_prompt_parent_sha_filter_v31a.ipa
```

用户真机验证：

```text
普通武将出杀 -> 无新破军提示
界徐盛出杀 -> 有确认/取消提示
界徐盛取消 -> 原杀正常继续
```

因此第一层 prompt 从 v31a 起视为**已完成、冻结**。后续第二层/扣置/返还 patch 不应重新设计或替换这条链。

### v28a 为什么会“所有武将都触发”

核心并不是 `optType=28` 时机错误，而是 ownership 所用对象理解错误。

`optAction + 0x88` 在这条 opt-action Resolve 中不是固定的 Sha source，而是**当前正在遍历/检查的 role**。opt action 会遍历场上角色。v28a 对这个 current role 调 `HasSpellId(11146)`，于是：

```text
某角色使用 Sha
-> opt action 遍历全场 role
-> 遍历到界徐盛时 HasSpellId(11146)=true
-> 当前这张 Sha 获得 11146 candidate
```

所以场上只要有界徐盛，别人的【杀】也会出现新破军提示。

### v29a 错误：`role + 0x1c4` 不是 Character ID

不要再使用：

```text
*(role + 0x1c4) == 491
```

原生代码在多个路径以 byte/seat 语义读取 `+0x1c4`；它不是 XML `Character ID`。将其与 `491` 比较会把界徐盛本人也过滤掉。

### v30a 错误：跨 Sha state 误继承 `x20` 语义

必须先解 `Sha::Resolve` jump table。已确认：

```text
state 0 -> 0x10016f04c
state 1 -> 0x10016f340
state 2 -> 0x10016f278   # optType=28 创建点
```

每次 `Sha::Resolve` 重新进入时，开头先：

```text
x20 = [Sha + 0x70]
```

只有 state0 的局部分支随后才执行过类似：

```text
x20 = sourceRole + 0x1e0
```

但 state2 不经过那条赋值。因此在：

```text
0x10016f28c mov x0,x20
0x10016f290 mov w1,#0x1c
0x10016f294 mov x2,x19
0x10016f29c bl sub_1002359e8
```

此时的 `x20` 是 `[Sha+0x70]` 的 game/context，而不是 `sourceRole+0x1e0`。

所以 v30a 的：

```text
sourceRole = *(optAction+0x70) - 0x1e0
```

建立在错误跨-state寄存器语义上，必然不能正确识别界徐盛。

### v31a 正确 ownership 数据流

`sub_1002359e8` 创建 opt action 时会保存调用者传入的 parent action：

```text
optAction + 0xa0 = parent action
```

在 Sha state2 调用点：

```text
x2 = x19 = parent Sha*
```

因此 v31a 从**明确的 parent Sha**恢复真正攻击者，而不是从 current iterated role 或 game/context 猜：

```text
parentSha = *(optAction + 0xa0)
sourceRole = *(parentSha + 0x110)
if sourceRole == null:
    sourceRole = *(parentSha + 0x90)

currentRole = *(optAction + 0x88)

accept 11146 only if:
    currentRole == sourceRole
    and sourceRole.HasSpellId(11146)
```

这样 opt action 仍可正常遍历全场，但只有“当前遍历角色就是本张 Sha 的真实 source，并且真实 source 拥有 11146”时才允许生成新破军 candidate。

### 以后逆向/patch 的强制经验

1. **先解 state machine / jump table，再解释寄存器。** 不得把 state0 某寄存器的临时语义直接带到 state2/stateN。
2. **action 字段必须沿构造器/父子 action 数据流证明。** 不因某偏移在另一个 action 类型里表示 source，就默认此 action 类型同义。
3. **优先依赖 parent action。** 对 trigger/opt action，若有 parent 指针，先从 parent 恢复 source/target/context，再做 ownership。
4. **区分 current iterated role 与 actual source role。** `optAction+0x88` 在本链是遍历角色，不是固定攻击者。
5. **不要把 seat/byte 字段当角色 ID。** `+0x1c4` 不能用于比较 XML Character ID 491。
6. **旧分析报告只是线索，不是事实。** 每次出现“静态逻辑看似正确但真机失败”，必须回到真实机器码/jump table重新验证，不在错误模型上叠补丁。
7. **行为实测优先。** v28 的“全局提示”、v29/30 的“全部消失”本身就是对象语义证据，应反推模型。
8. **分层冻结。** v31 已完成第一层 prompt。后续只接第二层 effect/UI，不再动 v31 第一层逻辑，除非出现明确回归。
9. **每个新版本只改变当前层。** 第二层版本必须验证 v31 的三项 smoke test：其他武将无提示、界徐盛有提示、取消正常。

### 当前后续顺序（从 v31a 开始）

```text
第一层：11146 trigger confirm/cancel       DONE / FROZEN (v31a)
第二层：Confirm -> target hand/equip UI     NEXT
第三层：选中牌扣置/临时移出原区域           TODO
第四层：界徐盛回合结束返还                   TODO
第五层：全量回归（含 v10 +1 / 酒叠加）       TODO
```

第二层优先复用 4.4.2.4 客户端已经存在的：

```text
GsCRoleOptTargetNtf / 等价语义
optType = OPT_SKILL_FLAG1 (28)
spellId = 11146
optSeatId = 界徐盛
targetSeatId = 当前 Sha 目标
spellCasterSeat = 界徐盛
-> OnRoleOptTargetLogic
-> TYPE_SPELL_EFFECT action
-> UISelCardDlg
```

此阶段目标只要求“点击第一层确定后出现目标手牌/装备选择窗口，并能取消/返回”；不要同时实现扣置与返还。

## 2026-09-04 第二层静态定位：Confirm -> 官方目标牌选择 UI

> 状态：以下 native 地址/字段已由 4.4.2.4 iOS 机器码与 Android 同名符号交叉确认；`v32a` 仍需用户真机验证，不能提前标记为完成。

### 客户端第二层入口已确认

`PoJunNew.castSpell()` 在第一层确认后发送 `PubGsCUseSpell(11146)`。客户端真正打开目标手牌/装备选择窗口的通知是：

```text
GsCRoleOptTargetNtf
opcode = 21215 / 0x52df
optType = 28 / OPT_SKILL_FLAG1
spellId = 11146
optSeatId = 界徐盛 seat
targetSeatId = 当前 Sha 目标 seat
spellCasterSeat = 界徐盛 seat
-> OnRoleOptTargetLogic
-> TYPE_SPELL_EFFECT action
-> UISelCardDlg
```

因此第一层 Confirm 后不应继续试图“再发一次 prompt”或手写 Lua 选牌窗口，而应进入原生 `RoleOptTarget` 通用机制。

### iOS 通用官方 helper

已精确定位：

```text
CAction::AskClientResponseSpell = 0x10023fa6c
```

它直接从静态 `0x52df` 消息模板构造 `GsCRoleOptTargetNtf` 并通过 `CGame vtable +0x50` 发送。

参数语义由 iOS 机器码和 Android 同名函数交叉确认：

```text
x0 = this CAction*
w1 = spellCasterSeat
w2 = spellId
w3 = optSeatId
w4 = targetSeatId
w5 = optType
w6 = timeOut
w7 = param
stack arg #9 = boost::function<void()> callback
```

不要再手写 0x52df packet layout；优先调用这个 helper。

### 第一层 Confirm 卡住的真实位置

`CTriggerAction::NetMsgUseSpellRpy` 已精确定位：

```text
0x1002371ac
```

其正常路径：

```text
读取 MsgUseSpell spellId
-> CSpellMgr::CastAsSpell
-> CSpellMgr::CastSpell
-> 返回 0x15 时推进 trigger
```

11146 在旧 iOS native 中没有对应 class/factory，因此第一层 Confirm 后在这里无法完成正常 `CastAsSpell/CastSpell`，这解释 v31 的“点确定卡住”。

第二层最小修复原则：

```text
只在 NetMsgUseSpellRpy 收到 spellId == 11146 时：
    绕过缺失的 CastAsSpell/CastSpell
    调 CAction::AskClientResponseSpell(... optType=28 ...)
其他 skillId：
    完整执行原逻辑
```

### CTriggerAction 已有 source/target 上下文，不再猜 Sha vector

第一层 native prompt builder `sub_1002364b0` 已证明：

```text
CTriggerAction +0x88 = 当前被 prompt 的 role
(CRole +0x1c4)       = 该 role 的 seat id
CTriggerAction +0xa8 = 当前 prompt target seat
```

其中 `+0xa8` 来自 `sub_1002359e8` 创建 action 时保存的 x3；Sha state2 调用前通过 Sha virtual `+0x70` 得到当前目标并传入 x3。

因此 v32 第二层应直接复用第一层 action 已验证的上下文：

```text
role = *(trigger + 0x88)
sourceSeat = *(uint32_t *)(role + 0x1c4)
targetSeat = *(uint32_t *)(trigger + 0xa8)

AskClientResponseSpell(
    trigger,
    sourceSeat,     // caster
    11146,
    sourceSeat,     // opt seat
    targetSeat,
    28,
    timeout,
    0,
    emptyCallback
)
```

这里 `+0x1c4` 可以使用，因为此处明确需要的是 seat id；禁止把它重新解释为 XML Character ID。

### v32a 诊断版边界

`sgs_jiexusheng_confirm_to_selectui_v32a.ipa`：

- 基于用户已验证的 v31a。
- 第一层 registration/candidate/source filter 完全不动。
- v10 条件伤害 +1 完全不动。
- 只 hook `CTriggerAction::NetMsgUseSpellRpy +0x20`（VA `0x1002371cc`）。
- 非 11146 replay 原 `mov x1,x22` 后回 `0x1002371d0`。
- 11146 调 `CAction::AskClientResponseSpell`，目标是出现官方 `UISelCardDlg`。
- 本版不宣称 `PubGsCMoveCard(11146)` 已有 native handler；选牌后的真正扣置可能继续卡住，属于下一层。

v32 真机验收顺序：

```text
1. v31 smoke test 仍全通过
2. Jie Xusheng Sha -> first confirm appears
3. Confirm -> official target hand/equip selection UI appears
4. UI target/name/current HP/max selection count correct
5. 记录 Cancel 和 Confirm-selection 后的具体行为
```

只有第 3-4 项真机通过后，才能冻结第二层 UI，并进入 `PubGsCMoveCard(11146)` native handler / 扣置阶段。

## 2026-09-04 维护纠正：v32 Confirm 闪退 + 点将缓存问题重新审计

### v32a 真机结果：失败，禁止重复

用户真机反馈：

```text
v32a 第一层提示仍可出现；
点击“确定”后立即闪退。
```

因此前一节“直接在 `CTriggerAction::NetMsgUseSpellRpy` 中调用 `AskClientResponseSpell(..., optType=28)`”只保留为**失败记录**，不能再作为推荐实现。

### v32 闪退的根因：action 生命周期错误，不是简单 ABI 错误

重新核对 `CAction::AskClientResponseSpell(0x10023fa6c)` 的 iOS 原生调用样本后确认：

- x0..x7 参数语义此前基本正确；
- 第 9 参数 empty `boost::function<void()>` 的栈传法也与原生样本一致；
- 所以 v32 的主要问题不是“函数地址错/第九参数完全错”。

真正的问题是调用层级：

```text
第一层 Confirm
-> CTriggerAction::NetMsgUseSpellRpy
-> 原生应完成：candidate 匹配 / 第一层等待态清理 / CastAsSpell 或 CastSpell / child effect action 创建 / trigger 推进
```

v32 却在这些步骤完成之前：

```text
同一个 CTriggerAction
-> 再次 AskClientResponseSpell
-> 再注册一次客户端等待/timeout/callback
-> 手动 return
```

这会让仍处于第一层等待状态的 TriggerAction 再承担第二层等待，破坏 action queue / timer / callback 生命周期。

另外，`CTriggerAction` vtable 中对应后续选择/移动牌回包的槽位并没有 11146 所需的专用处理逻辑；即使不立即闪退，后续第二层回包也没有正确 native receiver。

**强制结论：第二层不能继续复用同一个 CTriggerAction。第一层 Confirm 后必须创建独立 child action。**

### 旧 iOS 与新版客户端的协议代际差异

客户端 `PoJunNew` 能识别：

```text
GsCRoleOptTargetNtf optType=28 / OPT_SKILL_FLAG1
```

但扫描旧 iOS native 对 `CAction::AskClientResponseSpell` 的全部静态调用后，没有找到现成 `optType=28` 的本地 action 生命周期样本。

普通 `PoJun 414` 的 native `Resolve` 确实调用同一个 helper，但使用：

```text
optType = 24 / OPT_SEL_ACT
```

并且 PoJun 自己有专用 response handler（约 `0x100187f58`）。

因此：

- 可参考 414 的**父子 action / 等待 / 回包生命周期框架**；
- 不可复用 414 的技能规则效果；
- 不可把客户端新版 `optType=28` 直接塞给旧 `CTriggerAction` 后返回。

### 第二层下一步优先容器：CMoveCardAction

已定位 `CMoveCardAction`：

```text
constructor-like create = 0x1001b7c3c
Resolve                 = 0x1001b7d14
vtable                  ≈ 0x1010ef148
response/state methods  = 0x1001b80cc / 0x1001b80e4 / 0x1001b8124
```

构造流程明确：

```text
operator new(0x130)
-> base init 0x100231504
-> vtable = 0x1010ef148
-> +0x128 = 0
```

其 Resolve 自带 `AskClientResponseSpell`、state machine 和专用 response virtual functions，因此比 `CTriggerAction` 更适合作为新破军第二层“选择目标牌 / 接收 MoveCard 回包”的 child action 容器。

推荐结构：

```text
v31 first TriggerAction Confirm
-> 创建/入队专用第二层 CMoveCardAction child
-> child 自己发选牌 UI、等待回包
-> child 处理选中牌
-> 再进入扣置与回合结束返还
```

在完整还原 child action 创建/入队字段前，不生成新的“直接 Confirm→UI”猜测版。

### 点将问题：v12b/v13 的“selection_stable”并非真正干净基线

用户最新反馈：

```text
首次进入时可能无法选择界徐盛；
选其他将开局再退出后，重新进入可能缺将；
曾观察到吴国火包将整批消失（需继续真机复核）。
```

重新比较稳定总基线、v12b/v13、v31/v32 后确认：

- v31/v32 的 `game.lib` 与 v12b/v13 完全相同；
- 所以该问题不是 v31/v32 新增 native prompt patch 引起；
- v12b 只是移除了 174/491 的 `default="0"`，并没有撤销更早的点将 Lua 补丁。

历史 `game.lib` 继承过以下全局点将修改：

- ConfigLoadManager 强制注入/覆盖 174/491 runtime config；
- DjWarriorsNew 白名单和扩展包 bitmask 绕过；
- PackageManager:hasChr 白名单；
- SgsApp / TableScene / TeamScene 点将缓存与自动点将校验白名单。

这些逻辑直接涉及 package bitmask / cache / ownership，和“进入一局再退出后整批武将列表变化”的状态型症状高度相关。

因此以后不要仅因为文件名为 `selection_stable` 就把它视为已证明的点将稳定实现。

### v33a clean-selection 诊断基线

生成：

```text
sgs_jiexusheng_v31_clean_selection_v33a.ipa
```

构建原则：

```text
基于用户已验证第一层正确的 v31a；
保留 v31 Mach-O：第一层 prompt + v10 damage +1；
保留 174/491 XML/资源：Ex=7、491 SpellID1=11146、AssignRatio=20、CanUseDianJiangKa=1；
彻底恢复原始稳定 4.4.2.4 的 game.lib；
同步恢复 flist1 中 game.lib MD5/size；
不包含 v32 Confirm hook。
```

目的：移除所有历史全局点将缓存/包白名单 hack，让 491 尽量依靠正常配置自然进入当前可用包。

v33a 测试必须分两组：

```text
A. 点将稳定性
- fresh launch 进入点将
- 检查 491 与各扩展包武将是否齐全
- 选其他将开局 -> 退出 -> 重新点将，重复 3~5 次
- 检查是否再出现整包/整批缺将
- 重复选择 491

B. v31 第一层回归
- 其他武将出杀无新破军提示
- 界徐盛出杀有确认/取消
- 取消后原杀正常
```

v33a 不包含第二层修复：点“确定”回到 v31 的“无法继续/卡住”边界，但不应再出现 v32 的立即闪退。

### 新增强制工程规则

1. **“稳定版”必须按层定义。** v31 只证明第一层 prompt 稳定；不能顺带推断其 inherited game.lib 点将逻辑稳定。
2. **状态型 UI 回归优先审计缓存/package/ownership。** 若“进入一次游戏后列表改变”，不要先怀疑资源缺失。
3. **对历史实验基线做真正文件 diff。** 不只看最新 native patch；必须审计 inherited game.lib/XML 修改。
4. **恢复原版优于继续加白名单。** 当正确 XML 配置足够时，优先删除全局 runtime whitelist/cache bypass。
5. **一个 action 只承担其设计的 response 生命周期。** TriggerAction、MoveCardAction、DamageAction 不因字段相似就可互换。
6. **第二层 UI 必须有能消费其回包的 child action。** 只让客户端窗口出现而 native 没有 receiver，不算完成。

## 2026-09-04 v33a → v33b：点将问题按代码证据重新拆层

### 用户观察修正

用户重新确认此前“吴国火包将缺失”属于观察误差。以后不得把该现象作为点将根因证据。

**强制方法论：用户反馈用于定义复现现象和验收标准；根因判断必须优先依赖真实代码、二进制/module diff、状态机与数据流。** 若用户观察与代码证据冲突，以代码为主并继续设计可区分假设的测试。

### v33a 为什么曹冲/界徐盛彻底消失

v33a 将 `game.lib` 完全恢复到原始稳定 4.4.2.4。真机结果：174 曹冲、491 界徐盛均不再出现在点将列表。

重新解密并比较 `stable_game.zip` 与 v31 `game.lib`，确认 v31 相对原始客户端一共只修改 6 个 LuaJIT 模块：

```text
app.data.manager.ConfigLoadManager
app.ui.dj.DjWarriorsNew
app.data.manager.PackageManager
app.SgsApp
app.scenes.TableScene
app.scenes.TeamScene
```

逐模块 source-string diff 证明：

- `PackageManager`：仅新增 single-mode `id==174/491 -> hasChr=true` 白名单；
- `SgsApp / TableScene / TeamScene`：仅把原 Ex bitmask 条件扩成 `原条件 OR cid==174 OR cid==491`；
- `DjWarriorsNew`：仅对白名单绕过 Ex bitmask / `assignratio<=0` 等条件；
- **`ConfigLoadManager` 是唯一真正向 runtime config 注入 174/491/441/442/11146 的模块**，同步：
  - `characterdata / sgs.CharacterData`
  - `spelldata / sgs.SpellData`
  - `chrCfg`
  - `chrSelect`
  - `singleChrConfig`

因此 v33a 的消失根因不是“资源没了”或“某一包武将异常”，而是：**外部 XML 中虽有 174/491，但旧客户端运行时 config 表没有这两个后加 ID；移除 ConfigLoadManager runtime injection 后，DjWarriorsNew 没有完整 record 可枚举。**

### 为什么不应恢复其余五个 whitelist/cache hack

当前规范配置已经把 174/491 对齐到正常可用山包接口：

```text
ExType = 7
AssignRatio = 20
CanUseDianJiangKa = 1
```

正向对照：稳定版正常可用山包武将 50/51/52/53/54/55/57 全部 `Ex=7`。

原版 `DjWarriorsNew` 单机路径本身会：

```text
要求 chrSelect.CanUseDianJiangKa != 0
要求 ExType 位于 self.gameEx bitmask
single-mode 默认 bObtained=true
但 assignratio<=0 会把 bObtained 重新置 false
```

因此只要 runtime records 存在且配置为 `Ex=7 / AssignRatio=20`，174/491 应走正常山包条件，不需要额外 `id==174/491` 的 package/cache/scene bypass。

### v33b minimal-runtime 设计

生成：

```text
sgs_jiexusheng_v31_min_runtime_v33b.ipa
```

构建原则：

```text
以 v33a/v31 已验证 Mach-O 为战斗基线；
以原始稳定 stable_game.zip 为 game.lib 基线；
仅替换 app.data.manager.ConfigLoadManager 为 v31 runtime-injection 版本；
其余 4361 个 game.lib 模块全部保持原始稳定版；
不恢复 DjWarriorsNew / PackageManager / SgsApp / TableScene / TeamScene 的 174/491 白名单；
不带入 v32 Confirm hook。
```

静态验证：

```text
decrypted game.lib entries = 4362
changed modules vs original stable_game.zip = 1
only changed module = app.data.manager.ConfigLoadManager
```

Mach-O 与 v31 完全相同：

```text
executable SHA256 = ca2321010ae1278e1b9e683c26147dee68a5120e7d7954d7f40a442391b26a2a
0x235534 registration retained
0x235e6c v31 candidate hook retained
0x306388 v10 damage hook retained
0x2371cc remains original v31 instruction; v32 hook absent
```

### 新的强制点将排查顺序

以后新增武将“看不见/选不出/状态变化”时必须拆成以下层级逐层证明：

```text
1. external XML/resource 是否存在
2. runtime characterdata / chrCfg / chrSelect / singleChrConfig 是否存在且一致
3. Ex/package bitmask 是否自然通过
4. AssignRatio / CanUseDianJiangKa 是否自然通过
5. 只有上述正常配置仍失败，才考虑 PackageManager / Scene / cache 白名单
```

禁止一开始就在 `PackageManager / SgsApp / TableScene / TeamScene / DjWarriorsNew` 全局增加 ID 白名单，因为这会扩大状态缓存与扩展包逻辑的影响范围，也会让后续根因难以区分。

## 2026-09-04 强制规则：用户现象先对照当前项目事实，再判定是否为 bug

这是本项目后续排查的强制第一步，不是可选建议。

当用户描述任何运行时现象，例如：

```text
某个武将消失
某个国家/扩展包没有武将
某按钮没有出现
某技能没有触发
某一批武将进入/退出对局后发生变化
```

**不要先把用户描述直接转化成 bug 假设。** 必须先对照当前项目的实际代码、配置、稳定基线和已知禁用项，回答两个问题：

```text
A. 这个现象在当前项目里是否本来就是预期状态？
B. 如果不是，和稳定基线相比到底哪一层实际发生了变化？
```

优先证据顺序：

```text
1. 当前稳定基线 README / 明确的项目约束
2. character.xml / gs_character_config.xml / spell.xml 等实际配置
3. 当前 game.lib 对应模块和 runtime table
4. Mach-O / native 状态机和二进制 diff
5. 真机现象用于验证上述代码结论，而不是替代代码结论
```

### 本次反例："吴国火包将都没了"

用户曾观察到：

```text
吴国似乎没有火包将
```

当时错误做法是把它作为“点将缓存可能导致整批火包武将消失”的证据继续推理。

正确做法应该先检查项目稳定基线。当前 README 明确规定：

```text
太史慈正确 ID = 38
太史慈当前禁用
不要重新启用之前的天义实验补丁
```

在本项目当前点将基线中，太史慈 38 被刻意禁用，因此**吴国没有可见火包将本身是预期状态**，不应进入 bug 证据链。

由此形成强制规则：

> 用户报告的现象首先是“待核对事实”，不是“已确认异常”。先查项目中该对象是否本来存在、是否启用、是否属于当前包、是否被明确隐藏/禁用；只有与当前稳定基线不一致时，才把它升级为 bug 线索。

### 对用户反馈的正确使用方式

用户反馈仍然非常重要，但职责是：

```text
反馈 -> 指出需要核对的现象和复现路径
代码/配置 -> 判断现象是否异常，并确定根因
真机测试 -> 验证修复是否符合代码预期
```

不要变成：

```text
反馈 -> 直接假设根因 -> 围绕假设修改代码
```

尤其涉及“某个包/国家/武将缺失”时，必须先检查：

```text
- 该武将在当前版本是否实际存在
- 当前项目是否明确禁用/隐藏
- Ex/package 是否属于当前可用包
- point-general / ownership / AI 配置是否允许显示
- 稳定基线中是否本来就没有该项
```

只有完成这一步，才能进入 runtime config、cache、package bitmask、native patch 等后续排查。

## 2026-09-04 维护纠正：v33b minimal-runtime 失败，恢复 v31 完整点将链

### v33b 真机结果

用户真机反馈：`sgs_jiexusheng_v31_min_runtime_v33b.ipa` 中曹冲 174 与界徐盛 491 仍然不出现在点将界面。

因此以下旧判断被否定：

```text
只恢复 ConfigLoadManager runtime records，
依靠 Ex=7 / AssignRatio=20 / CanUseDianJiangKa=1，
即可让 174/491 自然进入原版 DjWarriorsNew 点将链。
```

不得再把这个判断当作已验证结论。

### 二进制事实：v31a vs v33b 只有两个文件不同

逐 ZIP-entry 对比：

```text
entry count: 1687 vs 1687
entry names: completely identical
different files only:
  res/hotfix2/game.lib
  res/flists/flist1
```

Mach-O、character.xml、spell.xml、gs_character_config.xml、174/491 资源均相同。

所以 v33b 中两个武将消失的根因确定属于**被裁剪的 game.lib 点将/拥有/缓存链**，不是 native combat patch、XML 或资源。

### v33c 处理原则

生成：

```text
sgs_jiexusheng_v31_selection_restored_v33c.ipa
```

v33c 直接恢复已真机验证成功的 v31a 完整 `game.lib` 与匹配 `flist1`；因此内容上 byte-for-byte 等同 v31a。

这是有意的回退，不是新的推断性 patch：

- 恢复曹冲/界徐盛已知可见/可选的事实基线；
- 保留 v31 已验证第一层新破军；
- 保留 v10 damage +1；
- 不包含 v32 Confirm 闪退 hook。

### 点将模块精简的新增硬规则

如果以后还要确定 `ConfigLoadManager / DjWarriorsNew / PackageManager / SgsApp / TableScene / TeamScene` 中哪些是真正必要模块：

1. 从**已真机成功的 v31 game.lib**开始；
2. 一次只恢复/删除一个模块或一个明确条件；
3. 每一步都必须真机验证“fresh launch 可见 + 可选 + 开局退出后再次可见”；
4. 没有对应 A/B 真机结果之前，不得根据 Ex/AssignRatio 的静态条件推断某个 whitelist/cache 模块“不必要”；
5. 用户反馈只决定待检查现象；最终结论必须同时有项目事实、代码/二进制 diff 和真机 A/B 支撑。


## 2026-09-04 v34：从 v31a 出发的点将显示根因 + 独立 child 面板实验

### 用户语义澄清

用户所说“曹冲/界徐盛无法选”在当前问题中指：

```text
174 / 491 有时根本不显示在“选择更多武将”的点将界面
```

不是“已经显示但点击后无法确认”。以后必须先区分：

```text
A. 不显示 / 被列表过滤
B. 显示但不可点
C. 点击后请求失败
D. 选出后进入游戏失败
```

不能把四种问题混成“选将失败”。

### 点将显示的实际项目链路

直接检查 v31a `app.ui.SelectCharacter`：

```lua
local chrlist = {}
for k,v in pairs(self.dealCharacter) do
    chrlist[#chrlist + 1] = v.uCharacterId
end
local djPanel = DjDlg.new({gameEx = app.sgCurModePkg, banChrData = chrlist})
```

也就是说，打开“选择更多武将”时，**当前开局已经发到初始候选区 `self.dealCharacter` 的所有武将都会进入 `banChrData`**。

随后 v31a `app.ui.dj.DjWarriorsNew:initContent` 明确：

```lua
... and not self:isInBanChrList(chr.id)
```

且 single-mode 的 `bObtained` 分支还会再次检查 `banChrData`。

因此：

```text
某武将进入初始随机候选 dealCharacter
-> 被加入 banChrData
-> “选择更多武将”列表故意不再显示它
```

这不是 cache 猜测，而是当前项目代码的直接数据流。

### 为什么 v31a 的 174/491 会偶发不显示

v31a 的 ConfigLoadManager 和外部 `gs_character_config.xml` 都把：

```text
174 AssignRatio = 20
491 AssignRatio = 20
```

但项目原始目标明确是：

```text
曹冲 / 界徐盛只允许玩家点将
不加入 AI / random 初始选将池
```

v31a 同时已经在 `DjWarriorsNew` 中对白名单 174/491 做了：

```lua
if chr.assignratio <= 0 and chr.id ~= 174 and chr.id ~= 491 then
    bObtained = false
end
```

所以正确的源头修复应是：

```text
174 / 491 AssignRatio 恢复为 0
保留 CanUseDianJiangKa = 1
保留 v31a 的 174/491 点将显示特例
```

而不是删除 v31a 的点将模块，也不是绕过 `banChrData`。

这样做的语义正好是：

```text
不进入初始随机 dealCharacter
-> 不会因为自己已经在 dealCharacter 而被 banChrData 隐藏
-> 仍可通过“选择更多武将”点将
```

### v34a：selection-only 隔离版（待真机验证）

文件：

```text
sgs_jiexusheng_v31_selection_ratio0_v34a.ipa
SHA256 d5d159baae838a872a7f04d0690db32354e93dbb14d51f82fb4a2c6e4f0d59e5
```

严格从用户已验证的 v31a 出发：

```text
Mach-O 与 v31a byte-for-byte 相同
只改 174/491：
  gs_character_config.xml AssignRatio 20 -> 0
  ConfigLoadManager chrUseConfig AssignRatio 20 -> 0
  ConfigLoadManager singleChrConfig assignratio 20 -> 0
其他点将模块完全保留 v31a
```

解密后 game.lib 内容 diff 只有：

```text
app.data.manager.ConfigLoadManager
```

这版用于单独验证点将显示稳定性，不包含第二层面板 patch。

### 第二层架构：从正常 CastSpell registry 创建独立 child

v32 已证明禁止：

```text
CTriggerAction 第一层 Confirm
-> 同一个 CTriggerAction 再 AskClientResponseSpell
```

正确旧引擎生命周期应是：

```text
CTriggerAction Confirm
-> 正常 NetMsgUseSpellRpy / CastSpell
-> spell registry / factory
-> 创建独立 child action
-> child 自己 Resolve / wait / response
```

普通 PoJun 414 是当前版本最接近的生命周期容器，已确认：

```text
factory/create-like = 0x1001879c0
Resolve             = 0x100187adc
AskClientResponseSpell call = 0x100187cc8
response handlers   = 0x100187d9c / 0x100187f58
```

PoJun Resolve 中当前参数直接来自 child 自己：

```text
source seat = child source role / +0x90
skill id    = child +0x88 object id
 target seat = child +0x128 role +0x1c4
```

原 414 调：

```text
AskClientResponseSpell(... spellId=414-ish, targetSeat, sourceSeat, optType=24 ...)
```

客户端 `OnRoleOptTargetLogic` 已确认对：

```text
spellId = 11146
optType = 28 / OPT_SKILL_FLAG1
```

会直接：

```text
push TYPE_SPELL_EFFECT action
-> UISelCardDlg.new(rep)
-> PoJunNew hand/equip 选择
-> 上限 = target current HP
```

### v34b：独立 PoJun child 的 panel-only 实验（待真机验证）

文件：

```text
sgs_jiexusheng_v31_selection_panel_v34b.ipa
SHA256 dc34d1a61e525c76390feb565a6db02c134d36539e9ed8317ffdc9ec56084988
```

先包含 v34a 的选将 ratio=0 修复。

Native 额外 patch：

```text
0x1002e0df8 -> cave 0x100002c00
  只在 CastSpell internal dispatcher 中：11146 lookup -> 414
  目的是让已有 registry/factory 创建合法 PoJun child action

0x100187cac -> cave 0x100002c40
  PoJun child 即将 AskClientResponseSpell 时：
  若 sourceRole.HasSpellId(11146)：
    spellId = 11146
    optType = 28
    source/target/timeout 继续用 child 已有原生上下文
  否则回原 414 optType=24 路径

0x100187d9c -> cave 0x100002d00
0x100187f58 -> cave 0x100002d80
  若 sourceRole.HasSpellId(11146)：
  第二层 reply 直接 state=4 / finish
  禁止进入旧 PoJun 414 的后续效果
  非 11146 则 replay 原路径
```

这里**只借 414 的 action 生命周期/容器，不借其技能语义**。禁止把这写成“使用 414 实现新破军”。

v34b 保留：

```text
v31 registration 0x235534
v31 candidate ownership 0x235e6c / 0x2b00
v10 damage 0x306388
```

且明确没有 v32 的：

```text
NetMsgUseSpellRpy direct AskClientResponseSpell hook
```

v34b 当前里程碑只要求：

```text
点第一层“确定”
-> 不立即闪退
-> 出现官方 11146 目标手牌/装备 UISelCardDlg
```

暂未实现：

```text
真正扣置所选牌
REMOVE/武将牌旁状态
回合结束返还
```

为了防止误执行旧 414，v34b 对 11146 的第二层回复目前直接结束 child。

### 新增强制原则：随机池与点将池必须分开

以后新增“只允许玩家点将”的武将，必须检查：

```text
1. AssignRatio / random weight 必须保持 0
2. CanUseDianJiangKa 必须允许
3. 点将 UI 若用 assignratio 判断 obtained，需要仅对目标 ID 做显示例外
4. 不得为了“让它显示”把随机权重设成正数
5. 必须检查 SelectCharacter 当前候选是否会被转换为 DjDlg banChrData
```

否则会出现典型假象：

```text
为了让武将在点将 UI 显示而把 AssignRatio 提高
-> 它反而进入初始随机候选
-> 初始候选被 banChrData 排除
-> 点将 UI 又偶发看不到它
```

## 2026-09-04 维护记录：v34 实测与 v35 narrow factory lookup

### v34a / v34b 真机结果

用户真机验证：

```text
v34a/v34b 点将：
- 曹冲 174、界徐盛 491 现在整体可正常显示；此前“偶发不显示”的主要问题已解决。
- 极少数情况下二者会“显示出来但点不出来”。这是独立的低优先级确认/换将链问题；后续单独分析，不要为了它重新修改列表构建或其他武将逻辑。

v34b 新破军：
- 第一层 Confirm 后不再像 v32 一样闪退。
- 但官方目标手牌/装备选择面板仍未出现。
```

因此可以冻结两条事实：

1. v34a 的 `AssignRatio=0 + v31 174/491 点将白名单` 方向有效；当前不要再动其他武将的点将列表逻辑。
2. v34b 的“独立 child action”架构比 v32 正确，因为 Confirm 不再崩；当前失败点发生在 child 创建/registry/factory 接线，而不是再次回到 `CTriggerAction` 二次等待问题。

### v34b 面板未出现的代码根因

v34b 在 `0x1002e0df8`（CastSpell dispatcher 入口）直接把：

```text
11146 -> 414
```

这一步发生得太早。

重新反汇编证明旧引擎把以下概念分开：

```text
logical skill identity / ownership / metadata
vs.
registry key / factory implementation
```

关键函数：

```text
sub_1002e2910 = 0x1002e2910
```

其流程明确：

```text
x1 skillId
-> w20 = skillId
-> registry tree 按 w20 搜索
-> 找到 entry
-> 后续 0x10017a4f8(metadata lookup) 仍使用同一个 w20
```

而真正 child 创建函数：

```text
sub_1002e152c = 0x1002e152c
```

也独立执行：

```text
skillId -> registry tree lookup -> metadata -> stored factory callable -> child action
```

所以 v34b 把整个 dispatcher 的逻辑 skillId 变成 414，会让 eligibility/metadata 也变成旧 PoJun 414。界徐盛真正拥有的是 11146，因此 request 可以被消费而不崩，但没有正确创建 11146 child，表现就是：

```text
Confirm 不闪退
但没有第二层面板
```

### 强制新原则：借 factory，不借 skill identity

后续所有“借旧 native action 容器承载新技能”必须遵循：

```text
新技能真实 ID / HasSpellId / metadata / protocol identity = 保持新 ID
旧技能 registry/factory key = 仅在 lookup 的极小窗口临时使用
找到 factory entry 后必须立即恢复新 ID
```

禁止再做：

```text
在 CastSpell / dispatcher 入口直接把新 ID 全局改成旧 ID
```

这会污染后续 ownership、metadata、状态和协议语义。

### v35a：narrow registry lookup bridge

基线：

```text
sgs_jiexusheng_v31_selection_ratio0_v34a.ipa
```

注意：v35a 不是基于 v34b 继续叠 dispatcher hack；它以 v34a 为底，只重新加入经过修正的 native 第二层桥。

#### A. 通用 registry eligibility helper

```text
0x1002e2928 -> 0x100002e00
```

逻辑：

```text
originalId = skillId
lookupId = (skillId == 11146) ? 414 : skillId
只用 lookupId 搜 registry tree
```

找到 entry 后：

```text
0x1002e296c -> 0x100002e40
恢复 logical skillId = originalId
再继续原 validator / metadata 流程
```

#### B. 实际 child-factory lookup

```text
0x1002e1584 -> 0x100002e80
```

同样只在 registry tree lookup 期间：

```text
11146 临时 lookup as 414
```

找到 factory entry 后：

```text
0x1002e15c0 -> 0x100002ec0
恢复 11146
再执行原 0x10016a354 / 0x10017a4f8 metadata 流程
```

因此期望实际语义为：

```text
first-level Confirm 11146
-> validator 看见 logical 11146，但 registry 查找借 414 entry
-> factory creation 看见 logical 11146，但 constructor/vtable 借 414 factory
-> 得到独立 PoJun child
-> child source 仍真实拥有 11146
-> child Resolve 发送 spellId=11146 / optType=28 官方卡牌面板
```

#### C. child panel / reply 路径

继续使用 v34b 已经证明“不立即闪退”的独立 child Resolve 结构：

```text
0x100187cac -> cave 0x100002c40
```

11146 source 时发送：

```text
spellId = 11146
optType = 28
target = child 当前目标
optSeat / spellCasterSeat = 界徐盛 seat
```

旧 414 response effect 继续只对 11146 分支安全终止，不执行翻面/摸牌：

```text
0x100187d9c -> 0x100002d00
0x100187f58 -> 0x100002d80
```

#### D. v34b 旧错误 hook 已移除

必须确认：

```text
0x1002e0df8 = ff8302d1
```

即恢复原始：

```text
sub sp, sp, #0xa0
```

不得存在 dispatcher-entry 11146→414 映射。

### v35a 测试边界

当前只验证：

```text
第一层 Confirm
-> 正常 CastSpell lifecycle
-> 独立 child 创建
-> 官方 PoJunNew 目标手牌/装备选择面板出现
```

若面板出现，检查：

```text
目标 = 当前 Sha target
最大选牌数 = 目标当前体力
可选区域 = 手牌 + 装备
```

本版仍未实现：

```text
选中牌真正移到 REMOVE / 武将牌旁
回合结束返还
```

### 当前点将问题优先级

174/491 极少数“显示但点不出来”暂时记录，不与第二层面板混修。

后续若单独处理，必须从：

```text
DjItemWarrior click
-> DjChrIdChanged
-> panelWarriors.curSelectedChrId
-> DjDlg close / ClientGmCmdReq cmdId=2
```

确认链做代码追踪；不要再修改 `AssignRatio`、列表枚举或其他武将 package/filter，除非代码证明问题仍在显示链。


## 2026-09-04 维护记录：v35 真机无面板；v36 construction metadata bridge

### v35a 真机结果

用户真机验证：

```text
选将正常（与 v34a 一致）
其他武将出杀正常
界徐盛第一层确认/取消正常
取消后杀/闪正常
点确定不闪退
但确定后仍然没有官方手牌/装备选择面板
```

因此继续冻结：

```text
v34a 点将列表方向
v31 第一层 trigger/prompt
v10 damage +1
独立 child action 作为第二层架构
```

不得再回到 v32 的同一 `CTriggerAction` 二次等待。

### v35 新发现：factory entry 后还有 native metadata 硬门槛

`sub_1002e152c` 的真实 child 创建路径在找到 registry entry 后执行：

```text
0x1002e15c0  bl  0x10016a354
0x1002e15c4  mov x1,x20
0x1002e15c8  bl  0x10017a4f8   # native metadata lookup
0x1002e15cc  cbz x0, fail      # metadata NULL => 不创建 child
```

v35 的设计错误是：

```text
registry lookup 临时 11146 -> 414
找到 factory 后恢复 11146
metadata lookup(11146)
```

但旧 iOS native 本来没有完整 11146 原生注册。对 v31 原生 executable 的静态交叉检查：

```text
32-bit literal 11146: 0 个
32-bit literal 414:   28 个
```

这与已知 `CreateXSpell<PoJun,414>` 存在、`PoJunNew,11146` 不存在一致。

因此 v35 很可能是：

```text
找到 414 factory entry
-> metadata(11146) == NULL
-> 0x1002e15cc 直接 fail
-> child 根本没有创建
-> Confirm 不崩，但也没有面板
```

### v36a：construction metadata 也借 414

基线：

```text
sgs_jiexusheng_v31_selection_panel_lookup_v35a.ipa
```

只新增一个 hook：

```text
0x1002e15c4 -> cave 0x100002f00
```

语义：

```text
if logical skillId == 11146:
    NativeMetadataLookup(414)
else:
    NativeMetadataLookup(original skillId)
```

注意这里借 414 metadata 的范围仅限**旧 PoJun child 的 construction contract**。

仍保持：

```text
第一层 request / source ownership = 11146
HasSpellId(source,11146)
第二层客户端请求 spellId = 11146
第二层 optType = 28
旧 414 翻面/摸牌 response 对 11146 分支继续禁止
```

因此新工程规则进一步细化为：

```text
借旧 native action 容器时：
- protocol identity 不借
- source ownership 不借
- 客户端 spellId 不借
- 但 factory 构造所必需的 registry entry / native construction metadata 可以一起借，前提是后续旧规则效果被明确隔离
```

### v36a 测试边界

只验证：

```text
第一层 Confirm
-> child 实际构造
-> child Resolve
-> 官方 PoJunNew 手牌/装备面板
```

若 v36a 仍然无面板，禁止继续盲改 registry/metadata/optType；下一步必须证明：

```text
factory callable 是否返回 non-null child
child 是否进入 action queue
child Resolve 是否真正被调用
```

然后只在第一个断点失败处修复。

## 2026-09-04 维护更新：停止 414 factory 猜测路线，按真实 native action 合同实现第二层

### v36a 真机结果

用户真机验证：

```text
- v34a/v35/v36 的选将整体已经 OK；
- 极少数情况下 174/491 已显示但点击后没有真正选出，暂列低优先级确认链问题；
- 界徐盛第一层新破军 Confirm/Cancel 正常；
- Cancel 后原 Sha 正常继续；
- Confirm 已不会像 v32 那样闪退；
- v34b / v35a / v36a Confirm 后均没有出现官方手牌/装备选择面板。
```

因此 v34-v36 的 `11146 -> 414 factory/metadata` 路线不得继续堆 patch。连续真机结果已经说明：只借 414 registry/factory/metadata 并不能得到 PoJunNew 第二层。

### 414 的项目级纠正

重新以官方 Android native `libgame.so` 为参考后确认：普通 `PoJun 414` 自己就是和旧破军规则绑定的专用状态机，而不是一个纯通用的“选目标牌 child action 容器”。

因此：

```text
可以参考 414 的 CSpell/CAction 生命周期形式；
不能把 414 action 本身改 ID 后当作 11146 的第二层；
不能继续通过提前/局部映射 11146 -> 414 来期待官方 PoJunNew 面板。
```

### 第二层必须遵守的真实旧引擎合同

第一层 `CTriggerAction::NetMsgUseSpellRpy` 后的项目结构应理解为：

```text
CTriggerAction::NetMsgUseSpellRpy
  -> CastAsSpell
  -> CastSpell
  -> 创建独立 CSpell-derived action
  -> CActionMgr::PushAction
  -> 第一层 action 清除 waiting / 推进
  -> child::Resolve
  -> child::AskClientResponseSpell
  -> child 自己接收对应回包
  -> ClearAllOfWaitingOpt / SetResolveStep / SetOverMark
```

第二层不能再复用第一层 `CTriggerAction`，也不能只发一个 UI packet 而没有对应 native receiver。

目前已还原的 iOS `CreateSpell`/`CSpell` 关键字段语义包括：

```text
+0x70  game
+0x88  CCardSpellData*
+0x90  source CRole*
+0x98  card vector
+0xe0  target-seat vector
+0xf8  parent CAction*
```

后续 dedicated 11146 child 必须按旧引擎自己的方式初始化这些字段并进入 `CActionMgr::PushAction`，不能手工拼一个不完整对象。

### 原生参考样本的分工：只借框架，不借规则

后续研究 11146 第二层时，优先对照以下现有 native 技能，但任何一个都只能作为结构参考：

#### CiXiongShuangGuJian

主要用于理解：

```text
- 独立 CSpell action 的 Resolve/等待/回包生命周期；
- 一次 MsgMoveCard 回包携带多个 cardId 的解析；
- cardId 合法性验证；
- ClearAllOfWaitingOpt -> SetResolveStep/SetOverMark。
```

已定位过的 iOS 关键点：

```text
Resolve            = 0x10029a914
Cancel reply       = 0x10029ad34
NetMsgMoveCardRpy  = 0x10029adc0
vtable             ≈ 0x1010f8848
object size        = 0x130
```

其技能效果、触发条件、性别逻辑等一律不能带入新破军。

#### HanBingJian

主要用于理解：

```text
- 从目标 hand/equip 区域验证被选择的牌仍合法；
- 目标区域/牌对象校验；
- 选牌回包后的状态推进。
```

不要走寒冰剑自己的弃牌次数/装备技能路线。

#### ShunShouQianYang / GuoHeChaiQiao

主要用于理解“从另一个角色区域选择牌”的成熟 action/MoveCard 结构；只看 target-zone、MoveCard 和 action 生命周期，不复用锦囊规则。

### 新增重点参考：MengJin / 猛进（ID 100）

用户明确建议：后续第二层也可以参考猛进，但不能走猛进原路。已根据实际项目核对：

客户端 4.4.2.4 自带：

```text
app.gamelogic.spell.huo.MengJin
SpellId.SKILL_CHARACTER_MENG_JING = 100
```

其 Lua 客户端从当前 trigger action 的 `targetSeatId` 读取目标，并通过 `PubGsCUseSpell` 发第一层请求。

更重要的是，iOS 4.4.2.4 native binary 明确存在：

```text
CreateXSpell<MengJin,100>
```

官方 Android native `libgame.so` 进一步保留完整符号：

```text
CreateXSpell<MengJin,100>::CreateInstance
MengJin::CanTriggerMe
MengJin::Resolve
MengJin::NetMsgMoveCardRpy
MengJin::TimeOutCallBack
```

Android 官方 `MengJin::Resolve` / `NetMsgMoveCardRpy` 已反汇编确认其真实链路为：

```text
CanTriggerMe
  -> 从 parent Sha/相关 action 验证技能来源与目标
  -> 检查目标存在可处理的 hand/equip 牌

Resolve
  -> 从 parent action 取得当前目标
  -> 保存 target role
  -> SetResolveStep
  -> AskClientResponseSpell
  -> 等待目标牌选择

NetMsgMoveCardRpy
  -> 检查 MoveCard reply 格式
  -> 从目标区域 Find(cardId)
  -> 保存被选择 card
  -> ClearAllOfWaitingOpt
  -> SetResolveStep

下一 Resolve state
  -> 调 CMoveCardAction::MoveCards
  -> 推进/结束 action

TimeOutCallBack
  -> 在合法目标牌中选择 fallback
  -> 继续同一 action state machine
```

因此猛进对新破军最有价值的参考点是：

```text
1. 如何从 parent Sha/action 获取真实 target；
2. 如何让一个独立 CSpell action 自己持有 target/card 状态；
3. AskClientResponseSpell -> NetMsgMoveCardRpy 的完整等待生命周期；
4. 回包后如何验证目标区域中的 card；
5. 如何调用 CMoveCardAction::MoveCards 进入真正的 native 移牌链；
6. timeout / invalid reply 如何 fail-safe，而不是破坏 parent Sha。
```

禁止直接复制的猛进语义：

```text
- 猛进自己的触发条件；
- optType / 单张牌数量；
- 弃置/移动目的区域；
- timeout 随机选牌规则；
- Resolve step 数量与具体效果；
- 任何“猛进技能 ID 100 作为 11146 carrier”的做法。
```

正确使用方式是：**把 MengJin 当作当前 4.4.2.4 引擎里“Sha parent -> target card selection -> MoveCard reply -> native MoveCards”的完整教学样本，再按 PoJunNew 11146 自己的协议、数量上限、区域和返还规则实现 dedicated child。**

目前只确认 iOS 存在 `CreateXSpell<MengJin,100>`；iOS `MengJin` 的精确 Resolve/vtable/response 地址尚未完成映射。后续如果使用，必须从当前 v34a/v31 Mach-O 实际定位，不得根据 Android 地址猜 iOS 地址。

### 下一版实现前的强制证明项

在生成下一版 IPA 前，至少先证明：

```text
1. dedicated 11146 child 的构造/初始化方式；
2. child 如何被 PushAction；
3. child 如何从 parent Sha 得到 source + target；
4. child Resolve 实际发送 GsCRoleOptTargetNtf / AskClientResponseSpell(11146, optType=28)；
5. PubGsCMoveCard 的回包由同一个 child 接收；
6. selected card IDs 的数量和 hand/equip 合法性验证；
7. 本里程碑先安全结束 child，不执行 414/CiXiong/HanBing/MengJin 的旧技能效果。
```

在这些点闭环前，不再通过 registry ID、metadata ID、optType 数字的试错来生成 IPA。

## 2026-09-04 v37：停止 registry 猜测，按真实 ActionMgr 合同创建 11146 dedicated child

### 用户要求
用户明确要求后续不要再“猜测着做”，必须像此前第一层 v31 一样，先真正理解项目结构，再加入第二层。

### 已废弃路线
以下路线正式停止：

```text
11146 -> 414 registry/factory/metadata 映射
```

原因：旧 PoJun 414 是规则绑定 action，不是通用 UI 容器；v34b/v35a/v36a 连续真机结果均为前三项正常但 Confirm 后不出现面板。

### 新的真实项目理解
已精确对齐 iOS 4.4.2.4 与 Android 官方具名实现：

```text
CTriggerAction::NetMsgUseSpellRpy
  -> CastSpell
  -> Create CSpell-derived child
  -> CActionMgr::PushAction(game+0xb0)
  -> parent trigger cleanup/progression
  -> child::Resolve
  -> AskClientResponseSpell
  -> child 自己接收 MoveCard reply
```

其中 `CActionMgr::PushAction = 0x1001e0768`。

### MengJin / 猛进精确 iOS 映射
猛进只作为“Sha parent -> target -> independent child -> Ask -> MoveCard reply -> MoveCards”的结构样本，不作为 11146 carrier：

```text
CreateInstance = 0x10026ab4c
Resolve        = 0x10026ac8c
MoveCard reply = 0x10026afec
object size    = 0x140
```

其对象扩展字段已确认：

```text
+0x128 target role
+0x130 selected card pointer
+0x138 MengJin state flag
```

v37 使用对象末尾 padding `+0x13f` 作为 synthetic marker `0xA7`，普通猛进对象不会命中 synthetic 分支。

### v37a 设计
基线固定为用户已验证选将基本正常的：

```text
sgs_jiexusheng_v31_selection_ratio0_v34a.ipa
```

不修改 `game.lib`、flist、XML、点将逻辑或其他武将。

第一层 Confirm 后，如果旧 native `CastSpell(11146)` 失败：

```text
if spellId != 11146:
    保持原失败路径
else:
    获取 ID100 的 CCardSpellData，仅用于已有 CSpell constructor/vtable
    MengJin::CreateInstance(metadata100, game)
    child.source = Jie source role
    child.parent = current Sha
    child.synthetic_marker = 0xA7
    PushAction(game+0xb0, child)
    向 TriggerAction 返回 0x15，按正常 CastSpell 成功路径 cleanup/progress
```

注意：这不是让界徐盛拥有或施放猛进 100；metadata100 只承担旧引擎中已经存在的 C++ 对象构造/析构/vtable 路由能力。

### synthetic Resolve
hook `MengJin::Resolve @ 0x10026ac8c`：

- 普通猛进：重放原 `sub sp,sp,#0xc0`，跳回 `0x10026ac90`，原逻辑完全不变。
- synthetic child：
  1. 从 `child+0xf8` 的 parent Sha，通过 vtable `+0x70` 获取当前 target seat；
  2. 通过 game vtable `+0x18` 取得 target role 并写 `child+0x128`；
  3. `SetResolveStep(1)`；
  4. 调用 `AskClientResponseSpell`：

```text
casterSeat   = Jie seat
spellId      = 11146
optSeatId    = Jie seat
targetSeatId = current Sha target
optType      = 28
param        = 0
```

目标是直接命中客户端已有：

```text
GsCRoleOptTargetNtf
-> OnRoleOptTargetLogic
-> spellId 11146 / OPT_SKILL_FLAG1(28)
-> UISelCardDlg(PoJunNew)
```

### synthetic MoveCard reply
hook `MengJin::NetMsgMoveCardRpy @ 0x10026afec`：

- 普通猛进：重放原首指令并回 `0x10026aff0`。
- synthetic child：当前里程碑只执行：

```text
ClearAllOfWaitingOpt
SetOverMark
```

不执行猛进弃牌，不执行 414/CiXiong/HanBing 旧规则，也还不真正扣置牌。

### v37a 真机里程碑
必须先验证：

```text
第一层 Confirm
-> independent synthetic child
-> 官方 PoJunNew 手牌/装备面板出现
-> 选择/确认后 child 安全结束
```

只有面板真实出现后，下一阶段才实现：

```text
多 cardId 合法性验证
-> CMoveCardAction::MoveCards
-> REMOVE/武将牌旁
-> turn-end restore
```

多卡回包格式继续参考 CiXiong；目标 hand/equip 合法性继续参考 HanBing；最终移动链参考 MengJin/顺手/过拆。任何旧技能都只能作为框架参考，不能整体承载 11146。


## 2026-09-04 v37 真机成功：第二层官方 PoJunNew 面板冻结

用户真机确认 `sgs_jiexusheng_v34a_dedicated_child_panel_v37a.ipa`：

```text
- 目标官方面板已经出现；
- 目标手牌显示正确；
- 目标装备显示正确；
- 第一层 Confirm 前三项继续正常。
```

因此必须冻结：

```text
v31 第一层 candidate/source ownership
-> v37 dedicated synthetic child
-> parent Sha target 获取
-> AskClientResponseSpell(11146,optType=28)
-> 官方 PoJunNew UISelCardDlg
```

后续不得再修改这一层来“顺手”解决扣牌/返牌；后半段只能从 MoveCard reply 之后接 native 规则。

用户再次明确：后续继续采用“先理解当前项目已有对象/状态机/数据流，再修改”的方法。最近 v31/v34/v37 的成功都建立在这种方式上；禁止退回根据症状猜地址、猜 skillId、猜 registry mapping 的试错路线。

## 2026-09-04 v38：按项目原生 RemovedZone / MoveCards / TurnEnd state 接扣置与返还

### 项目事实

`CRole` zone：

```text
+0x90  HandZone
+0xc8  EquipZone
+0x138 CRemovedFromGameZone (zone type 4)
```

七星 `QiXing` 已证明原生存在：

```text
hand(+0x90) -> removed(+0x138)
removed(+0x138) -> hand(+0x90)
```

使用 `CMoveCardAction::MoveCards @ 0x1001b317c`。调用合同已由 iOS QiXing callsite 证明：`x7=source zone`，第一个 stack arg=`destination zone`，helper 自己将 movement action 接入 ActionMgr。

### 多牌持久记录

`CRoleSpellMgr::AddSpellState @ 0x100247f78` 完整反汇编确认：每次调用都分配新的 state 节点并插链，不按 spellId 去重。因此 v38 使用“一张扣置牌一个轻量 state”。

借 `CBiFaState(212)` 的 0x28-byte trivial chassis：

```text
+0x08 spellId
+0x10 source / captured current-turn owner
+0x18 target role
+0x20 update flag
+0x24 cardId
```

创建后立即改为：

```text
spellId = 11146
source  = 当前 turn owner（不是固定 Jie；保证回合外出杀也在真实当前回合结束返还）
target  = 被杀目标
cardId  = 本条扣置牌
+0x20   = 1，避免 generic state UI update
```

这不执行 BiFa 技能规则；只借 state 对象、链表和 TurnEnd 生命周期。

### TurnEnd 生命周期

官方 Android/iOS 对照确认：

```text
CGame::TurnEnd
-> currentRole->OnTurnEnd()
-> OnEventRemoveSpellState({type=3, role=currentTurnOwner})
```

`CRoleSpellMgr::OnEventRemoveSpellState` 顺序：

```text
IsCanBeRemove(param)
-> OnRemove()
-> state destructor/unlink/delete
```

因此 v38：

- hook `CBiFaState::IsCanBeRemove @ 0x10024741c`：synthetic 11146 只在 `type=3 && param.role==state+0x10` 时返回 true；非 synthetic 重放原首指令回原函数。
- hook base `CSpellState::OnRemove @ 0x100247028`：synthetic 11146 仅把 `state+0x24` 对应 card 从 `target.removed` 原生 MoveCards 到 `target.hand`；其他 state 原函数仍为 no-op。
- 永远不扫描/清空整个 removed zone，避免误动 QiXing/BuQu/BiFa 等牌。

### v38 MoveCard reply

`MengJin::NetMsgMoveCardRpy @ 0x10026afec` 的 MsgMoveCard 布局由原函数再次确认：

```text
u16 @ +0x17
u16 @ +0x19
card IDs from +0x1b
```

PoJunNew client 同时发送 `cardCnt` 和 `dataCnt`；v38 要求二者相等、>0、<= target current HP，card IDs 无重复，且每张仍在目标 hand/equip。

合法后：

```text
hand/equip -> target.removed(+0x138)
```

每张牌通过 native `MoveCards` 入队，并建立对应 turn-end state。然后 synthetic child `ClearAllOfWaitingOpt + SetOverMark`，原 Sha 继续。

### 第二层 Cancel / Timeout

PoJunNew 面板关闭按钮发送 `CGsRoleOptRep(optType=2)`，不是 MoveCard reply。

Android vtable relocation 已证明 MengJin 继承 `CSpell::NetMsgCancelRpy`；iOS MengJin vtable slot3=`0x1002315b8`，原函数是 `ret`。v38 hook 该函数，但先检查 `vptr==MengJin vtable 0x1010f6418` 和 synthetic marker；synthetic Cancel 只 ClearWaiting+SetOver，其他 CSpell 原 no-op。

`MengJin::TimeOutCallBack @ 0x10026b110` 也加 synthetic gate：11146 超时只视为不发动并结束 child；normal MengJin 重放原首指令回原函数，绝不执行猛进随机选牌语义。

### v38a

```text
sgs_jiexusheng_v37_detain_restore_v38a.ipa
SHA256 cb71bab0cb4e1aa7ccc61807073ecbed2eebeb12c236c34637c9152699c5f268
```

相对用户已验证 v37a：1687 entries 中只有主 Mach-O 内容变化；`game.lib / flist1 / XML / 点将` byte-for-byte 不变。

新增 cave：`0x100003400..0x100003788`。

新增 hooks：

```text
0x10026afec MoveCard reply
0x10024741c turn-end IsCanBeRemove
0x100247028 state OnRemove / return
0x1002315b8 second-panel Cancel
0x10026b110 synthetic Timeout
```

### v38 真机验收

依次验证：

```text
1. v37 官方面板不回归；
2. second-panel Cancel 不扣牌，Sha 继续；
3. 1 hand -> removed；
4. 1 equip -> removed；
5. hand+equip 多选 <= HP 全部扣置；
6. 扣置后 Sha/Shan/damage 正常；
7. 当前回合结束，本次 11146 扣置牌全部回目标手牌；
8. 原装备牌也回手，不自动装备；
9. v10 +1 / 酒叠加不回归；
10. 其他 removed-zone 技能不被误清理。
```

当前尚未宣称：目标在回合结束前死亡时的官方精确牌处置；以及多牌是否需要从“每 card 一个 native MoveCards action”进一步合并为单一 mixed-zone movement event。只有真机核心路径通过后，再根据项目触发语义决定是否需要优化，不提前改动成功层。

## 2026-09-05 维护：v38 面板静止/不可选根因已由 ActionMgr + MengJin state machine 证明；v39a 修复 waiting state

### 真机事实

用户测试 `v38a`：

```text
官方 PoJunNew 面板仍可正确出现；
目标手牌与装备显示正确；
但面板下方时间条不动；
手牌/装备无法实际点选。
```

因此必须继续冻结已经验证成功的：

```text
v31 第一层 trigger
v34a 点将
v37 dedicated child
source / target recovery
spellId=11146
optType=28
官方 PoJunNew UISelCardDlg 路由
```

不要因为“面板不可操作”回退这些已经由真机证明正确的层。

### 已证明根因：synthetic Resolve 没有实现原生 state1 wait

Android 官方具名函数 `CActionMgr::ProcessAction(CGame*)` 已反汇编确认：

```text
取栈顶 action
if action+0x78 bit0(over):
    销毁 / pop
else:
    每一轮继续调用 action->Resolve()
```

**ProcessAction 不会因为 `action+0x78 bit1(waiting)` 已置位就停止调用 Resolve。**

因此“等待客户端回包”必须由具体 action 自己的 Resolve state machine 实现。

原版 iOS `MengJin::Resolve @ 0x10026ac8c` 明确读取：

```text
[action + 0x68] = resolveStep
```

并通过 4-state jump table 分发。jump-table bytes：

```text
@ 0x100fc2f35 = 00 a9 22 3f
```

其中：

```text
state 0 -> 取得 parent Sha target
           SetResolveStep(1)
           AskClientResponseSpell

state 1 -> 直接跳到 0x10026af84 epilogue
           return
           不再发送通知
           等待 MoveCard / Cancel / Timeout 回包

state 2 -> 处理已经选择的牌
state 3 -> 后续结束
```

v37/v38 synthetic Resolve 的错误是：

```text
marker==0xA7
-> 每一次 Resolve 都重新：
   target lookup
   SetResolveStep(1)
   AskClientResponseSpell(11146,optType28)
```

它完全没有读取 `resolveStep`。

所以 ActionMgr 实际会形成：

```text
Resolve -> 发第二层通知 -> waiting=1
下一 ProcessAction -> Resolve 再次执行
-> 再发同一通知 / 重置 timer / 重建 dialog
-> 循环
```

这可以同时解释真机现象：

```text
面板内容正确       -> 通知 payload / target / UI route 是对的
时间条不动         -> timeout/dialog 被持续重新初始化
牌无法稳定点选     -> dialog/input layer 被持续同类通知重建
```

因此不要把该现象误诊为：

```text
dataCnt=0
真实手牌 ID 未下发
UISelCardDlg unableSel
timeout helper 地址错误
```

这些方向目前均无代码证据支持。

### v39a 修复

生成：

```text
sgs_jiexusheng_v37_state_wait_fix_v39a.ipa
```

基线直接使用真机验证过面板内容正确的 `v37a`，**不带入 v38 未验证的扣置/返还代码**。

synthetic Resolve 增加原生 state gate：

```asm
ldr w9, [x0,#0x68]     ; resolveStep
cbz w9, state0_body    ; 只有 state0 发一次 PoJunNew 面板
ret                    ; state1+ 等待回包
```

精确 patch：

```text
0x1000030a8:
    b.eq 0x100003200

0x100003200:
    ldr w9,[x0,#0x68]
    cbz w9,0x1000030b4
    ret
```

其余 v37 panel state0 body 保持原样。

### v39a 测试目标

只验证 UI interaction，不验证扣牌：

```text
1. 第一层 Confirm/Cancel 不回归。
2. 官方 PoJunNew 面板仍正确出现。
3. 时间条开始正常倒计时。
4. 手牌牌背 / 装备能够实际点选。
5. 选中后视觉状态、剩余可选数量提示、确定按钮正常更新。
6. 确定后 child 安全结束，原【杀】继续；本版不真正扣置。
```

若 v39a 通过，冻结：

```text
TriggerAction -> dedicated child -> state0 prompt -> state1 wait -> MoveCard reply
```

然后才从 v39a 继续扣置/返还。

### PoJunNew 隐藏手牌回包：v38 parser 必须废弃

客户端 `UISelCardDlg:onSelCardsForPoJun()` 对对手未知手牌使用 cardId=0 的牌背。
确认时：

```text
cardCnt = 总选择数（包含未知手牌牌背）
dataCnt = data[] 中显式 cardId 数量
```

因此合法情况可以是：

```text
cardCnt > dataCnt
unknownHandCnt = cardCnt - dataCnt
```

v38 曾要求 `dataCnt == cardCnt`，该条件已经确定错误。

后续真正实现 native 扣置时必须：

```text
1. 验证 cardCnt <= target current HP
2. 验证 data[] 显式 ID 仍属于 target hand/equip
3. unknownHandCnt = cardCnt - dataCnt
4. 从 target HandZone 中补取 unknownHandCnt 张匿名手牌
5. 再统一进入 RemovedFromGameZone
```

不要恢复 `dataCnt == cardCnt`。

### 方法论再次确认

本次根因来自：

```text
用户现象
-> 客户端 UI 代码排除简单 selectable/data 假设
-> CActionMgr::ProcessAction 实际代码
-> MengJin::Resolve 原生 state jump table
-> 找到 state1 waiting 合同缺失
-> 最小 state-machine 修复
```

后续继续保持这种“先还原项目合同，再修改”的流程；不要根据现象猜 skillId、packet 或 UI 开关。

## 2026-09-05 维护：v39 第二层交互真机通过；重复按钮归类；v40 接真实扣置/返还

### v39 真机结果：第二层交互正式冻结
用户反馈：
- 第二层 PoJunNew 官方面板保持正确；
- 时间条正常运动；
- 目标手牌牌背与装备可以实际点击选中；
- 因此 v39 的 `resolveStep==0 才发通知 / state1 等待回包` 修复已真机验证。

从现在起冻结：
```text
first Confirm
-> dedicated synthetic child
-> target/source 恢复
-> 11146 / optType28
-> 官方 UISelCardDlg
-> 正常 timeout bar
-> 正常 touch/select
```
后续不得为扣置/返还修改这层，除非明确出现回归。

### v39 第二层确定后再次出现“确定/取消”的项目解释
用户观察：第二层选牌后点确定，会再次出现外观像第一层“是否发动破军”的 action。

不要立刻解释成 native candidate 又触发。
客户端代码证明：
```text
OnRoleOptTargetLogic(PoJunNew)
-> pushActionToSeat(TYPE_SPELL_EFFECT,11146)
-> UISelCardDlg

UISelCardDlg:onSelCardsForPoJun(confirm)
-> send PubGsCMoveCard
-> hideDlg()
-> 不 popAction()
```
而 `GameButtonLogic` 对普通 `TYPE_SPELL_EFFECT` 最终会通用执行：
```text
spell:canUse()
show Confirm = spell:canConfirm()
active Confirm = spell:canPlay()
show/active Cancel = action:canCancel()
```
因此 dialog 隐藏后，残留 `TYPE_SPELL_EFFECT(11146)` 本身就可能显示一组与第一层非常相似的确定/取消按钮。

v39 的 synthetic MoveCard reply 又只是：
```text
ClearAllOfWaitingOpt
SetOverMark
```
没有真实 MoveCards / parent Sha 后续 trigger，所以该现象在 UI-only 里程碑中属于可解释的预期残留。

强制策略：
- **不要先加全局 popAction / used flag / 禁用 candidate 来消除这个按钮。**
- 先恢复真实 `MoveCards -> ActionMgr -> parent Sha -> ask-Shan` 流程。
- 若真实扣置后该按钮仍存在，再判定是 client action-stack cleanup 还是 fresh trigger。

### PubGsCMoveCard 隐藏手牌协议：修正 v38 错误假设
PoJunNew 客户端选择对手隐藏手牌时，牌背可以是 `cardId=0`：
- `cardCnt` = 总选中张数；
- `dataCnt` = `data[]` 中带真实 ID 的张数；
- 隐藏手牌不会进入 `data[]`。

正确关系：
```text
hiddenHandCnt = cardCnt - dataCnt
```
所以 v38 的：
```text
dataCnt == cardCnt
```
是确定错误，禁止复用。

官方 `MengJin::NetMsgMoveCardRpy` 提供了 native 正向参考：
- 当客户端回包没有给出隐藏目标手牌的具体 ID 时，native 从 target HandZone 的 `CPlayCard*` vector 随机选择真实牌；
- iOS `CSgsPubFun::rand_uint` = `0x10023bd0c`；
- target HandZone vector begin/end = `target+0xa0 / target+0xa8`；
- `CPlayCard* +0x10 -> card data -> +0x00 cardId`。

`CiXiongShuangGuJian::NetMsgMoveCardRpy` 交叉证明当前 native `MsgMoveCard`：
- count at `+0x17`；
- IDs from `+0x1b`；
- u16 / stride 2。

### v40a：从 v39 接入真实扣置/返还
输出：
```text
sgs_jiexusheng_v39_detain_restore_v40a.ipa
```
SHA256：
```text
1d78315263fba2d6a0be4ef521af69dc9a84efcc87f2694234722dc72b626372
```
Main SHA256：
```text
9a51b21cf7f65112986e0819d126155e360b7c14602549b349ab183dfa7cff76
```

基线严格为用户刚验证通过交互的 v39a；相对 v39：
```text
1687 entries 名称完全一致
唯一内容变化 = 主 Mach-O
game.lib / flist1 / XML / 点将 / 资源 byte-for-byte 不变
```

#### v40 MoveCard reply
Hook：
```text
MengJin::NetMsgMoveCardRpy 0x10026afec
-> synthetic cave 0x100003800
```
普通 MengJin marker!=0xA7：重放原首指令并回原函数。

synthetic 11146：
1. `dataCnt<=cardCnt<=target.currHp`。
2. 显式 IDs：去重并验证仍属于 target Hand/Equip。
3. `cardCnt-dataCnt` 张隐藏手牌：
   - 从 HandZone `CPlayCard*` vector 取真实牌；
   - 每张使用 `rand_uint` 选随机起点并环形扫描；
   - 跳过已经显式/随机选中的 ID，保证多选不重复。
4. 所有真实 IDs 在移动前完整确定。
5. 每张牌建立一条 synthetic turn-end state。
6. 原生 `CMoveCardAction::MoveCards`：
```text
target.hand(+0x90) / equip(+0xc8)
    -> target.removed(+0x138)
spellId=11146
```
7. child ClearWaiting + SetOverMark。

#### Turn-end restore
沿用已审计的 v38 support lifecycle，但现在叠在 v39 正确 waiting state 上：
```text
CGame::TurnEnd
-> OnEventRemoveSpellState(type=3,currentTurnOwner)
-> synthetic state OnRemove
-> target.removed(+0x138) -> target.hand(+0x90)
```
只按 state 记录的 cardId 返还；绝不扫描整个 removed zone。

#### Cancel / timeout
第二层取消、超时只结束 synthetic child，不执行猛进规则。

### v40 真机测试优先级
1. v39 UI/timer/select regression。
2. 只选隐藏手牌 -> 真正扣 1 张。
3. 只选装备 -> 装备槽清空并进入 removed。
4. 混合多选 -> hidden hand 随机解析 + equipment explicit IDs。
5. 扣置后原 Sha 继续 ask-Shan / damage。
6. 观察 v39 的重复按钮是否在真实 MoveCards + 后续 trigger 后自然消失。
7. 当前回合结束 -> 全部返回 target hand。
8. v10 damage +1 / 酒叠加 / 其他武将 / 点将回归。

尚未真机验证 v40 的扣置/返还，不得把它标记为已完成。

## 2026-09-05：v40 真机无效后的项目级重审 — v41 PubGsCMoveCard 协议修复

### 真机事实
用户验证 `v40a` 与 `v39a` 效果一致：第二层面板仍可操作，但确认后没有实际扣置牌。因此 v40 的“扣置/返还已接入”不得视为有效实现。

### iOS 4.4.2.4 真正的 PubGsCMoveCard 分发链
重新从旧 iOS Mach-O 还原：

```text
客户端 PubGsCMoveCard (opcode 21209)
-> CGLogicCore::ProcessNetMsg ≈ 0x10017e0cc
-> switch index 0 ≈ 0x10017e16c
-> 公共 packet 校验
-> game+0xb0 取 latest action
-> 要求 action +0x78 waiting bit1
-> action vtable +0x20
-> NetMsgMoveCardRpy
```

这证明 v40 hook `MengJin::NetMsgMoveCardRpy @ 0x10026afec` 的 virtual slot 本身是正确的；不能再解释成“回包落到别的函数”。

### 单机 PubGsCMoveCard 的确定 packet layout
`PubGsCMoveCard` Lua 与 iOS native 一致：

```text
+0x0c fromZone      u8
+0x0d toZone        u8
+0x0e fromId        u8
+0x0f toId          u8
+0x10 fromPosition  u16
+0x12 toPosition    u16
+0x14 srcSeatId     u8
+0x15 spellId       u16
+0x17 cardCnt       u16
+0x19 dataCnt       u16
+0x1b data[]        u16[]
```

特别注意：`MsgBase:compareVersionInGame()` 在 single mode 返回 false，因此即使较新客户端通常用 UInt32 cardId，**单机 PubGsCMoveCard 的 data[] 仍按 UInt16 编码**。以后不得把联网 4.0.5.9+ 的 UInt32 格式套到此处。

### 旧 iOS 公共校验的确定规则
在 `CGLogicCore::ProcessNetMsg` 中已逐指令确认：

```text
1 <= cardCnt <= 256
dataCnt <= cardCnt
packetLen == 27 + 2*dataCnt
```

所以 PoJunNew 的一张隐藏手牌：

```text
cardCnt=1
dataCnt=0
```

在公共层是合法的。

### 旧协议代际 bug：重复 ID 检查错误使用 cardCnt
原 4.4.2.4 在长度按 `dataCnt` 校验以后，重复 cardId 检查却按 `cardCnt` 遍历 `msg+0x1b`。

对新版 PoJunNew：隐藏对手手牌计入 `cardCnt`，但不进入 `data[]`。因此：

```text
(1,0) 一张隐藏手牌：cardCnt<2，碰巧绕过重复检查
(2,0) 两张隐藏手牌：旧代码会读 data[] 之外的包尾，可能把 0 当重复而拒绝
(2,1) 隐藏+装备：结果依赖越界包尾，不能依赖
(2,2) 两张显式牌：原逻辑正常
```

v41 仅在 `spellId==11146` 时把这一段重复检查的 loop bound 改为 `dataCnt`；其他技能保持原行为。

Patch：

```text
0x10017e1bc -> cave 0x100003b00
```

### v40 handler 的确定机器码 bug
v40 report 写的是“允许 cardCnt>dataCnt 并补隐藏手牌”，但实际机器码相反：

```text
w25 = cardCnt
w26 = dataCnt
cbz w26, fail             // 错：dataCnt=0 直接失败
cmp w25,w26; b.hi fail    // 错：拒绝 cardCnt>dataCnt
compare w26 with targetHp // 错
allocate w26*4            // 错
explicit loop to w25      // 错，会按 cardCnt 读 data[]
```

所以用户若选一张隐藏手牌 `(1,0)`，v40 **机器码确定立即失败**；这足以解释“和 v39 完全一致”。

以后报告中的“意图逻辑”不得代替对最终 IPA 机器码的回检。复杂 cave 必须在打包后重新反汇编验证寄存器含义。

### v41 handler 修复合同

```text
cardCnt = msg+0x17
knownCnt = dataCnt = msg+0x19

require cardCnt > 0
require knownCnt <= cardCnt
require cardCnt <= target.currHp
allocate finalIds[cardCnt]

for i in [0, knownCnt):
    id = data[i]
    require id != 0
    require unique
    require id still in target.hand or target.equip
    finalIds.push(id)

while len(finalIds) < cardCnt:
    从 target HandZone 的真实 CPlayCard* vector
    按 MengJin 已验证方式随机起点扫描
    选择尚未进入 finalIds 的真实 cardId

for id in finalIds:
    确认 source=Hand 或 Equip
    建立 11146 turn-end synthetic state
    CMoveCardAction::MoveCards(source -> target.RemovedFromGameZone)

ClearWaiting
SetOverMark
```

handler 内 `handCount` 只在显式 ID 阶段结束后复用 w26；总 `cardCnt` 在 effect loop 每轮重新从 packet +0x17 读取，避免被 current cardId 覆盖。

### v41 静态 truth table

```text
1 hidden      1,0 -> PASS -> hidden 1
1 equip       1,1 -> PASS -> hidden 0
2 hidden      2,0 -> PASS -> hidden 2
hidden+equip  2,1 -> PASS -> hidden 1
2 equip       2,2 -> PASS, duplicate check over 2 known IDs
invalid       2,3 -> original public validator rejects
```

### v41 文件

```text
sgs_jiexusheng_v39_detain_restore_protocolfix_v41a.ipa
IPA SHA256 = 6aae91274d5a1098ebe66826b6d4fca64e9eb7583985175f9eacad0c0d10a053
main SHA256 = e993619af1dedfa72be92def97c16f6f6ca2639f6b77c60b785205fee009bcc2
```

v40 -> v41：1687 个文件，entry names 完全一致，只有主 Mach-O 内容变化；`game.lib / flist1 / XML / resources` 不变。

### 当前验证优先级
先只测试 **1 张隐藏手牌**。这是 v40 中已证明必然失败、而 v41 从公共 validator 到 action handler 已静态闭环的最小路径。

若一张隐藏手牌仍不移动：

```text
不要再回头猜 packet / virtual dispatch。
下一层直接审计 CMoveCardAction::MoveCards 的实参和它返回的 action 是否真正进入 ActionMgr。
```

若一张成功，再依次测试：1装备、2隐藏、隐藏+装备、Sha 是否继续、TurnEnd 是否返还。

### 继续冻结
- v34a 点将显示逻辑；
- v31 第一层 11146 ownership/candidate；
- v37 dedicated child / 官方 PoJunNew 面板；
- v39 resolveStep state1 等待与真实可交互 UI；
- v10 damage +1。

不得为了后半段 effect 回归而重新改这些层。


## 2026-09-05 维护：v41 真机反馈后的 v42a 修复尝试

### 真机事实
用户验证 v41a 后反馈：
- 第二层确认后仍会反复出现 11146 确认，直到手动取消；
- 每次确认都能继续弃掉对方若干手牌，说明 v41 的 `PubGsCMoveCard -> NetMsgMoveCardRpy -> MoveCards` 路径已实际生效；
- 暂不能扣装备牌；
- 目标武将脸上疑似出现破军标注，回合结束手牌数疑似恢复；
- v10 的伤害 +1 疑似穿透白银狮子，这是错误行为。

### v42a 改动
基线：`sgs_jiexusheng_v40_base_to_v41a_rebased.ipa` / v41a。

1. 重复确认/重复弃牌修复：
   - 不回退 v37/v39 官方 UI 与 waiting state；
   - 给 parent Sha 对象 `+0x13f` 写入 used marker `0xA8`；
   - v31 candidate 阶段若 parent Sha 已有 `0xA8`，返回不可用；
   - v37 fallback child 创建阶段若 parent Sha 已有 `0xA8`，拒绝再次创建 synthetic child；
   - 首次成功创建 synthetic child 时，继续写 child marker `0xA7`，并写 parent Sha marker `0xA8`。

2. 白银狮子修复：
   - 原 v10 hook 在 `0x100306384` 的防具/减伤处理调用之后执行，导致 +1 太晚，绕过白银狮子的最终伤害上限；
   - v42a 将 hook 前移到 `0x100306380`，即先执行破军 +1，再回到原 `mov x2,x20; bl 0x100258ef4` 防具/减伤链；
   - 恢复 `0x100306388` 为原始 `ldr w8,[x19,#0x98]`。

3. 装备扣置：
   - v42a 没有把匿名 `cardCnt-dataCnt` 强行解释成装备；
   - v41 handler 已有显式 ID 支路：先查 `target.hand(+0x90)`，再查 `target.equip(+0xc8)`，MoveCards 使用对应 source zone；
   - 如果真机仍无法扣装备，下一步应还原/记录 `UISelCardDlg:onSelCardsForPoJun(confirm)` 对装备的 `PubGsCMoveCard` 回包：是否把装备真实 `cardId` 放入 `data[]`，以及 `fromZone/fromPosition` 是否携带装备信息；不要在 native 端盲猜。

### v42a 输出
```text
sgs_jiexusheng_v41_repeat_baiyin_fix_v42a.ipa
```

静态边界：
- 相对 v41a 只改主 Mach-O；
- game.lib / flist1 / XML / 资源不变；
- v31/v37/v39/v41 主链继续保留。

### v42a 测试预期
1. 一次【杀】内只应出现一次破军第一层确认；第二层确认后不应继续重复弹出直到取消。
2. 选对方隐藏手牌仍应扣置，并显示破军标记；当前回合结束应返回。
3. 白银狮子目标受到满足条件的界徐盛【杀】时，最终伤害应被白银狮子限制为 1，不再被破军 +1 穿透。
4. 装备牌若客户端发送真实 `cardId`，应被扣置；若仍失败，下一轮只查客户端 equipment 回包，不修改 hand/equip MoveCards。

## 2026-09-06：v42 真机确认 + v43 白银狮子优先级再次前移

### v42a 真机结果
用户验证 `sgs_jiexusheng_v41_repeat_baiyin_fix_v42a.ipa`：

```text
已修复：
- 11146 重复确认/重复弃牌；
- 手牌扣置；
- 装备扣置；
- 破军标记显示；
- 当前回合结束返还。

仍未修复：
- 破军 damage +1 仍会穿透白银狮子。
```

### 根因修正
v42a 把破军 +1 从 `0x100306388` 前移到 `0x100306380`，即只放在最后一个 helper `0x100258ef4` 前。
真机仍穿透白银狮子，说明白银狮子的最终减伤/封顶不在最后 helper，而在更早的 damage/equipment helper 中。

当前 damage 函数相关顺序：

```text
0x100306330 -> call 0x10016b7bc
0x100306354 -> call 0x100188704
0x10030636c -> call 0x100288808
0x100306384 -> call 0x100258ef4
0x100306388 -> ldr final damage
0x10030638c -> store final damage
```

因此 v43a 将已有 11146 +1 cave 的入口从 `0x100306380` 再前移到 `0x100306330`，让所有四个原生 damage/equipment helper 都能看到已经 +1 后的 damage。

### v43a patch

```text
0x306330: 64224339 -> b4f1f317
  原始：ldrb w4,[x19,#0xc8]
  新增：branch to existing damage +1 cave @ 0x100002a00

0x306380: a0f1f317 -> e20314aa
  恢复原始 mov x2,x20

0x2aac: e20314aa350e0c14 -> 64224339210e0c14
  cave tail 从 replay mov x2,x20; return 0x306384
  改为 replay ldrb w4,[x19,#0xc8]; return 0x306334
```

### v43a 输出

```text
sgs_jiexusheng_v42_baiyin_early_fix_v43a.ipa
IPA SHA256: 82bfb44d177c7235772b57383790e2ff89eab3aba0da6d2f2487bf7d42b3bbf3
Main SHA256: 075211691ea069371c2f5c9225eaba2fa7ab039fc016d3e174699ebe1643cd1f
ZIP integrity: OK
Only changed zip entry vs v42a: Payload/三国杀单机版.app/三国杀单机版
```

### 测试预期

1. 已修复项不得回归：一次杀只触发一次破军，手牌/装备可扣，标记显示，回合结束返还。
2. 白银狮子目标受到界徐盛满足条件的杀时，最终伤害应限制为 1。
3. 非白银狮子目标仍应保持破军 +1；酒杀叠加不应回归。



## 2026-09-06 v42a 全局结算错乱后的项目级回归审计

### 真机事实
用户反馈 v42a 虽修复了重复确认、装备扣置、破军标记、回合结束返还，但整局结算出现更致命问题：有时莫名弃牌和回合错乱。此时必须暂停界徐盛技能继续实现，先与最早稳定无曹冲/界徐盛基线做全项目对比。

### 对比结论
最早稳定基线 `com.yoka.sgsdj_4.4.2.4_mountain7_no_taishi_unlimited_dj.ipa` SHA256 为 `770a2568bc32c893636486d707c167c2e5f8147264a94838f267de42107d16f4`。

`stage1_config_resources` 相对基线只新增 174/491 资源并修改 character.xml / gs_character_config.xml / spell.xml，主 Mach-O 完全相同。

`clean_visible_no_rule_bridge` 相对基线主 Mach-O 也完全相同，只涉及可见性所需 game.lib/runtime injection 与 flist1。

因此，整局结算错乱不应优先归因于 XML/资源/可见性配置，而应优先归因于 v37-v43 主 Mach-O 全局 hook。

### 高风险改错点
1. v42a 为修重复触发而写 `parent Sha +0x13f = 0xA8`。`+0x13f` 只曾证明是 MengJin synthetic child 0x140 对象尾部 marker，可用作 child marker；从未证明 Sha parent 对象在 +0x13f 处也是安全 padding。向 Sha parent 写该字节可能破坏 Sha action/state，导致回合/结算异常。
2. v38-v42 的 turn-end restore 使用全局 `CBiFaState::IsCanBeRemove` / `OnRemove` hook，并借 212 state 记录 cardId。若 filter 或 state 字段判断不严，可能在 unrelated state cleanup 时执行错误 MoveCards/restore，表现为莫名弃牌或回合结束异常。
3. v42/v43 白银狮子尝试改的是中心 damage settlement 函数；此类 patch 必须最后处理，不能在全局不稳定时继续前移/试探。

### 下一步强制策略
先做全局隔离版 v44a：从 `clean_visible_no_rule_bridge` 复制，保留 174/491 可见性配置和资源，但移除全部 PoJun native hooks / damage hook / turn-end restore hook，主 Mach-O 回到最早稳定基线。

v44a 只用于验证普通整局流程是否恢复稳定；不要测试界徐盛技能完整性。若 v44a 普通局稳定，则问题确定在主 Mach-O hook，下一轮应按优先级逐个重新引入：先不写 Sha parent marker；再改用更安全的 one-shot flag 位置或 action-local state；最后才重新设计 restore。


## 2026-09-08 最终维护状态：v49a 可用基线

当前推荐基线：`sgs_v48_sha_only_trigger_fix_v49a.ipa`。

SHA：

```text
IPA SHA256 = 8711fbe9730e7c05031057365e3e91fe4fce2ef96fbc95dcde1136efe8f1efb8
Main SHA256 = 28825812ab73db8f4500b775a6b6b8b3355b7dd31a701ccc6d7519e4e8b346c3
```

已验证：

```text
1. 普通整局稳定，无 v42a/v43a 的莫名弃牌/回合错乱。
2. 174/491 可见可选。
3. 界徐盛【杀】触发破军，【决斗】不触发。
4. 一次【杀】内只确认一次。
5. PoJunNew 第二层手牌/装备可扣置。
6. 当前回合结束返还。
7. 白银狮子/同类减伤不再被破军 +1 穿透。
```

关键规则：

```text
1. 后续以 v49a 为基线。
2. 不恢复 Lua 全局规则桥接。
3. 不使用 parent Sha +0x13f 作为 used marker。
4. 不把 role+0x1c4 当 Character ID。
5. 非杀误触发必须通过 parent action vtable / Sha action 校验处理。
6. 扣装备触发失去装备类技能目前视为原始事件链行为，不屏蔽。
7. 扣置装备在过拆/顺手面板中残留显示但点击不生效，暂定为客户端 UI cache 问题，除非影响结算，不优先修。
```
