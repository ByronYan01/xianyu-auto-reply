# Mac 本地 24 小时运行配置指南

本文档用于将闲鱼自动回复系统配置为 Mac 本地 24 小时运行。

---

## 一、系统设置（一次性配置）

### 1.1 关闭系统睡眠

```bash
# 终端执行 - 防止 Mac 进入睡眠
sudo pmset -a sleep 0
sudo pmset -a disksleep 0
sudo pmset -a displaysleep 0
```

### 1.2 苹果菜单设置

```
苹果菜单 → 系统设置 → 锁屏：
├── 关闭"显示器进入睡眠"
└── 关闭"开始使用电池电源时进入睡眠"
```

### 1.3 电源设置

```
苹果菜单 → 系统设置 → 电源 → 电源适配器：
├── 开启"防止自动进入睡眠"
├── 关闭"启用 Power Nap"
└── 关闭"让硬盘进入睡眠"
```

### 1.4 显示器设置

```
苹果菜单 → 系统设置 → 显示器：
└── 关闭"优化能源消耗"
```

---

## 二、创建服务自启动

### 2.1 创建 LaunchAgent 服务文件

```bash
# 将项目路径替换为你的实际路径
PROJECT_PATH="/path/to/xianyu-auto-reply"

cat > ~/Library/LaunchAgents/com.xianyu-auto-reply.plist << EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.xianyu-auto-reply</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>-c</string>
        <string>cd ${PROJECT_PATH} && docker compose up -d</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>StartInterval</key>
    <integer>60</integer>
</dict>
</plist>
EOF
```

### 2.2 加载服务

```bash
launchctl load ~/Library/LaunchAgents/com.xianyu-auto-reply.plist
```

### 2.3 验证服务状态

```bash
launchctl list | grep xianyu
```

---

## 三、配置定期重启

### 3.1 创建优雅关闭脚本

```bash
# 将项目路径替换为你的实际路径
PROJECT_PATH="/path/to/xianyu-auto-reply"

cat > ~/graceful_shutdown.sh << EOF
#!/bin/bash
echo "\$(date): 优雅关闭闲鱼服务..."
cd ${PROJECT_PATH}
docker compose stop
echo "\$(date): 服务已停止，准备重启..."
EOF

chmod +x ~/graceful_shutdown.sh
```

### 3.2 设置定时重启（编辑 crontab）

```bash
crontab -e

# 按 i 进入编辑模式，添加以下内容：
# 每天凌晨 3:55 优雅关闭，4:00 重启
55 3 * * * /Users/$(whoami)/graceful_shutdown.sh
0 4 * * * sudo shutdown -r now

# 按 ESC 退出编辑，输入 :wq 保存
```

### 3.3 或者用 launchd 替代 crontab（更稳定）

创建定时任务 plist：

```bash
cat > ~/Library/LaunchAgents/com.xianyu-auto-reply-scheduled.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.xianyu-auto-reply-scheduled</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>/Users/$(whoami)/graceful_shutdown.sh</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Hour</key>
        <integer>3</integer>
        <key>Minute</key>
        <integer>55</integer>
    </dict>
</dict>
</plist>
EOF

# 添加定时重启脚本
cat > ~/scheduled_reboot.sh << 'EOF'
#!/bin/bash
sudo shutdown -r now
EOF
chmod +x ~/scheduled_reboot.sh

# 添加另一条定时任务管理重启
# 注：shutdown 需要管理员权限，可能需要输入密码
```

### 3.4 立即测试定时重启

```bash
# 测试模式：1分钟后重启（谨慎使用）
# sudo shutdown -r +1

# 取消定时重启
sudo shutdown -c
```

---

## 四、创建网络监控脚本（可选）

### 4.1 创建监控脚本

```bash
PROJECT_PATH="/path/to/xianyu-auto-reply"

cat > ~/watchdog.sh << EOF
#!/bin/bash
LOG_FILE="/Users/\$(whoami)/watchdog.log"

while true; do
    if ! ping -c 1 8.8.8.8 > /dev/null 2>&1; then
        echo "\$(date): 网络断开，等待重连..." >> \$LOG_FILE
        sleep 30
    else
        # 检查 Docker 服务是否运行
        if ! docker ps | grep -q xianyu-backend-web; then
            echo "\$(date): 服务停止，重启中..." >> \$LOG_FILE
            cd ${PROJECT_PATH} && docker compose restart
        fi
    fi
    sleep 60
done
EOF

chmod +x ~/watchdog.sh
```

### 4.2 启动监控

```bash
nohup ~/watchdog.sh > ~/watchdog.log 2>&1 &
```

