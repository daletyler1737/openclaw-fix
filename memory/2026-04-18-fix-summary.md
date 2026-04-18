# 2026-04-18 113/114 故障修复总结

## 故障现象

- 113 和 114 的飞书 bot 能接收消息、显示已读，但 bot 不回复
- 主服务器（dale003）正常

## 故障时间线

| 时间 | 事件 |
|------|------|
| 04:16 | 用户发现 113 不回复 |
| 04:18 | 尝试重启 113 gateway（发现残留进程问题） |
| 04:22 | 113 恢复 |
| 04:24 | 用户修复 114（同样是残留进程） |
| 04:34 | 113 报 "LLM idle timeout"（NVIDIA NIM 模型不可用） |
| 04:34 | 113 报错 "Missing API key for selected provider" |
| 04:46 | 两台都故障，114 报同样 Missing API key |

## 根因分析

### 问题 1：Gateway 残留进程（04:18, 04:24）
- 每次重启 gateway 时，旧进程未完全退出
- 两个 gateway 进程绑定同一端口，新消息无法正确分发
- **解决**：`pkill -9 -f openclaw` 强制杀掉后重启

### 问题 2：NVIDIA NIM API 不可访问（04:34）
- 113 和 114 都无法访问 `https://integrate.api.nvidia.com`（timeout）
- 两台都使用 NVIDIA NIM 作为默认模型
- **解决**：切换到 minimax API（`https://api.minimaxi.com/anthropic/v1`）

### 问题 3：auth-profiles.json 缺失（04:34 之后）
- 第一次修复只添加了 `models.json` 中的 provider 配置
- OpenClaw 的 API key 实际存储在 `auth-profiles.json` 中
- 两台都报错 `No API key found for provider "minimax"`
- **解决**：手动创建 auth-profiles.json，写入 minimax API key

## 修复步骤

### 113 修复（dale@192.168.1.113）

**1. 创建 auth-profiles.json**
```bash
python3 -c '
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
'
```

**2. 重启 gateway**
```bash
pkill -9 -f openclaw
sleep 2
nohup openclaw gateway > /dev/null 2>&1 &
sleep 25
```

### 114 修复（dale@192.168.1.114）

**1. 创建 auth-profiles.json（含 nvidia-nim + minimax）**
```bash
python3 -c '
import json
d = {
    "version": 1,
    "profiles": {
        "nvidia-nim:default": {
            "type": "api_key",
            "provider": "nvidia-nim",
            "key": "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"
        },
        "minimax:default": {
            "type": "api_key",
            "provider": "minimax",
            "key": "sk-cp-ymanZIbnWmh_CnFD_xrKpUjDARS2U1F9fZgwWk3NhM9ILq8B6ayyDAm-vi39Et-uiCXSTnjrJRV9IMoUroVOePgnNDee-NlIcnKz2mI2dYmDw1NSKFQcXOA"
        }
    }
}
with open("/home/dale/.openclaw/agents/main/agent/auth-profiles.json", "w") as f:
    json.dump(d, f, indent=2)
'
```

**2. 重启 gateway**
```bash
pkill -9 -f openclaw
sleep 2
nohup openclaw gateway > /dev/null 2>&1 &
sleep 25
```

## 两台服务器配置现状

### 113（192.168.1.113）
- Gateway auth token: `3d24ff0b43d3f796c5f10503f08881898224616b8dfad70f`
- 默认模型: `minimax/MiniMax-M2.7`
- 飞书 appId: `cli_a94ca8fd82b8dbd2`

### 114（192.168.1.114）
- Gateway auth token: `621ede5fde363053ec27eac5c7df602119a23fec375473d1`
- 默认模型: `minimax/MiniMax-M2.7`
- 飞书 appId（二太子）: `cli_a94c990d623c1bcd`

## 关键教训

1. **OpenClaw 的 API key 配置在两处**：
   - `models.json` — provider 的 baseUrl 和 model 定义
   - `auth-profiles.json` — 实际 API key（之前一直忽略了这个）

2. **Gateway 重启前先杀干净**：`pkill -9 -f openclaw` 确保没有残留进程

3. **NVIDIA NIM API** 在这些服务器上网络不可达，需要用 minimax 替代
