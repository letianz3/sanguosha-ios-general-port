# SGS iOS 4.4.2.4 曹冲 / 界徐盛移植最终综合报告（v49a）

## 当前最终结论

v49a 为当前最终成功版本。用户最终真机补充验证：**决斗误触发破军 bug 已成功修复，界徐盛已经完美实现。**

## 最终版本

- IPA: `sgs_v48_sha_only_trigger_fix_v49a.ipa`
- IPA SHA256: `8711fbe9730e7c05031057365e3e91fe4fce2ef96fbc95dcde1136efe8f1efb8`
- IPA size: `87790970` bytes
- Repository maintenance date: 2026-09-08

## 已验证功能

1. 新武将 174 曹冲、491 界徐盛可见可选。
2. 普通整局结算稳定，没有 v42a/v43a 的莫名弃牌和回合错乱。
3. 界徐盛【破军】最终实现：
   - 仅【杀】触发；
   - 【决斗】不再误触发；
   - 一次【杀】内只确认一次；
   - 第二层官方 PoJunNew 面板可选择目标手牌/装备；
   - 手牌/装备可扣置到武将牌旁；
   - 当前回合结束返还；
   - 破军标记正常显示。
4. 白银狮子/同类减伤 pipeline 已处理：破军 +1 先进入基础伤害，再由原始防具/减伤/封顶 helper 处理。

## 最终版本演进摘要

### v44a：全局稳定基线

从最早稳定版本回退掉所有界徐盛 native hooks，只保留新武将资源、XML 和可见性所需 game.lib。真机结果：整局稳定，但界徐盛不可选择。

### v45a：稳定 + 可选

在 v44a 上只修点将配置：

- 174 / 491 `Ex=7`
- `AssignRatio=20`
- `CanUseDianJiangKa=1`

真机结果：整局稳定，新武将可选。

### v46a：恢复界徐盛 v41 技能逻辑

在 v45a 上恢复 v41a 的核心 native 逻辑：

- v31 第一层确认；
- v37 dedicated synthetic child；
- v39 第二层 state wait；
- v41 `PubGsCMoveCard` protocol fix；
- hand/equip 扣置；
- turn-end restore。

真机结果：技能逻辑恢复，但一次杀内重复确认回归。

### v47a：重复确认修复

只恢复 v42a 的 repeat-only guard，不恢复造成全局风险的 damage hook。真机结果：一次杀内重复确认修复，整局稳定。

### v48a：白银狮子 / pipeline 修复

将破军 +1 从后置改成前置到 damage helper pipeline 之前。真机结果：白银狮子及类似减伤/封顶问题修复。

### v49a：非杀误触发修复

修复界徐盛使用【决斗】误触发破军：

```text
parentAction = *(optAction + 0xa0)
parentVptr = *(parentAction)
require parentVptr == Sha vtable 0x1010ea9a0
```

真机结果：决斗不再触发破军；杀仍正常触发。

## 关键工程边界

1. 不再把 parent Sha `+0x13f` 当作安全字段盲写。该位置只在 synthetic MengJin child 0x140 对象尾部 marker 中证明安全。
2. 不再把 `role + 0x1c4` 当 Character ID，它在多个 native 路径中更接近 byte/seat 语义。
3. 不再用 v29/v30 的错误 ownership filter。
4. 不再恢复 Lua 全局规则桥接。
5. 后续维护以 v49a 为最终成功基线，按小步 patch + 真机 smoke test 方式推进。

## 已知行为说明

1. 扣除装备时可能触发白银狮子、孙尚香【枭姬】等“失去装备/失去牌”类被动。当前实现是从 `target.equip(+0xc8)` 经原始 MoveCards 进入 `target.removed(+0x138)`，因此按原始事件链触发属于可解释行为，本版不屏蔽。
2. 被扣置过的装备可能仍在后续【过河拆桥】/【顺手牵羊】面板中显示，但实测点击不生效。当前先视为客户端显示缓存问题，不强行改 UI cache，以避免重新引入全局结算风险。

