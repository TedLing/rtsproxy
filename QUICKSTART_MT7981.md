# 🚀 快速开始 - MT7981路由器编译指南

## 📌 前置准备

1. **Fork项目**（已完成）
   - 你的仓库：https://github.com/TedLing/rtsproxy

2. **配置GitHub Token**（用于自动上传Release）
   - 进入 Settings → Developer settings → Personal access tokens
   - 生成新Token，勾选 `repo` 权限
   - 在仓库 Settings → Secrets and variables → Actions 中添加：
     - Name: `GH_TOKEN`
     - Value: 你的Token值

## ⚡ 快速编译（3步完成）

### 方法一：通过Release触发（推荐）

```bash
# 1. 创建Git Tag
git tag v1.0.0
git push origin v1.0.0

# 2. 在GitHub上创建Release
# 访问：https://github.com/TedLing/rtsproxy/releases/new
# Tag选择: v1.0.0
# 点击 "Publish release"

# 3. 等待编译完成（约10-15分钟）
# 访问：https://github.com/TedLing/rtsproxy/actions
```

### 方法二：手动触发Workflow

```bash
# 1. 访问Actions页面
# https://github.com/TedLing/rtsproxy/actions

# 2. 左侧选择 "Build ImmortalWrt MT7981 Package"

# 3. 点击 "Run workflow" → 选择分支 → "Run workflow"

# 4. 等待编译完成
```

## 📥 获取编译产物

### 方式一：从Release下载（推荐）

```bash
# 访问最新Release
https://github.com/TedLing/rtsproxy/releases/latest

# 下载两个文件：
# - rtsproxy_v1.0.0_immortalwrt-mt7981.ipk
# - luci-app-rtsproxy_v1.0.0_all.ipk
```

### 方式二：从Artifact下载

```bash
# 1. 访问Actions → 最近的workflow运行
# 2. 页面底部找到 "rtsproxy-ipk-mt7981"
# 3. 点击下载
```

## 🔧 安装到MT7981路由器

### 一键安装脚本

```bash
#!/bin/bash
# install.sh - MT7981路由器一键安装脚本

ROUTER_IP="192.168.1.1"  # 修改为你的路由器IP
VERSION="v1.0.0"          # 修改为实际版本

echo "📤 上传IPK到路由器..."
scp rtsproxy_${VERSION}_immortalwrt-mt7981.ipk root@${ROUTER_IP}:/tmp/
scp luci-app-rtsproxy_${VERSION}_all.ipk root@${ROUTER_IP}:/tmp/

echo "🔌 连接到路由器并安装..."
ssh root@${ROUTER_IP} << 'EOF'
    echo "📦 安装rtsproxy本体..."
    opkg install /tmp/rtsproxy_*.ipk
    
    echo "🎨 安装LuCI界面..."
    opkg install /tmp/luci-app-rtsproxy_*.ipk
    
    echo "🗑️ 清理临时文件..."
    rm /tmp/rtsproxy_*.ipk /tmp/luci-app-rtsproxy_*.ipk
    
    echo "⚙️ 启用并启动服务..."
    /etc/init.d/rtsproxy enable
    /etc/init.d/rtsproxy start
    
    echo "✅ 安装完成！"
    echo ""
    echo "📊 服务状态："
    /etc/init.d/rtsproxy status
    
    echo ""
    echo "🌐 访问LuCI界面配置："
    echo "   http://${ROUTER_IP}/cgi-bin/luci/admin/services/rtsproxy"
EOF

echo "✨ 全部完成！"
```

### 手动安装步骤

```bash
# 1. 上传文件
scp rtsproxy_*.ipk root@192.168.1.1:/tmp/
scp luci-app-rtsproxy_*.ipk root@192.168.1.1:/tmp/

# 2. SSH登录
ssh root@192.168.1.1

# 3. 安装
opkg install /tmp/rtsproxy_*.ipk
opkg install /tmp/luci-app-rtsproxy_*.ipk

# 4. 启动
/etc/init.d/rtsproxy enable
/etc/init.d/rtsproxy start

# 5. 验证
/etc/init.d/rtsproxy status
ps | grep rtsproxy
netstat -tlnp | grep 8554
```

