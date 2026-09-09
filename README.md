# SGS iOS 单机联网武将本地移植

本仓库用于维护《三国杀单机版 iOS 4.4.2.4》的本地移植实验：在离线单机环境中，把部分原本依赖联网版本、或当前客户端未完整开放的武将技能，尽量按原版逻辑补回到本地可运行版本中。

当前重点是“界徐盛 / 新破军”的完整本地实现。项目目标不是简单改 XML 让武将出现，而是让武将能够在本地局内完成真实技能流程，包括触发条件、确认窗口、选牌面板、扣置、标记、回合结束返还、伤害加成，以及与防具/锦囊/其他技能的兼容。

## 当前进度

当前稳定归档版本：`v49a`。

已完成：

- 界徐盛已可在本地正常选择。
- 新破军只会在界徐盛使用【杀】时触发。
- 已修复【决斗】误触发新破军的问题。
- 一次【杀】内新破军只确认一次，不再重复弹窗。
- 第一层确认/取消流程正常；取消后原【杀】继续结算。
- 第二层使用官方 PoJunNew 选牌面板，可选择目标手牌和装备。
- 被扣置牌会从当前区域移出，并显示破军标记。
- 当前回合结束后，被扣置牌会正常返还。
- 破军伤害 +1 已接入原始伤害 pipeline，白银狮子及同类减伤/封顶效果正常处理。
- 普通武将、普通出牌、杀/闪/伤害/弃牌/回合结束等主体结算保持稳定。

已知取舍：

- 扣置装备属于从装备区失去牌，因此可能触发白银狮子、孙尚香【枭姬】等“失去装备”类被动。本项目当前保留原始事件链行为，不强行屏蔽。
- 已扣置装备在部分客户端 UI 中可能仍被过河拆桥/顺手牵羊面板显示，但已验证点击不生效；当前视为显示缓存问题，不优先改动，以避免破坏主体结算。

## 仓库内容

```text
README.md
reports/
  FINAL_REPORT_v49.md
  sgs_v48_sha_only_trigger_fix_v49a_report.txt
skills/
  SKILL_7_final_v49.md
artifacts/
  sgs_v48_sha_only_trigger_fix_v49a.ipa
  sgs_v48_sha_only_trigger_fix_v49a.ipa.sha256
```

说明：

- `README.md`：项目目标、当前进度和使用方式。
- `reports/FINAL_REPORT_v49.md`：从基线到 v49a 的综合维护报告。
- `reports/sgs_v48_sha_only_trigger_fix_v49a_report.txt`：v49a 构建与 patch 校验报告。
- `skills/SKILL_7_final_v49.md`：维护本项目时使用的逆向/补丁经验记录。
- `artifacts/*.ipa`：最终可安装包，未签名或需自行重签。
- `artifacts/*.sha256`：IPA 校验文件。

## 如何使用

### 1. 获取 IPA

下载最终 IPA：

```text
artifacts/sgs_v48_sha_only_trigger_fix_v49a.ipa
```

或从 GitHub Release 附件中下载同名文件。

### 2. 校验文件

使用 sha256 校验最终 IPA：

```text
8711fbe9730e7c05031057365e3e91fe4fce2ef96fbc95dcde1136efe8f1efb8  sgs_v48_sha_only_trigger_fix_v49a.ipa
```

macOS / Linux：

```bash
shasum -a 256 artifacts/sgs_v48_sha_only_trigger_fix_v49a.ipa
```

Windows PowerShell：

```powershell
Get-FileHash .\artifacts\sgs_v48_sha_only_trigger_fix_v49a.ipa -Algorithm SHA256
```

### 3. 重签并安装

该 IPA 不是 App Store 安装包。安装到 iOS 设备前，需要使用你自己的证书/工具重签，例如 AltStore、Sideloadly、TrollStore、企业证书或自有开发者证书流程。

本仓库不提供证书，也不包含自动签名流程。

### 4. 游戏内验证

推荐测试顺序：

```text
1. 普通武将打一局，确认主体结算稳定。
2. 选择界徐盛。
3. 使用【杀】，确认出现新破军提示。
4. 点取消，确认原【杀】继续结算。
5. 再次使用【杀】，点确定，进入目标手牌/装备选择面板。
6. 扣置手牌，确认标记和回合结束返还。
7. 扣置装备，确认装备移出、标记、回合结束返还。
8. 目标装备白银狮子时测试伤害，确认最终伤害不穿透防具封顶。
9. 使用【决斗】，确认不会触发新破军。
```

## 维护原则

本项目经历过多轮回归问题，后续维护必须遵守以下原则：

1. 以 `v49a` 作为当前稳定基线。
2. 不要随意恢复 v42a/v43a 的全量 native hook；这些版本曾出现整局结算错乱。
3. 每次只改一个层级，并做普通局回归测试。
4. 优先保持游戏主体完整性，再追求单个技能细节。
5. 不要把 UI 显示缓存问题升级成全局牌区/结算逻辑改动，除非有明确证据。
6. 修改 action/state 相关逻辑时，必须先证明 parent action、source role、target role、state machine 的真实语义。

## 当前最终结论

截至 `v49a`，界徐盛已经在本地单机版中完整实现，核心技能链路和主体游戏结算均已通过真机测试。
