Reply contains only the translated content. Keep the original format, translate to English

# Add Network Functionality for Brother-HL2260D Printer on ARM64 Devices

_To read Chinese Version, Click this: [Chinese](./README.md)_

## Introduction
On AMD64/x32 devices, this is a simple task that only requires using the official tool for installation. On ARM64 devices, the official driver is not provided. After failing to build with Docker, I referred to article [1] and successfully deployed the network functionality on Orange Pi Zero3.

If you don't want to build the deb package yourself, you can directly download it from Release, see the _Usage_ section.

## Usage

- Download the pre-built deb packages
```bash
wget https://github.com/KirsminX/brother-hl2260d-on-arm/releases/download/0.0.1/chl2260dcupswrapper-3.2.0-1.armhf.deb  
wget https://github.com/KirsminX/brother-hl2260d-on-arm/releases/download/0.0.1/hl2260dlpr-3.2.0-1.armhf.deb  
```
- Install dependencies
```bash
sudo apt update
sudo apt install psutils cups libc6 libstdc++6 libusb-1.0-0 gsfonts ghostscript -y
```
- Install drivers
```bash
sudo dpkg -i 
chl2260dcupswrapper-3.2.0-1.armhf.deb 
hl2260dlpr-3.2.0-1.armhf.deb 
sudo systemctl restart cups
```
- Visit https://<IP>:631/admin, delete the automatically added printer, and manually add it again.

## Download Files

Download the relevant files from the [Brother driver download page](https://support.brother.com/g/b/downloadlist.aspx?c=cn&lang=zh&prod=hl2260d_cn&os=128).

### File List

| File Name | Note |
|-----------|------|
| **Original Files** | |
| `hl2260dcupswrapper-3.2.0-1.i386.deb` | CUPS driver |
| `hl2260dlpr-3.2.0-1.i386.deb` | LPR driver |
| **Compatible Files** | |
| `brgenprintml2pdrv-4.0.0-1.armhf.deb` | Brother ARM generic driver file |
| `brhl2270dwcups_src-2.0.4-2.tar.gz` | brcupsconfig4 source code |

## Build Image

### Extract and Modify LPR Driver

```bash
dpkg -x hl2260dlpr-3.2.0-1.i386.deb hl2260dlpr-3.2.0-1.armhf.ext
dpkg-deb -e hl2260dlpr-3.2.0-1.i386.deb hl2260dlpr-3.2.0-1.armhf.ext/DEBIAN
sed -i 's/Architecture: i386/Architecture: armhf/' hl2260dlpr-3.2.0-1.armhf.ext/DEBIAN/control
echo true > hl2260dlpr-3.2.0-1.armhf.ext/opt/brother/Printers/HL2260D/inf/braddprinter
```

> [!TIP]  
> The path mentioned in the original text `/usr/local/Brother/Printer/HL2260D/inf/braddprinter` should actually be `/opt/brother/Printers/HL2260D/inf/braddprinter`

### Copy ARM Generic Driver

```bash
dpkg -x brgenprintml2pdrv-4.0.0-1.armhf.deb brgenprintml2pdrv-4.0.0-1.armhf.ext
cp brgenprintml2pdrv-4.0.0-1.armhf.ext/opt/brother/Printers/BrGenPrintML2/lpd/armv7l/rawtobr3 \
   hl2260dlpr-3.2.0-1.armhf.ext/opt/brother/Printers/HL2260D/lpd
```

### Repack LPR Driver

```bash
cd hl2260dlpr-3.2.0-1.armhf.ext
find . -type f ! -regex '.*.hg.*' ! -regex '.*?debian-binary.*' ! -regex '.*?DEBIAN.*' -printf '%P ' | xargs md5sum > DEBIAN/md5sums
cd ..
chmod 755 hl2260dlpr-3.2.0-1.armhf.ext/DEBIAN/p* \
          hl2260dlpr-3.2.0-1.armhf.ext/opt/brother/Printers/HL2260D/inf/* \
          hl2260dlpr-3.2.0-1.armhf.ext/opt/brother/Printers/HL2260D/lpd/*
dpkg-deb -b hl2260dlpr-3.2.0-1.armhf.ext hl2260dlpr-3.2.0-1.armhf.deb
```

### Compile brcupsconfig4

```bash
sudo apt install gcc-9-arm-linux-gnueabihf libc6-dev-armhf-cross -y
tar zxvf brhl2270dwcups_src-2.0.4-2.tar.gz
cd brhl2270dwcups_src-2.0.4-2
arm-linux-gnueabihf-gcc-9 brcupsconfig3/brcupsconfig.c -o brcupsconfig4
```

### Build Cupswrapper

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

## Installation

```bash
sudo dpkg --add-architecture armhf
sudo apt update
sudo apt install libc6:armhf
sudo apt install psutils cups
sudo dpkg -i chl2260dcupswrapper-3.2.0-1.armhf.deb hl2260dlpr-3.2.0-1.armhf.deb
```

> [!WARNING]  
> Known issues:  
> - The automatically added printer does not work. Please visit https://<ip>:631/admin, delete the original printer, and add it again.

> [!TIP]  
> If you are an Ubuntu user, it is recommended to uninstall AppArmor or allow the configuration file. If you don't want to bother, just uninstall it directly, since I don't use snap anyway.  
> Finally, delete all temporary files, open [https://yourIP:631](https://yourIP:631) in your browser, and follow the prompts to add the printer. Good luck! This article took 2 weeks to complete.

[1]: [@alexivkin Brother printer drivers for Raspberry Pi and other ARM devices](https://github.com/alexivkin/brother-in-arms)