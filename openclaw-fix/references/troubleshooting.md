# OpenClaw 116/117 故障修复手册

## 服务器连接信息

| 机器 | IP | 用户 | 密码 | Gateway 端口 | token |
|------|-----|------|------|-------------|-------|
| 116 | 192.168.1.116 | dale005 | Bot2#nee | 18789 | 44fb47f86d17de82fa3030a953071d58a6357459ba0b6398 |
| 117 | 192.168.1.117 | dale006 | Bot2#nee | 18789 | 12c0da7bca0d21fa7a5104b43560e200155c0e1b03d99694 |

## 快速检查命令

```bash
# 116 - 检查进程状态
sshpass -p 'Bot2#nee' ssh -o StrictHostKeyChecking=no dale005@192.168.1.116 "ps aux | grep openclaw | grep -v grep"

# 117 - 检查进程状态
sshpass -p 'Bot2#nee' ssh -o StrictHostKeyChecking=no dale006@192.168.1.117 "ps aux | grep openclaw | grep -v grep"

# 116 - 检查 gateway health
curl -s http://192.168.1.116:18789/health

# 116 - 查看 recent 日志
tail -30 /home/dale005/openclaw.log

# 117 - 查看 recent 日志
tail -30 /home/dale006/gw.log
```

## 常见故障与解决方案

### 1. Bot 能已读但不回复（残留进程）

**症状**：消息已读但无回复，`ps aux` 显示多个 openclaw 进程

**修复**：
```bash
kill -9 $(ps aux | grep openclaw-gateway | grep -v grep | awk '{print $2}') 2>/dev/null
sleep 2
nohup openclaw gateway > /home/dale005/openclaw.log 2>&1 &
sleep 25
```

### 2. "No API key found for provider XXX"

**症状**：报错 `No API key found for provider "minimax"` 或其他 provider

**原因**：OpenClaw API key 存在 `auth-profiles.json`，不是只存在 `models.json`

**修复**：
```bash
# 创建 auth-profiles.json
sshpass -p 'Bot2#nee' ssh -o StrictHostKeyChecking=no dale005@192.168.1.116 "mkdir -p /home/dale005/.openclaw/agents/main/agent && cat > /home/dale005/.openclaw/agents/main/agent/auth-profiles.json << 'EOF'
{\"version\":1,\"profiles\":{\"minimax-portal:default\":{\"type\":\"oauth\",\"provider\":\"minimax-portal\",\"access\":\"sk-cp-ymanZIbnWmh_CnFD_xrKpUjDARS2U1F9fZgwWk3NhM9ILq8B6ayyDAm-vi39Et-uiCXSTnjrJRV9IMoUroVOePgnNDee-NlIcnKz2mI2dYmDw1NSKFQcXOA\",\"refresh\":\"sVXqphm_h3yWmZPvSzOX8_Fo4i0OCRzjTxOIZ2MTSzk=\",\"expires\":1806347961565}}}
EOF"
```

### 3. HTTP 401 authentication_error: invalid api key

**症状**：日志出现 `401 {"type":"error","error":{"type":"authentication_error","message":"invalid api key"}}`

**根因**：models.json 中的 baseUrl 错误使用了 `api.minimax.io`，实际应该是 `api.minimaxi.com`

**验证方法**：
```bash
# 从服务器测试 API 是否通
curl -H "Authorization: Bearer <KEY>" -H "Content-Type: application/json" https://api.minimaxi.com/anthropic/v1/messages -d '{"model":"MiniMax-M2.7","max_tokens":10,"messages":[{"role":"user","content":"test"}]}'
# 返回 message → API 正常
# 返回 401 → key 失效或 baseUrl 错误
```

**修复**：
```bash
# 修复 baseUrl
sshpass -p 'Bot2#nee' ssh -o StrictHostKeyChecking=no dale005@192.168.1.116 "sed -i 's|api.minimax.io|api.minimaxi.com|g' /home/dale005/.openclaw/agents/main/agent/models.json"
sshpass -p 'Bot2#nee' ssh -o StrictHostKeyChecking=no dale006@192.168.1.117 "sed -i 's|api.minimax.io|api.minimaxi.com|g' /home/dale006/.openclaw/agents/main/agent/models.json"
```

### 4. 模型是 openai/gpt-5.4 而不是 MiniMax-M2.7

**症状**：日志显示 `agent model: openai/gpt-5.4`，但你在 models.json 配置的是 minimax

**根因**：openclaw.json 缺少 `agents.defaults.model.primary` 配置，导致 gateway 使用内置默认模型

**修复**：
```bash
# 在 openclaw.json 中添加 agents.defaults
# 确保 openclaw.json 包含：
{
  "agents": {
    "defaults": {
      "model": {
        "primary": "minimax-portal/MiniMax-M2.7"
      },
      "models": {
        "minimax-portal/MiniMax-M2.7": {
          "alias": "minimax-m2.7"
        }
      }
    }
  }
}
```

### 5. timeout of 10000ms exceeded

**症状**：日志出现 `timeout of 10000ms exceeded`

**根因**：可能是网络问题，或 minimax API 不可达

**检查**：
```bash
sshpass -p 'Bot2#nee' ssh -o StrictHostKeyChecking=no dale005@192.168.1.116 "curl -v --max-time 10 https://api.minimaxi.com/anthropic/v1/messages -H 'Authorization: Bearer <KEY>'"
```

---

## API Key 说明

**minimax 全局 API key**（所有机器通用）：
```
sk-cp-ymanZIbnWmh_CnFD_xrKpUjDARS2U1F9fZgwWk3NhM9ILq8B6ayyDAm-vi39Et-uiCXSTnjrJRV9IMoUroVOePgnNDee-NlIcnKz2mI2dYmDw1NSKFQcXOA
```

**两个 API 域名对比**：
| baseUrl | 状态 | 说明 |
|---------|------|------|
| `https://api.minimaxi.com/anthropic/v1` | ✅ 正确 | 主服务器用这个 |
| `https://api.minimax.io/anthropic/v1` | ❌ 错误 | 会导致 401 |

---

## 部署新服务器 checklist

新安装 OpenClaw 后，必须依次完成以下配置：

1. 安装 OpenClaw（用 npm overrides 解决 libsignal 问题）
2. 创建 `auth-profiles.json`
3. 创建 `models.json`（baseUrl 用 `api.minimaxi.com`）
4. 在 `openclaw.json` 中添加 `agents.defaults.model.primary`
5. 配置飞书渠道
6. 重启 gateway
7. 配对测试

---

## 已验证的修复案例

### 案例：116/117 初始部署时 baseUrl 错误
- **问题**：116/117 配置完成后一直报 `401 invalid api key`
- **排查**：从服务器 curl 测试发现 key 有效，但 baseUrl 错写成 `api.minimax.io`
- **解决**：`sed -i 's|api.minimax.io|api.minimaxi.com|g' models.json` 后重启
- **教训**：新部署服务器时，主服务器的 models.json baseUrl 是正确参考，但不要直接复制，要确认域名是否正确
