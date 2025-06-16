# Adding Network Functionality to Brother-HL2260D Printer on ARM64 Devices

## Preface
On AMD64/x32 devices, this is a straightforward task requiring only official tool installation. However, Brother does not provide official drivers for ARM64 architecture. After failing to build with Docker, I successfully deployed network functionality on an Orange Pi Zero3 by referencing article [1].

If you don't want to compile the `.deb` packages yourself, you can directly download them from the Releases section.

## Usage

Install Pre-Built Deb Packages
```bash
wget https://github.com/KirsminX/brother-hl2260d-on-arm/releases/download/0.0.1/chl2260dcupswrapper-3.2.0-1.armhf.deb
wget https://github.com/KirsminX/brother-hl2260d-on-arm/releases/download/0.0.1/hl2260dlpr-3.2.0-1.armhf.deb
```
Install Dependencies

```bash
sudo apt update
sudo apt install psutils cups libc6 libstdc++6 libusb-1.0-0 gsfonts ghostscript -y
```

Install the Driver

```bash
sudo dpkg -i chl2260dcupswrapper-3.2.0-1.armhf.deb hl2260dlpr-3.2.0-1.armhf.deb
sudo systemctl restart cups
```

Add the Printer

- Open your browser and go to: `https://<your-ip>:631/admin`
- Remove any automatically added printers
- Manually add your Brother HL-2260D printer using the CUPS interface

## Download Files

Download the following files from [Brother Driver Download Page](https://support.brother.com/g/b/downloadlist.aspx?c=cn&lang=zh&prod=hl2260d_cn&os=128).

### File List  

| Filename | Comment |
|--------|------|
| **Original Files** | |
| `hl2260dcupswrapper-3.2.0-1.i386.deb` | CUPS Driver |
| `hl2260dlpr-3.2.0-1.i386.deb` | LPR Driver |
| **Compatible Files** | |
| `brgenprintml2pdrv-4.0.0-1.armhf.deb` | Brother ARM Universal Driver |
| `brhl2270dwcups_src-2.0.4-2.tar.gz` | brcupsconfig4 Source Code |

## Building Image

### Extract and Modify LPR Driver

```bash
dpkg -x hl2260dlpr-3.2.0-1.i386.deb hl2260dlpr-3.2.0-1.armhf.ext
dpkg-deb -e hl2260dlpr-3.2.0-1.i386.deb hl2260dlpr-3.2.0-1.armhf.ext/DEBIAN
sed -i 's/Architecture: i386/Architecture: armhf/' hl2260dlpr-3.2.0-1.armhf.ext/DEBIAN/control
echo true > hl2260dlpr-3.2.0-1.armhf.ext/opt/brother/Printers/HL2260D/inf/braddprinter
```

> [!TIP]  
> The original article mentions path `/usr/local/Brother/Printer/HL2260D/inf/braddprinter`, which should actually be `/opt/brother/Printers/HL2260D/inf/braddprinter`

### Copy ARM Universal Driver

```bash
dpkg -x brgenprintml2pdrv-4.0.0-1.armhf.deb brgenprintml2pdrv-4.0.0-1.armhf.ext
cp brgenprintml2pdrv-4.0.0-1.armhf.ext/opt/brother/Printers/BrGenPrintML2/lpd/armv7l/rawtobr3 \
   hl2260dlpr-3.2.0-1.armhf.ext/opt/brother/Printers/HL2260D/lpd
```

### Repackage LPR Driver

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

> [!TIP]  
> Ubuntu users are advised to remove AppArmor or configure policies accordingly. Just remove it if you hate hassle (I don't use snap anyway)  
> Finally, delete all temporary files and open [https://your_ip:631](https://your_ip:631) in browser to add printer following instructions. Good luck! This guide took 2 weeks

[1]: [@alexivkin Brother printer drivers for Raspberry Pi and other ARM devices](https://github.com/alexivkin/brother-in-arms)  