## 🎯 配置rtsproxy

### 通过LuCI界面（推荐）

1. 浏览器访问：`http://192.168.1.1`
2. 进入：**服务** → **RTSP Proxy**
3. 配置参数：
   - 监听端口：8554（默认）
   - Buffer Pool大小：8192
   - 日志级别：info
   - NAT模式：根据需要选择
4. 点击 **保存并应用**

### 通过配置文件

```bash
# SSH登录路由器
ssh root@192.168.1.1

# 编辑配置文件
vi /etc/rtsproxy/config.json

# 示例配置
{
    "port": 8554,
    "buffer_pool_block_size": 2048,
    "buffer_pool_count": 8192,
    "log_level": "info",
    "log_max_lines": 10000,
    "nat_enabled": false,
    "nat_method": "stun",
    "token": "",
    "listen_interface": "0.0.0.0",
    "upstream_interface": ""
}

# 重启服务
/etc/init.d/rtsproxy restart
```

## ✅ 验证安装

```bash
# 1. 检查服务状态
/etc/init.d/rtsproxy status

# 2. 查看进程
ps | grep rtsproxy

# 3. 检查端口
netstat -tlnp | grep 8554

# 4. 测试RTSP代理
# 使用VLC或其他播放器打开：
# rtsp://192.168.1.1:8554/你的RTSP流地址

# 5. 查看日志
logread | grep rtsproxy
```

## 🔍 故障排查

### 问题1：IPK安装失败

```bash
# 检查依赖
opkg update
opkg install libstdcpp libpthread

# 重新安装
opkg install --force-reinstall /tmp/rtsproxy_*.ipk
```

### 问题2：服务无法启动

```bash
# 查看详细错误
/etc/init.d/rtsproxy start
logread | grep rtsproxy

# 检查配置文件
cat /etc/rtsproxy/config.json

# 检查端口占用
netstat -tlnp | grep 8554
```

### 问题3：LuCI界面不显示

```bash
# 清除LuCI缓存
rm -rf /tmp/luci-*

# 重启uhttpd
/etc/init.d/uhttpd restart

# 刷新浏览器（Ctrl+F5）
```

### 问题4：编译失败

```bash
# 查看完整日志
# 访问Actions → 失败的workflow → 查看日志

# 常见原因：
# 1. 网络问题：重试workflow
# 2. 依赖缺失：检查Makefile
# 3. 代码错误：检查提交的代码
```

## 📊 编译时间参考

| 阶段 | 耗时 | 说明 |
|------|------|------|
| 克隆源码 | 2-5分钟 | 首次需要，后续使用缓存 |
| 更新Feeds | 1-2分钟 | - |
| 编译rtsproxy | 5-8分钟 | C++编译+Meson构建 |
| 编译LuCI | 1-2分钟 | - |
| **总计** | **10-15分钟** | 缓存命中后 |

## 💡 优化建议

### 1. 启用缓存加速

GitHub Actions会自动缓存ImmortalWrt SDK，第二次编译会快很多。

### 2. 定期清理旧Release

保留最近5-10个Release即可，删除旧的以节省空间。

### 3. 使用Tag命名规范

推荐使用语义化版本：
- `v1.0.0` - 主版本
- `v1.0.1` - 补丁版本
- `v1.1.0` - 功能版本

### 4. 监控编译状态

可以设置GitHub通知：
- Settings → Notifications
- 启用Email或Webhook通知

## 📞 获取帮助

- 📖 详细文档：[BUILD_IMMORTALWRT.md](BUILD_IMMORTALWRT.md)
- 📝 优化说明：[WORKFLOW_OPTIMIZATION.md](WORKFLOW_OPTIMIZATION.md)
- 🐛 提交Issue：https://github.com/TedLing/rtsproxy/issues
- 💬 讨论区：https://github.com/TedLing/rtsproxy/discussions

---

**祝你使用愉快！** 🎉
