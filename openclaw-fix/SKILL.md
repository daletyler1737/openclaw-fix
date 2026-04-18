---
name: openclaw-fix
description: OpenClaw 服务器（113/114）故障诊断与修复技能。当用户报告 bot 不回复、gateway 报错（No API key、timeout、401 认证失败等）、或需要重启/修复 OpenClaw gateway 时使用。适用于局域网内 113（192.168.1.113）和 114（192.168.1.114）两台服务器的 OpenClaw 问题排查。
---

# OpenClaw Fix Skill

用于诊断和修复 OpenClaw 服务器（113/114）的各类故障。

## 参考文档

详细故障排查步骤见 [troubleshooting.md](references/troubleshooting.md)。

## 快速修复流程

遇到 bot 故障时，按以下顺序检查：

### Step 1: 检查进程和 Gateway 状态
```bash
sshpass -p '<password>' ssh -o StrictHostKeyChecking=no dale@<IP> "ps aux | grep openclaw | grep -v grep; curl -s --connect-timeout 5 http://localhost:18789/health"
```
- 113 密码: `173784`，auth token: `3d24ff0b43d3f796c5f10503f08881898224616b8dfad70f`
- 114 密码: `814852896`，auth token: `621ede5fde363053ec27eac5c7df602119a23fec375473d1`

### Step 2: 检查日志中的错误
```bash
tail -30 /tmp/openclaw-1000/openclaw-2026-04-18.log | grep -E 'error|fail|timeout|auth'
```

### Step 3: 对症修复

| 日志关键词 | 根因 | 修复 |
|-----------|------|------|
| `already running` | 残留进程 | `pkill -9 -f openclaw` 后重启 |
| `No API key found for provider` | auth-profiles.json 缺失 | 创建 auth-profiles.json |
| `timedOut: true` | 模型 API 不可达 | 切换到 minimax |
| `401 Invalid Authentication` | API key 失效 | 切换 provider |
| `LLM idle timeout` | 模型响应超时 | 检查网络或切换 provider |

### Step 4: 重启 Gateway
```bash
pkill -9 -f openclaw
sleep 2
nohup openclaw gateway > /dev/null 2>&1 &
sleep 25
```

## 重要配置位置

OpenClaw API key 存储在**两处**，修复时两处都要检查：
- `models.json` — provider 定义（baseUrl、model ID）
- `auth-profiles.json` — 实际 API key（容易遗漏）

## 参考

- [troubleshooting.md](references/troubleshooting.md) — 完整故障排查手册
