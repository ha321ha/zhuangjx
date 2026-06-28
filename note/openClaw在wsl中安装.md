


## 安装

脚本会自动：检测系统 → 装 Node（若缺）→ 装 OpenClaw → 启动 onboarding

```
curl -fsSL https://openclaw.ai/install.sh | bash
```

## cc switch配置openclaw配置目录

### wsl中查找配置目录

```
ls -la ~/.openclaw
cd ~
pwd
```

cc switch - 高级 - 目录配置 -openclaw

```
\\wsl$\Ubuntu-24.04\home\zc\.openclaw
```

## 设置令牌

```
openclaw config set gateway.auth.token "Zc220789"
```

## 重启

```
openclaw gateway restart
```

## 设置基本规则

```
openClaw基本规则
1、配置（openclaw配置，window配置）、应用或脚本（window应用、python、nodejs）等变动（新增、修改、删除），
需要征求用户同意，也要记录好变更记录。
2、操作window浏览器edge用browser工具。
3、操作window用interop（微软官方设计的 WSL-Windows 互操作机制）。
```


