# OpenClaw 113/114 故障修复手册

## 服务器连接信息

| 机器 | IP | 用户 | 密码 | Gateway 端口 |
|------|-----|------|------|------|
| 113 | 192.168.1.113 | dale | 173784 | 18789 |
| 114 | 192.168.1.114 | dale | 814852896 | 18789 |

## 快速检查命令

```bash
# 检查进程状态
ps aux | grep openclaw | grep -v grep

# 检查 gateway health
curl -s http://localhost:18789/health

# 查看 recent 日志
tail -30 /tmp/openclaw-1000/openclaw-2026-04-18.log
```

## 常见故障与解决方案

### 1. Bot 能已读但不回复（残留进程）

**症状**：消息已读但无回复，`ps aux` 显示多个 openclaw 进程

**修复**：
```bash
pkill -9 -f openclaw
sleep 2
nohup openclaw gateway > /dev/null 2>&1 &
sleep 25
curl -s http://localhost:18789/health
```

### 2. "No API key found for provider XXX"

**症状**：报错 `No API key found for provider "minimax"` 或其他 provider

**原因**：OpenClaw API key 存在 `auth-profiles.json`，不是只存在 `models.json`

**修复 - minimax provider**：
```python
import json
with open("/home/dale/.openclaw/agents/main/agent/auth-profiles.json", "r") as f:
    d = json.load(f)
d["profiles"]["minimax:default"] = {
    "type": "api_key",
    "provider": "minimax",
    "key": "sk-cp-ymanZIbnWmh_CnFD_xrKpUjDARS2U1F9fZgwWk3NhM9ILq8B6ayyDAm-vi39Et-uiCXSTnjrJRV9IMoUroVOePgnNDee-NlIcnKz2mI2dYmDw1NSKFQcXOA"
}
with open("/home/dale/.openclaw/agents/main/agent/auth-profiles.json", "w") as f:
    json.dump(d, f, indent=2)
```

**然后重启 gateway**：
```bash
pkill -9 -f openclaw
sleep 2
nohup openclaw gateway > /dev/null 2>&1 &
```

### 3. NVIDIA NIM API timeout

**症状**：报错 `timedOut: true` 或 LLM idle timeout

**原因**：NVIDIA NIM API (`integrate.api.nvidia.com`) 从这些服务器网络不可达

**解决**：切换到 minimax provider
```bash
# 修改默认模型
python3 -c '
import json
with open("/home/dale/.openclaw/openclaw.json", "r") as f:
    d = json.load(f)
d["agents"]["defaults"]["model"]["primary"] = "minimax/MiniMax-M2.7"
with open("/home/dale/.openclaw/openclaw.json", "w") as f:
    json.dump(d, f, indent=2)
'
```

### 4. 模型 401 Invalid Authentication

**症状**：报错 `401 Invalid Authentication`

**原因**：该 provider 的 API key 已失效

**解决**：切到其他可用的 provider（如 minimax）

## API Key 存储位置

OpenClaw 的 provider API key 配置在**两处**，缺一不可：

| 文件 | 用途 |
|------|------|
| `~/.openclaw/agents/main/agent/models.json` | provider 的 baseUrl、model ID 定义 |
| `~/.openclaw/agents/main/agent/auth-profiles.json` | 实际 API key |

## 关键文件路径

| 文件 | 113 路径 | 114 路径 |
|------|----------|----------|
| openclaw.json | `/home/dale/.openclaw/openclaw.json` | `/home/dale/.openclaw/openclaw.json` |
| models.json | `/home/dale/.openclaw/agents/main/agent/models.json` | 同上 |
| auth-profiles.json | `/home/dale/.openclaw/agents/main/agent/auth-profiles.json` | 同上 |
| Gateway 日志 | `/tmp/openclaw-1000/openclaw-2026-04-18.log` | 同上 |
| Gateway auth token | `3d24ff0b43d3f796c5f10503f08881898224616b8dfad70f` | `621ede5fde363053ec27eac5c7df602119a23fec375473d1` |

## minimax API 验证

从 113 测试 minimax API 是否可达：
```bash
curl -s --connect-timeout 10 -m 20 -X POST 'https://api.minimaxi.com/anthropic/v1/messages' \
  -H 'Content-Type: application/json' \
  -H 'x-api-key: sk-cp-ymanZIbnWmh_CnFD_xrKpUjDARS2U1F9fZgwWk3NhM9ILq8B6ayyDAm-vi39Et-uiCXSTnjrJRV9IMoUroVOePgnNDee-NlIcnKz2mI2dYmDw1NSKFQcXOA' \
  -H 'anthropic-version: 2023-06-01' \
  -d '{"model":"MiniMax-M2.7","max_tokens":10,"messages":[{"role":"user","content":"hi"}]}'
```
