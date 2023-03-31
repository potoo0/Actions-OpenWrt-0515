ESXi-7.0U3sd-19482531-ESIR-NVME-USBNIC-IGCNIC-4G-0501.iso: 

- [eSir](https://t.me/esirplayground)([Google Drive](https://drive.google.com/drive/folders/1dqNUrMf9n7i3y1aSh68U5Yf44WQ3KCuh)) 自定义补丁的固件, 基于 ESXi-7.0U3
- 集成usb和i225网卡和设置虚拟内存 4G。

> eSir 提醒: 为了获得最好的网络连接速度，请将网卡类型设置为VMXNET3

## 1. esxi

下载：登录 [VMware Customer Connect](https://customerconnect.vmware.com/cn/home) -> 产品与账户 -> [所有产品](https://customerconnect.vmware.com/cn/downloads/#all_products) -> esxi ... 下载产品

安装：

1. 注意在 *runweasel cdromBoot* 后要设置虚拟内存(默认是 128G，会占硬盘 128GB): `autoPartitionOSDataSize=4096`
2. ...



## 2. openwrt - lede

### 2.1 编译镜像

本次使用 github actions 编译。

> 涉及的仓库:
>
> - opw 源码 [lede](https://github.com/coolsnowwolf/lede)
> - github 云编译 [Actions-OpenWrt](https://github.com/P3TERX/Actions-OpenWrt)
> - opw 仓库 [openwrt-packages](https://github.com/kenzok8/openwrt-packages)

以下部分步骤参考 [Actions-OpenWrt](https://github.com/P3TERX/Actions-OpenWrt) 作者的博文 [使用 GitHub Actions 云编译 OpenWrt](https://p3terx.com/archives/build-openwrt-with-github-actions.html)。

> github actions 云编译 openwrt 不能直接使用 ssh 配置并编译，会报错：
>
> ```bash
>  make[3] -C toolchain/fortify-headers compile
>  make[3] -C toolchain/nasm compile
>  make[3] -C toolchain/gcc/initial compile
> make: *** wait: No child processes.  Stop.
> make: *** Waiting for unfinished jobs....
> make: *** wait: No child processes.  Stop.
> Error: Process completed with exit code 143.
> ```
>
> 所以此处把 配置 和 编译 步骤拆分。

1. 新建 Actions: [Actions-OpenWrt](https://github.com/P3TERX/Actions-OpenWrt) 仓库使用 **Use this template**；

2. 镜像自动上传: 修改 *.github/workflows/build-openwrt.yml*: *UPLOAD_WETRANSFER* 为 true。注意 CowTransfer 需要登录才能使用，此处不要再用；release 需要关联 tag，所以此处不再使用。

3. 添加 opw 插件仓库: 修改 *diy-part1.sh*, 追加:

   - ```bash
      sed -i '$a src-git kenzo https://github.com/kenzok8/openwrt-packages' feeds.conf.default
      sed -i '$a src-git small https://github.com/kenzok8/small' feeds.conf.default
      # https://github.com/jerrykuku/openwrt-package
      ```

4. 启动 actions: Actions 的 *Run Workflow* 时把 *SSH connection to Actions*的值改为 `true`;

5. 浏览器打开 https://tmate.io/t/Lxxxxxxxx, 如果是黑屏的话先按下 Q;

6. 开始配置：`cd openwrt && make menuconfig` (* 表示将打包到镜像中):

   1. cpu 架构，我这里保持 x86_64；
   2. luci: application 配置插件
      1. luci-app-adbyby-plus 广告屏蔽大师 Plus+
      2. luci-app-aliyundrive-webdav
      3. luci-app-argonne-config 主题设置
      4. luci-app-firewall 防火墙和端口转发
      5. 取消 luci-app-ssr-plus
      6. luci-app-ttyd
      7. luci-app-vssr
      8. 取消 luci-app-zerotier
   3. luci: Themes 配置主题:
      1. luci-theme-argonne 与 luci-app-argonne-config 搭配
   4. ipv6 扩展：Extra packages -> 选中 ipv6helper

7. 配置结束后（如果无法配置，可在本地 docker 安装基本依赖进行配置操作见附），把配置上传到网盘:

   1. ```bash
      # 安装 cli 工具
      curl -fsSL git.io/file-transfer | sh
      # 上传
      ./transfer wet .config
      ```

8. 上传结束后根据地址 cli 上传后的地址下载到本地。完成后执行 exit ，并到 Actions 里 cancel。

9. 上传 *.config* 到 仓库的根目录，Actions 的 *Run Workflow* 时把 *SSH connection to Actions*的值改为 `false` 开始编译。默认的两线程下耗时 1h50m。

10. 编译输出的文件:

    1. *config.buildinfo*: build 的差异配置，扩展为完整配置: `cp config.buildinfo .config && make defconfig`

    2. *\*.manifest*: 固件包含的软件以及版本

    3. *\*-squashfs-combined-efi.img*: efi 引导的固件, esxi 推荐使用此 img 转换

    4. *\*-squashfs-combined-efi.vmdk*: efi 引导的虚拟机 Vmware 文件，esxi 不要用此文件

    5. *\*-squashfs-rootfs.img*: 不带引导的固件，需要单独设置 grub/syslinux 来进行引导

    6. >[squashfs 说明](https://openwrt.org/docs/techref/filesystems): 一种只读的文件系统，每次修改: *actually a copy of it is being copied to the second (JFFS2) partition*

#### docker lede 基本环境

```bash
############ make lede config ############
# docker exec -it ubt bash

# container
cd ~
HTTP_PROXY='localhost:10809';HTTPS_PROXY='localhost:10809'

apt install -y git vim curl wget
apt install -y python3 python3-distutils file build-essential rsync unzip subversion libncurses5-dev zlib1g-dev gawk gcc-multilib flex git-core gettext libssl-dev  

git clone https://github.com/coolsnowwolf/lede openwrt
cd openwrt

sed -i '$a src-git kenzo https://github.com/kenzok8/openwrt-packages' feeds.conf.default
sed -i '$a src-git small https://github.com/kenzok8/small' feeds.conf.default

./scripts/feeds update -a
./scripts/feeds install -a

make menuconfig

# copy config to host
# docker cp ubt:/root/openwrt/.config ./
```



### 2.2 转为 vmdk

将 *squashfs-combined-efi.img* 使用 [starwind v2v converter](https://www.starwindsoftware.com/starwind-v2v-converter#download) (需要注册才能下载，需要安装) 转换 vmdk

> 注意：路径中不要有中文；如果有 vmware virtual disk development kit unrecoverable error 错误的话，更改 starwind 版本。

操作顺序：

1. local file > select *squashfs-combined-efi.img*
2. local file
3. vmdk
4. esxi server image
5. esxi pre-allocated image
6. 会生成两个 vmdk 文件，均需要用到

### 2.3 安装

#### 2.3.1 安装到 esxi

1. 自定义设置里 删除硬盘，添加硬盘 -> 现有硬盘 -> 上传上一步得到的两个 vmdk，上传完成后只会显示一个，选中作为虚拟机硬盘 -> 保存。
2. 保存后再次编辑此虚拟机：设置硬盘大小；虚拟机选项 -> 引导选项 改为 bios

esxi 硬件直通:

1. 管理 -> 硬件 -> PCI设备，安装自己设备型号查找，例如此处打开网卡直通，找到最后两个 I255 网卡，表格前端勾选，表格左上角 “切换直通”，点击后等待几秒。表格 “直通” 列显示为 *活动* 表明已切换。
2. 虚拟机使用直通设备：虚拟机设置 -> 添加其他设备 -> PCI 设备，PCI 设备行选择具体设备，例如此处选择 I255 网卡。

esxi 虚拟机开机自启:

1. 管理 -> 自动启动，修改 “已启用” 为 *是*，“启动延时” 我这里改为 *10s*；
2. 下面虚拟机中选择并配置。

如果需要 openwrt 下的 ip 能访问 esxi:

1. openwrt 虚拟机的网络配置器加上 esxi 的管理口;
2. esxi 虚拟交换机的安全中 “混杂模式”、“MAC地址更改”、“伪传输” 改为  *接受*；
3. openwrt 的 lan 口加入 esxi 的管理口。

#### 2.3.2 直接安装

对于 emmc 存储，lede 源码中已经默认支持，所以上面编译的镜像直接使用 *\*-squashfs-combined-efi.img* 即可。

此处使用 pe 将镜像 写盘，注意如果 openwrt 镜像是 64 位，那 pe 也需要 64 位，步骤:

1. U 盘安装 [ventoy](https://github.com/ventoy/Ventoy/releases)
2. [wepe](https://www.wepe.com.cn/download.html) 导出镜像，放到 ventoy 分出的 U 盘里
3. openwrt 引导到 wepe
4. 使用 wepe 的 diskgenius 将目标盘 删除所有分区
5. win+e 查看 u 盘的盘符
6. win+r 输入 cmd 切换到 u 盘。如u 盘的盘符为 c 盘，则输入: `c:` 
7. 输入 `physdiskwrite -u your-image.img`，按提示选择目标盘...
8. 如果遇到类似错误 *written write error after 1048576 bytes* 或 *error #5 occurred while writing to disk at sector 2048* 则尝试对目标盘重新分盘以及格式化；尝试不要 ventoy 引导 wepe，wepe 写为 u 盘启动盘...

没有鼠标键盘常用快捷键操作:

1. win+d 返回到桌面使用 tab 选择到任一桌面图标，上下左右移动，enter 启动；
2. 软件启动后使用 tab 切换选择，alt 选择工具栏的菜单，左右选择菜单项...
3. win+r 启动; win+x 系统快速菜单（打开 cmd 等等）
4. shift+tab 为 tab 的逆向。



### 2.4 基本设置

1. ```bash
   # 修改 ip
   vim /etc/config/network
   
   # 修改 lan 下的地址
   # ipaddr: 192.168.1.1 -> 192.168.2.200
   ```

2. `passwd` 修改密码

3. `reboot` 重启

## 3. ipv6 设置

> ipv6 地址段说明:
>
> - `2000::/3`: 全球单播，即公网地址
>   - `240e::/18`: 电信
>   - `2408:8000::/20`: 联通
>   - `2409:8000::/20`: 移动
> - `fc::/7`: 局域网
> - `fe80::/10`: 链路本地，必须有该地址，自动生成
> - `::1/128`: 本地回环

wan:

- 高级设置: 内置的 IPV6 管理 -> 勾选；获取IPv6 -> 自动

lan:

- 基本设置: IPv6 分配长度 >= 运营商，一般都可以取 64
- 高级设置: 内置的 IPV6 管理 -> 不勾选
- 下方IPv6设置: 路由通告 -> 服务器模式；DHCPv6服务 -> 服务器模式；NDP代理 -> 禁用；DHCPv6 模式 -> 无 + 有

全局网络选项:

- IPv6 ULA 前缀清空

DHCP/DNS:

- 高级设置: 禁止解析 IPv6 DNS 记录 -> 不勾选

### 防火墙

只允许特定 ip 和端口

基本设置:

- 基本设置: 转发 -> 拒绝
- 区域: wan 下的入站, 转发 为拒绝

#### 方式一: 通信规则

打开路由器端口下点击添加:

- 限制地址 -> 仅 IPv6
- 源区域 -> wan
- 源xxx 保持所有
- 目标区域 -> lan
- 目标地址 -> `::1:2:3:4/::ffff:ffff:ffff:ffff` 注: 选择重启不变的后几部分, 我这里是后四部分，斜杠后的 ffff 个数与前面一致
- 目标端口 -> `80 443 8000-9999` 注: 多个端口使用空格
- 动作 -> 接受

#### 方式二: 自定义规则

使用命令的方式配置，方便快速添加和编辑。

末尾输入:

```bash
# my pc
ip6tables -t filter -A zone_wan_forward -p tcp -d ::1:2:3:4/::ffff:ffff:ffff:ffff -m multiport --dports 80,443,8000:9999 -m comment --comment "ipv6-firewall" -j zone_lan_dest_ACCEPT
ip6tables -t filter -A zone_wan_forward -p udp -d ::1:2:3:4/::ffff:ffff:ffff:ffff -m multiport --dports 80,443,8000:9999 -m comment --comment "ipv6-firewall" -j zone_lan_dest_ACCEPT
```

上面命令其实是从方式一界面设置导出的命令，导出步骤:

- 查看 ipv6 防火墙规则列表: `ip6tables -nL --line-numbers`
- 将 ipv6 防火墙规则显示成命令: `fw3 -6 print`
- 仅仅目的端口不同可以合并为一条，例如: `-m multiport --dports `