### 4.3 设置监控自启动

```bash
cat > ~/Library/LaunchAgents/com.xianyu-auto-reply-watchdog.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.xianyu-auto-reply-watchdog</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/$(whoami)/watchdog.sh</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
EOF

launchctl load ~/Library/LaunchAgents/com.xianyu-auto-reply-watchdog.plist
```

---

## 五、常用命令

### 5.1 服务管理

```bash
# 查看服务状态
launchctl list | grep xianyu
docker compose -C /path/to/xianyu-auto-reply ps

# 启动服务
cd /path/to/xianyu-auto-reply && docker compose up -d

# 停止服务
cd /path/to/xianyu-auto-reply && docker compose stop

# 重启服务
cd /path/to/xianyu-auto-reply && docker compose restart

# 查看日志
docker compose -C /path/to/xianyu-auto-reply logs -f
```

### 5.2 服务卸载（如果需要）

```bash
# 卸载自启动服务
launchctl unload ~/Library/LaunchAgents/com.xianyu-auto-reply.plist
launchctl unload ~/Library/LaunchAgents/com.xianyu-auto-reply-watchdog.plist
launchctl unload ~/Library/LaunchAgents/com.xianyu-auto-reply-scheduled.plist

# 删除服务文件
rm ~/Library/LaunchAgents/com.xianyu-auto-reply.plist
rm ~/Library/LaunchAgents/com.xianyu-auto-reply-watchdog.plist
rm ~/Library/LaunchAgents/com.xianyu-auto-reply-scheduled.plist
```

### 5.3 系统设置恢复

```bash
# 恢复系统睡眠设置（如果需要）
sudo pmset -a sleep 1
sudo pmset -a disksleep 1
sudo pmset -a displaysleep 1

# 取消定时重启
sudo shutdown -c
sudo crontab -e  # 删除相关行
```

---

## 六、重启流程

```
┌─────────────────────────────────────────────────────────────┐
│                    重启时间线（默认凌晨）                     │
├─────────────────────────────────────────────────────────────┤
│  03:55  →  优雅关闭 Docker 服务                             │
│  04:00  →  Mac 自动重启                                     │
│  04:05  →  launchd 自动启动 Docker Compose                 │
│  04:05  →  launchd 自动启动网络监控                         │
│  04:10  →  所有服务就绪                                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 七、故障排查

### 7.1 服务没有自启动

```bash
# 检查服务是否加载
launchctl list | grep xianyu

# 重新加载服务
launchctl unload ~/Library/LaunchAgents/com.xianyu-auto-reply.plist
launchctl load ~/Library/LaunchAgents/com.xianyu-auto-reply.plist

# 检查 Docker 是否运行
docker info
```

### 7.2 查看日志

```bash
# 查看 Docker 服务日志
docker compose -C /path/to/xianyu-auto-reply logs -f

# 查看监控日志
cat ~/watchdog.log
```

### 7.3 手动恢复服务

```bash
cd /path/to/xianyu-auto-reply
docker compose down
docker compose up -d
```

---

## 八、注意事项

| 项目 | 说明 |
|------|------|
| **电费** | Mac 24h 开着约 100W，月电费约 ¥50-100 |
| **硬件损耗** | M 系列芯片损耗低，Intel 机器注意散热 |
| **网络** | 家庭宽带可能断线，监控脚本会自动恢复 |
| **系统更新** | 设置"今夜稍后"更新，避免自动重启 |
| **密码** | crontab 中的 sudo 需要管理员密码 |

---

## 九、配置清单

执行完本文档后，你应该完成以下配置：

- [ ] 关闭系统睡眠
- [ ] 创建服务自启动 plist
- [ ] 验证服务自动启动
- [ ] 创建优雅关闭脚本
- [ ] 设置定时重启
- [ ] （可选）创建网络监控脚本
- [ ] 测试定时重启

---

## 十、快速开始

如果你已经配置过，只需要在新 Mac 上执行：

```bash
# 1. 修改 PROJECT_PATH 为实际路径后执行
PROJECT_PATH="/path/to/xianyu-auto-reply"

# 2. 创建并加载自启动服务
cat > ~/Library/LaunchAgents/com.xianyu-auto-reply.plist << EOF
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.xianyu-auto-reply</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/bash</string>
        <string>-c</string>
        <string>cd ${PROJECT_PATH} && docker compose up -d</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
EOF

launchctl load ~/Library/LaunchAgents/com.xianyu-auto-reply.plist

# 3. 关闭系统睡眠
sudo pmset -a sleep 0
sudo pmset -a disksleep 0
sudo pmset -a displaysleep 0
```