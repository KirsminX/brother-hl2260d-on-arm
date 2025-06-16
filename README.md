# 在 ARM64 设备上为 Brother-HL2260D 打印机添加网络功能

_To read English Version, Click this：[English](./README_EN.md)_

## 序
在 AMD64/x32 设备上，这是一个简单的工作，只需要使用官方工具安装即可。在 ARM64 设备上，官方并没有提供驱动程序。在尝试使用 Docker 构建失败后，我参考文章 [1] 成功在香橙派 Zero3 上部署了网络功能。

如果你不想要自行编译 deb 包，可以直接从 Release 下载，见 _使用_ 章节

## 使用

- 下载编译好的 deb 包
```bash
wget https://github.com/KirsminX/brother-hl2260d-on-arm/releases/download/0.0.1/chl2260dcupswrapper-3.2.0-1.armhf.deb
wget https://github.com/KirsminX/brother-hl2260d-on-arm/releases/download/0.0.1/hl2260dlpr-3.2.0-1.armhf.deb
```
- 安装依赖
```bash
sudo apt update
sudo apt install psutils cups libc6 libstdc++6 libusb-1.0-0 gsfonts ghostscript -y
```
- 安装驱动
```bash
sudo dpkg -i 
chl2260dcupswrapper-3.2.0-1.armhf.deb 
hl2260dlpr-3.2.0-1.armhf.deb 
sudo systemctl restart cups
```
- 访问 https://<IP>:631/admin，删除自动添加的打印机，重新手动添加

## 下载文件

在 [Brother 驱动下载页面](https://support.brother.com/g/b/downloadlist.aspx?c=cn&lang=zh&prod=hl2260d_cn&os=128) 中下载相关文件。

### 文件列表

| 文件名 | 注释 |
|--------|------|
| **原始文件** | |
| `hl2260dcupswrapper-3.2.0-1.i386.deb` | CUPS 驱动 |
| `hl2260dlpr-3.2.0-1.i386.deb` | LPR 驱动 |
| **兼容文件** | |
| `brgenprintml2pdrv-4.0.0-1.armhf.deb` | Brother ARM 通用驱动文件 |
| `brhl2270dwcups_src-2.0.4-2.tar.gz` | brcupsconfig4 源码 |

## 构建镜像

### 解压并修改 LPR 驱动

```bash
dpkg -x hl2260dlpr-3.2.0-1.i386.deb hl2260dlpr-3.2.0-1.armhf.ext
dpkg-deb -e hl2260dlpr-3.2.0-1.i386.deb hl2260dlpr-3.2.0-1.armhf.ext/DEBIAN
sed -i 's/Architecture: i386/Architecture: armhf/' hl2260dlpr-3.2.0-1.armhf.ext/DEBIAN/control
echo true > hl2260dlpr-3.2.0-1.armhf.ext/opt/brother/Printers/HL2260D/inf/braddprinter
```

> [!TIP]  
> 原文提到的路径 `/usr/local/Brother/Printer/HL2260D/inf/braddprinter` 实际应为 `/opt/brother/Printers/HL2260D/inf/braddprinter`

### 复制 ARM 通用驱动

```bash
dpkg -x brgenprintml2pdrv-4.0.0-1.armhf.deb brgenprintml2pdrv-4.0.0-1.armhf.ext
cp brgenprintml2pdrv-4.0.0-1.armhf.ext/opt/brother/Printers/BrGenPrintML2/lpd/armv7l/rawtobr3 \
   hl2260dlpr-3.2.0-1.armhf.ext/opt/brother/Printers/HL2260D/lpd
```

### 重新打包 LPR 驱动

```bash
cd hl2260dlpr-3.2.0-1.armhf.ext
find . -type f ! -regex '.*.hg.*' ! -regex '.*?debian-binary.*' ! -regex '.*?DEBIAN.*' -printf '%P ' | xargs md5sum > DEBIAN/md5sums
cd ..
chmod 755 hl2260dlpr-3.2.0-1.armhf.ext/DEBIAN/p* \
          hl2260dlpr-3.2.0-1.armhf.ext/opt/brother/Printers/HL2260D/inf/* \
          hl2260dlpr-3.2.0-1.armhf.ext/opt/brother/Printers/HL2260D/lpd/*
dpkg-deb -b hl2260dlpr-3.2.0-1.armhf.ext hl2260dlpr-3.2.0-1.armhf.deb
```

### 编译 brcupsconfig4

```bash
sudo apt install gcc-9-arm-linux-gnueabihf libc6-dev-armhf-cross -y
tar zxvf brhl2270dwcups_src-2.0.4-2.tar.gz
cd brhl2270dwcups_src-2.0.4-2
arm-linux-gnueabihf-gcc-9 brcupsconfig3/brcupsconfig.c -o brcupsconfig4
```

### 构建 Cupswrapper

```bash
dpkg -x hl2260dcupswrapper-3.2.0-1.i386.deb hl2260dcupswrapper-3.2.0-1.armhf.ext
dpkg-deb -e hl2260dcupswrapper-3.2.0-1.i386.deb hl2260dcupswrapper-3.2.0-1.armhf.ext/DEBIAN
sed -i 's/Architecture: i386/Architecture: armhf/' hl2260dcupswrapper-3.2.0-1.armhf.ext/DEBIAN/control
cp brhl2270dwcups_src-2.0.4-2/brcupsconfig4 hl2260dcupswrapper-3.2.0-1.armhf.ext/opt/brother/Printers/HL2260D/cupswrapper

cd hl2260dcupswrapper-3.2.0-1.armhf.ext
find . -type f ! -regex '.*.hg.*' ! -regex '.*?debian-binary.*' ! -regex '.*?DEBIAN.*' -printf '%P ' | xargs md5sum > DEBIAN/md5sums
cd ..

chmod 755 hl2260dcupswrapper-3.2.0-1.armhf.ext/DEBIAN/p* \
          hl2260dcupswrapper-3.2.0-1.armhf.ext/opt/brother/Printers/HL2260D/cupswrapper/*
dpkg-deb -b hl2260dcupswrapper-3.2.0-1.armhf.ext chl2260dcupswrapper-3.2.0-1.armhf.deb
```

## 安装

```bash
sudo dpkg --add-architecture armhf
sudo apt update
sudo apt install libc6:armhf
sudo apt install psutils cups
sudo dpkg -i chl2260dcupswrapper-3.2.0-1.armhf.deb hl2260dlpr-3.2.0-1.armhf.deb
```

> [!WARNING]  
> 当前已知问题：  
> - 自动添加的打印机无法使用，请访问 https://<ip>:631/admin 删除原有打印机、重新添加。

> [!TIP]  
> 如果你是 Ubuntu 用户，建议卸载 AppArmor，或者配置文件放行。懒得折腾就直接卸了吧，反正我不用 snap  
> 最后，删除所有临时文件，使用浏览器打开 [https://你的IP:631](https://你的IP:631)，按照提示添加打印机。祝顺利！本文用时 2 周

[1]: [@alexivkin Brother printer drivers for Raspberry Pi and other ARM devices](https://github.com/alexivkin/brother-in-arms)
```