## v49a 原始构建报告

```text
界徐盛 v49a：只修非杀误触发破军；不处理失去装备被动

基线: sgs_v47_baiyin_pipeline_fix_v48a.ipa
输出: sgs_v48_sha_only_trigger_fix_v49a.ipa
ZIP integrity: OK
Output IPA SHA256: 8711fbe9730e7c05031057365e3e91fe4fce2ef96fbc95dcde1136efe8f1efb8
Output main SHA256: 28825812ab73db8f4500b775a6b6b8b3355b7dd31a701ccc6d7519e4e8b346c3

一、改动
1. 只修“决斗触发破军”：在 11146 candidate repeat guard 中增加 parent action vtable 校验。
2. 只有 parent action 的 vptr == Sha vtable 才允许进入 11146 candidate 逻辑。
3. 不修改 MoveCards / TurnEnd restore / damage pipeline / game.lib / XML / 资源。
4. 不屏蔽白银狮子、枭姬等失去装备触发。

二、改动对应项目代码依据
v31/v48 的第一层 gate 只验证：
- skillId == 11146
- parent action +0x110/+0x90 得到的 sourceRole
- currentRole == sourceRole
- sourceRole.HasSpellId(11146)

这可以区分“谁使用”，但不能区分 parent action 是 Sha 还是 Duel。
所以界徐盛使用决斗时，也可能满足 sourceRole/HasSpellId 条件而误触发。

本版新增硬校验：
- parent action = *(optAction+0xa0)
- parent vptr = *(parent action)
- Sha vptr = 0x1010ea9a0
- 只有 parent vptr 匹配 Sha 才继续 11146。

装备扣置触发白银狮子/枭姬：
- 当前实现确实从 target.equip(+0xc8) MoveCards 到 target.removed(+0x138)。
- 按原始事件链，这属于从装备区失去牌。
- 因此本版不把失去装备类被动当 bug 屏蔽，避免破坏其他角色/装备结算完整性。

扣置过的装备后续在过拆/顺手 UI 中仍可见：
- 你已验证点击不生效，说明 native zone/MoveCards 权限层已阻止。
- 本轮不做客户端显示层强行过滤，避免引入新的 UI/cache 全局回归。

三、实际 patch
0x2b60 cave replaced with:

/mnt/data/sgs_v49_fix/patch_v49.o:	file format mach-o arm64

Disassembly of section __TEXT,__text:

0000000000000000 <ltmp0>:
       0: b4ffff6a     	cbz	x10, 0xffffffffffffffec <ltmp0+0xffffffffffffffec>
       4: f940014d     	ldr	x13, [x10]
       8: 9000874e     	adrp	x14, 0x10e8000 <ltmp0+0x10e8000>
       c: 912681ce     	add	x14, x14, #0x9a0
      10: eb0e01bf     	cmp	x13, x14
      14: 54fffec1     	b.ne	0xffffffffffffffec <ltmp0+0xffffffffffffffec>
      18: 3944fd4c     	ldrb	w12, [x10, #0x13f]
      1c: 7102a19f     	cmp	w12, #0xa8
      20: 54fffe60     	b.eq	0xffffffffffffffec <ltmp0+0xffffffffffffffec>
      24: f940894b     	ldr	x11, [x10, #0x110]
      28: 17ffffe4     	b	0xffffffffffffffb8 <ltmp0+0xffffffffffffffb8>


四、测试预期
1. 界徐盛使用【杀】：破军仍正常触发。
2. 界徐盛使用【决斗】：不应触发破军确认。
3. 手牌/装备扣置、破军标记、回合结束返还不回归。
4. 白银狮子/枭姬等失去装备触发保持原始事件链。
5. 扣置过的装备若仍在过拆/顺手 UI 中显示，但点击不生效，先视为显示缓存问题，不影响本版规则正确性。

```
