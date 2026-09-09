# SGS iOS 单机移植维护仓库

当前归档版本：`v49a`。

## 最终状态

- 界徐盛已完美实现。
- v49a 已修复【决斗】误触发破军。
- 游戏主体结算稳定，无 v42a/v43a 的莫名弃牌和回合错乱。
- 破军只在【杀】触发，一次【杀】内只确认一次。
- 官方 PoJunNew 二层面板可扣置手牌/装备，武将牌旁标记正常，当前回合结束返还。
- 白银狮子及同类减伤/封顶 pipeline 正常。

## 文件

- `reports/FINAL_REPORT_v49.md`：最终综合报告。
- `reports/sgs_v48_sha_only_trigger_fix_v49a_report.txt`：v49a 原始构建报告。
- `skills/SKILL_7_final_v49.md`：本项目维护 Skill。
- `artifacts/sgs_v48_sha_only_trigger_fix_v49a.ipa.sha256`：最终 IPA 校验。

## IPA

最终 IPA 文件：`sgs_v48_sha_only_trigger_fix_v49a.ipa`

SHA256：

```text
8711fbe9730e7c05031057365e3e91fe4fce2ef96fbc95dcde1136efe8f1efb8
```

注意：IPA 本体约 83.72 MB。若未放入 GitHub 仓库，请用该 sha256 文件校验本地/Release 附件。
