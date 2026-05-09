# ImmortalWrt MT798x 编译指南

本指南说明如何为基于 [immortalwrt-mt798x-6.6-padavanonly](https://github.com/Yuzhii0718/immortalwrt-mt798x-6.6-padavanonly.git) 的MT7981路由器编译rtsproxy。

## 前置要求

- Ubuntu 20.04/22.04 LTS 或 Debian 11/12
- 至少 20GB 可用磁盘空间
- 稳定的网络连接（需要下载大量依赖）

## 安装依赖

```bash
sudo apt-get update
sudo apt-get install -y \
    build-essential libncurses5-dev gawk git gettext libssl-dev \
    xsltproc wget unzip qemu-utils zstd python3-pyelftools \
    meson ninja-build
```

## 方法一：使用GitHub Actions自动编译（推荐）

项目已配置GitHub Actions工作流，会自动为immortalwrt-mt7981编译IPK包。

1. Fork本项目
2. 在GitHub上触发workflow（Release或手动触发）
3. 从Actions页面下载编译好的IPK包

编译产物命名格式：
- `rtsproxy_<version>_immortalwrt-immortalwrt-mt7981.ipk`
- `luci-app-rtsproxy_<version>_all.ipk`

## 方法二：本地手动编译

### 1. 克隆ImmortalWrt源码

```bash
git clone --depth 1 --branch main https://github.com/Yuzhii0718/immortalwrt-mt798x-6.6-padavanonly.git immortalwrt-mt798x
cd immortalwrt-mt798x
```

### 2. 更新Feeds

```bash
./scripts/feeds update -a
./scripts/feeds install -a
```

### 3. 准备rtsproxy包

```bash
# 创建包目录
mkdir -p package/rtsproxy
mkdir -p package/luci-app-rtsproxy

# 复制rtsproxy本体文件
cp /path/to/rtsproxy/openwrt/rtsproxy/Makefile package/rtsproxy/
cp -r /path/to/rtsproxy/openwrt/rtsproxy/files package/rtsproxy/
cp -r /path/to/rtsproxy/src package/rtsproxy/
cp -r /path/to/rtsproxy/include package/rtsproxy/
cp -r /path/to/rtsproxy/webui package/rtsproxy/
cp /path/to/rtsproxy/meson.build package/rtsproxy/
cp /path/to/rtsproxy/meson_options.txt package/rtsproxy/

# 复制LuCI界面文件
cp /path/to/rtsproxy/openwrt/luci-app-rtsproxy/Makefile package/luci-app-rtsproxy/
cp -r /path/to/rtsproxy/openwrt/luci-app-rtsproxy/root package/luci-app-rtsproxy/
cp -r /path/to/rtsproxy/openwrt/luci-app-rtsproxy/htdocs package/luci-app-rtsproxy/ 2>/dev/null || true
```

### 4. 修复换行符（Windows用户必需）

```bash
find package/rtsproxy package/luci-app-rtsproxy -type f -exec sed -i 's/\r$//' {} +
```

### 5. 配置编译选项

```bash
# 启用rtsproxy和LuCI支持
echo "CONFIG_PACKAGE_rtsproxy=y" >> .config
echo "CONFIG_PACKAGE_luci-app-rtsproxy=y" >> .config

# 生成完整配置
make defconfig
```

### 6. 编译

```bash
# 编译rtsproxy本体
make package/rtsproxy/compile V=s

# 编译LuCI界面
make package/luci-app-rtsproxy/compile V=s
```

### 7. 查找编译产物

编译成功后，IPK包位于：

```bash
bin/packages/<architecture>/base/rtsproxy_*.ipk
bin/packages/<architecture>/base/luci-app-rtsproxy_*.ipk
```

通常在 `bin/packages/aarch64_cortex-a53/base/` 或类似路径下。

## 安装到路由器

### 方法一：通过SCP上传并安装

```bash
# 上传IPK到路由器
scp rtsproxy_*.ipk root@192.168.1.1:/tmp/
scp luci-app-rtsproxy_*.ipk root@192.168.1.1:/tmp/

# SSH登录路由器
ssh root@192.168.1.1

# 安装（先装本体，再装LuCI）
opkg install /tmp/rtsproxy_*.ipk
opkg install /tmp/luci-app-rtsproxy_*.ipk

# 清理临时文件
rm /tmp/rtsproxy_*.ipk /tmp/luci-app-rtsproxy_*.ipk

# 启动服务
/etc/init.d/rtsproxy enable
/etc/init.d/rtsproxy start
```

### 方法二：通过LuCI界面安装

1. 登录LuCI管理界面
2. 进入 **系统** → **软件包**
3. 点击 **上传软件包**
4. 先上传并安装 `rtsproxy_*.ipk`
5. 再上传并安装 `luci-app-rtsproxy_*.ipk`
6. 刷新页面后在 **服务** 菜单中找到rtsproxy配置

## 常见问题

### Q1: 编译时提示找不到meson或ninja

```bash
# 在ImmortalWrt源码目录中安装
./scripts/feeds install meson ninja

# 或者在系统中安装
sudo apt-get install meson ninja-build
```

### Q2: IPK安装时提示依赖不满足

确保先安装了基础依赖：

```bash
opkg update
opkg install libstdcpp libpthread
```

### Q3: 编译失败，提示架构不匹配

检查 `.config` 中的目标架构设置：

```bash
make menuconfig
# 确认 Target System 和 Subtarget 正确设置为 MediaTek Ralink ARM
```

### Q4: LuCI界面不显示

1. 确认已安装luci-app-rtsproxy
2. 清除浏览器缓存
3. 重启uhttpd：`/etc/init.d/uhttpd restart`
4. 检查LuCI菜单配置：`ls /usr/lib/lua/luci/controller/`

## Makefile关键配置说明

### rtsproxy/Makefile

```makefile
PKGARCH:=*          # 允许在所有架构上安装
SUBMENU:=Web Servers/Proxies  # 在LuCI中分类显示
MESON_ARGS += \     # Meson编译优化参数
    -Dbuildtype=release \
    -Dstrip=true \
    -Db_lto=true
```

### luci-app-rtsproxy/Makefile

```makefile
LUCI_PKGARCH:=all   # LuCI包是架构无关的
LUCI_DEPENDS:=+rtsproxy  # 自动依赖rtsproxy本体
```

## 验证安装

安装完成后，可以通过以下方式验证：

```bash
# 检查服务状态
/etc/init.d/rtsproxy status

# 查看版本信息
rtsproxy --help

# 检查进程是否运行
ps | grep rtsproxy

# 查看监听端口（默认8554）
netstat -tlnp | grep 8554
```

## 技术支持

如遇问题，请提交Issue到：
- GitHub: https://github.com/plsy1/rtsproxy/issues
- ImmortalWrt MT798x: https://github.com/Yuzhii0718/immortalwrt-mt798x-6.6-padavanonly/issues
