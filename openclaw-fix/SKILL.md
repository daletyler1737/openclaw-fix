---
name: openclaw-fix
description: OpenClaw 服务器（116/117）故障诊断与修复技能。当用户报告 bot 不回复、gateway 报错（No API key、timeout、401 认证失败等）、或需要重启/修复 OpenClaw gateway 时使用。适用于局域网内 116（192.168.1.116）和 117（192.168.1.117）两台服务器的 OpenClaw 问题排查。
---

# OpenClaw Fix Skill

用于诊断和修复 OpenClaw 服务器（116/117）的各类故障。

## 服务器连接信息

| 机器 | IP | 用户 | 密码 | Gateway 端口 | token |
|------|-----|------|------|-------------|-------|
| 116 | 192.168.1.116 | dale005 | Bot2#nee | 18789 | 44fb47f... |
| 117 | 192.168.1.117 | dale006 | Bot2#nee | 18789 | 12c0da7... |

## 重要教训

⚠️ **教训1：API baseUrl 必须用 `api.minimaxi.com`，不是 `api.minimax.io`**
- 主服务器用 `api.minimaxi.com` ✅
- 116/117 初始配置错用了 `api.minimax.io` ❌ → 报 401 invalid api key
- 修复：`sed -i 's|api.minimax.io|api.minimaxi.com|g' models.json`

⚠️ **教训2：三个文件必须同时配置**
- `auth-profiles.json` — 存储 API key（容易忘）
- `models.json` — provider 定义（baseUrl、model ID）
- `openclaw.json` — 必须有 `agents.defaults.model.primary` 指定默认模型

⚠️ **教训3：修改配置后必须重启 gateway**
- `kill -9` 所有 openclaw 进程后重启

---

## 快速修复流程

遇到 bot 故障时，按以下顺序检查：

### Step 1: 检查 Gateway 状态
```bash
sshpass -p 'Bot2#nee' ssh -o StrictHostKeyChecking=no dale005@192.168.1.116 "ps aux | grep openclaw-gateway | grep -v grep; tail -5 /home/dale005/openclaw.log"
```

### Step 2: 检查关键错误
```bash
# 401 认证失败 → 检查 baseUrl 是否正确
grep '401\|invalid api key' /home/dale005/openclaw.log

# No API key found → 检查 auth-profiles.json 是否存在
cat /home/dale005/.openclaw/agents/main/agent/auth-profiles.json

# 模型是 openai/gpt-5.4 而不是你配置的 → 检查 openclaw.json 是否缺少 agents.defaults
grep 'agent model' /home/dale005/openclaw.log
```

### Step 3: 对症修复

| 日志关键词 | 根因 | 修复 |
|-----------|------|------|
| `401 invalid api key` + baseUrl 是 `api.minimax.io` | baseUrl 错误 | sed 替换为 `api.minimaxi.com` 后重启 |
| `No API key found for provider "openai"` | 未配置 `agents.defaults.model.primary` | 在 openclaw.json 添加 agents.defaults |
| `No API key found for provider "minimax"` | auth-profiles.json 缺失或 key 错误 | 复制 auth-profiles.json |
| 多个残留进程 | 之前 gateway 未完全关闭 | `pkill -9 -f openclaw` 后重启 |
| `timeout of 10000ms exceeded` | minimax API 网络不通 | 检查网络和 baseUrl |

### Step 4: 重启 Gateway
```bash
kill -9 $(ps aux | grep openclaw-gateway | grep -v grep | awk '{print $2}') 2>/dev/null
sleep 2
nohup openclaw gateway > /home/dale005/openclaw.log 2>&1 &
sleep 20
tail -5 /home/dale005/openclaw.log
```

---

## API Key 配置

minimax API key（所有机器通用）：
```
sk-cp-ymanZIbnWmh_CnFD_xrKpUjDARS2U1F9fZgwWk3NhM9ILq8B6ayyDAm-vi39Et-uiCXSTnjrJRV9IMoUroVOePgnNDee-NlIcnKz2mI2dYmDw1NSKFQcXOA
```

**auth-profiles.json 内容**：
```json
{
  "version": 1,
  "profiles": {
    "minimax-portal:default": {
      "type": "oauth",
      "provider": "minimax-portal",
      "access": "sk-cp-ymanZIbnWmh_CnFD_xrKpUjDARS2U1F9fZgwWk3NhM9ILq8B6ayyDAm-vi39Et-uiCXSTnjrJRV9IMoUroVOePgnNDee-NlIcnKz2mI2dYmDw1NSKFQcXOA",
      "refresh": "sVXqphm_h3yWmZPvSzOX8_Fo4i0OCRzjTxOIZ2MTSzk=",
      "expires": 1806347961565
    }
  }
}
```

**models.json 中的 baseUrl 必须正确**：
```json
"minimax-portal": {
  "baseUrl": "https://api.minimaxi.com/anthropic/v1",
  ...
}
```

**openclaw.json 必须包含 agents.defaults**：
```json
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "minimax-portal/MiniMax-M2.7"
      }
    }
  }
}
```

---

## 部署新服务器 checklist

新安装 OpenClaw 后，必须依次完成以下配置：

1. ✅ 安装 OpenClaw（用 npm overrides 解决 libsignal 问题）
2. ✅ 创建 `/home/<user>/.openclaw/agents/main/agent/auth-profiles.json`
3. ✅ 创建 `/home/<user>/.openclaw/agents/main/agent/models.json`（baseUrl 用 `api.minimaxi.com`）
4. ✅ 在 `openclaw.json` 中添加 `agents.defaults.model.primary`
5. ✅ 配置飞书渠道（在 openclaw.json channels.feishu）
6. ✅ 重启 gateway
7. ✅ 配对测试

## 参考

- [troubleshooting.md](references/troubleshooting.md) — 完整故障排查手册
