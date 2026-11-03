# nas-ssh-ops-skill

通过 SSH 操作 NAS 的标准规范 Skill，适用于 Claude Code / Codex / WorkBuddy / CatPaw 等 Agent。

## 背景

为 mac 与 NAS 的 Tailscale 组网过程中沉淀出的 SSH 操作规范，固化为 Agent 标准技能，让任何 Agent 动 NAS 时都自动遵守同一份纪律：

- `BatchMode=yes` 强制密钥，杜绝脚本卡在密码提示
- `'bash -s'` + 引号 heredoc 批量执行，避免反复握手
- 敏感配置脱敏展示，token / auth key 不外泄
- 备份 → `sed` → `diff` 的配置修改流程
- 借容器 root 写宿主机目录
- docker 组免 sudo，systemd `kill` 重启法
- 不代持密码，需要交互 sudo 的操作交给用户

## 适用范围

以绿联 DH4300 Plus（UGOS）为例，等价适用于群晖、飞牛、普通 Linux。

## 安装

把 `SKILL.md` 放到对应 Agent 的 skills 目录即可：

```bash
# Claude Code
ln -s "$PWD/SKILL.md" ~/.claude/skills/nas-ssh-ops/SKILL.md

# Codex
ln -s "$PWD/SKILL.md" ~/.codex/skills/nas-ssh-ops/SKILL.md

# WorkBuddy
ln -s "$PWD/SKILL.md" ~/.workbuddy/skills/nas-ssh-ops/SKILL.md
```

或直接 symlink 整个目录：

```bash
mkdir -p ~/.shared-agent/skills
ln -s "$PWD" ~/.shared-agent/skills/nas-ssh-ops
```

## 使用

在 Agent 会话中说「通过 SSH 操作 NAS」或「排查 NAS 上的 xxx 服务」即可触发本规范。Agent 会按本规范执行：先验证免密、heredoc 批量执行、脱敏展示、备份后修改、diff 校验。

## 内容

- `SKILL.md` — 规范主体（8 章：前置条件 / 连接方式 / 读文件 / 写文件 / Docker 权限 / 改配置流程 / 诊断命令 / 安全纪律 + 场景对照表）

## 相关

- 《Tailscale 实战》— mac 与 NAS 的 Tailscale 组网过程，NAS 部分已对齐本规范
- 《通过 SSH 高效操作 NAS：可直接复制的标准规范》（2026-11-02-nas-ssh-operations.md）

## License

MIT