# Arch Linux

折腾archlinux的笔记。

## 安装系统

### 检验安装模式

通过U盘进入安装环境后，在虚拟终端中输入

```bash
$ ls /sys/firmware/efi/efivars
```

如果没有报错，则是UEFI安装模式。通过`lsblk`查看磁盘状况，同时坚持安装介质是否完整，存在缺损时无法访问磁盘。

### vconsole

虚拟终端的一些设置，不影响系统的安装进程，可以跳过。临时设置字体方式，以`LatGrkCyr`字体为例。

```bash
$ setfont /usr/share/kbd/consolefonts/LatGrkCyr-12x22.psfu.gz
```

想要永久设置虚拟终端的字体，需要在文件`/etc/vconsole.conf`中添加设置

```shell
FONT=LatGrkCyr-12x22
```

修改虚拟终端的键盘布局，以交换`esc`与`caps lock`按键为例。默认情况下，使用的键盘布局为`us`布局，布局文件存放于`/usr/share/kbd/keymaps/i386/qwerty/`。

- 复制一份`us.map.gz`文件，将其修改成其他名字，例如`personal.map.gz`；
- 使用`gunzip personal.map.gz`解压;
- 编辑`personal`文件，交换`keycode 1`与`keycode 58`的值（对应`Escape`与`CapsLock`）;
- 保存后，用`gzip personal`压缩该文件。
- 在文件`/etc/vconsole.conf`中添加：`KEYMAP=personal`;

### 连接网络

启动U盘介质中，自带连接wifi的工具，`iwlist`与`wpa_supplicant`，连接过程如下：

- `ip link`查看电脑的网络设备，是否存在无线网络设备。设网络设备为`wlan0`，使用`ip link set wlan0 up`可以启动该设备;
- `iwlist wlan0 scan | grep ESSID`将无线网扫描结果通过管道(`|`)匹配(`grep ESSID`)后返回网络名称;
- `wpa_passphrase 网络名 密码 > 文件名`生成对应无限网络的配置文件，文件结尾可以用`.conf`;
- `wpa_supplicant -c 文件名 -i wlan0 &`在后台通过生成的配置文件连接无限网;
- 有可能此时依旧无法联网，是未能分配ip的原因，使用`dhcpcd &`后台运行动态ip分配的请求;

### 更新时间

```bash
$ timedatectl set-ntp true
```

### 磁盘分区

使用UEFI模式安装的系统需要划分三个磁盘区域，一个用来放启动引导，最少300M；一个用来做交换空间，最少512M；最后一个用来做主空间。做双系统最好新建一个EFI分区，即第一个。这样删除第二个系统的时候就无需担心对主系统影响，直接在主系统中格式化分配出来的磁盘空间即可。

划分磁盘的工具有很多，官网教程与B站UP主TheCW使用的是`fdisk`。这里使用一个终端图形化版的工具`cfdisk`，该工具默认打开第一个磁盘，如果有多个磁盘时，需要手动指定。

例如，当前HP电脑有`/dev/sda`与`/dev/sdb`两个磁盘，双系统是装在第二个磁盘中的。需要使用`cfdisk /dev/sdb`才能够看到从Windows中分出的50G自由空间。

`cfdisk /dev/sdb`流程：

- 选中空闲的磁盘条目，通过方向键选择[New];
- 输入划分多少空间，默认是所有可用空间，需要手动修改。
- 回车后，选择[Write]并输入yes，**必须输入yes再回车，直接用回车不会将更改内容真的写入磁盘**;

### 格式化磁盘空间

设划分出来的三个磁盘空间为`/dev/sdb7`, `/dev/sdb8`, `/dev/sdb9`。分别对应EFI，交换空间，主空间。

- 格式化引导区(EFI)：`mkfs.fat -F 32 /dev/sdb7`;
- 格式化主分区：`mkfs.ext4 /dev/sdb9`;
- 格式化交换空间：`mkswap /dev/sdb8`;

### 挂载

启动交换空间

```bash
$ swapon /dev/sdb8
```

挂载主空间

```bash
$ mount /dev/sdb9 /mnt
```

挂载启动分区

```bash
$ mount --mkdir /dev/sdb7 /mnt/boot
```

> 使用`lsblk`来查看挂载情况，在对应的分区后面应该会有三个相应的标志。

### pacman

在下载安装内容之前，先添加国内pacman的源。使用下面的命令查看并安装所有可用镜像。

```bash
# 先同步pacman
$ pacman -Sy

# 查看可用镜像
$ pacman -Ss mirrorlist
# 输出形式应该是core/pacman-mirrorlist xxxxxx
# 安装pacman-mirrorlist即可
$ pacman -S pacman-mirrorlist
```

由于系统会在带一个`mirrorlist`文件，所以新安装的文件会有一个后缀，注意程序输出显示。通常为`mirrorlist.pacnew`。

将文件`mirrorlist.pacnew`中关于中国的镜像地址，复制到同路经下的`mirrorlist`文件顶部。路经为：`/etc/pacman.d/mirrorlist`

**添加32位软件源：**

编辑`/etc/pacman.conf`

```shell
# 找到下面两条注释内容，将注释取消
#[multilib]
#Include = /etc/pacman.d/mirrorlist
```

**添加国内软件源：**

编辑`/etc/pacman.conf`

```
# 在文件末尾添加下面两条

[archlinuxcn]
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch
```

> 添加国内源后，要先安装`archlinuxcn-keyring`导入GPG key才能安装国内源的软件，不然会显示证书不信任还是不匹配问题。

> ~~修改镜像与源后需要先使用`pacman -Syy`同步内容~~

> **[!/2022/5/24]** `Syy`是强制更新软件源，根据网上说法，通常不建议强制更新。正常更新系统为：`sudo pacman -Syu`。暂时的策略为，非必要情况下不连续使用两个相同的字母，如yy，cc等。

**清理缓存**

```shell
# remove all old package cache
$ sudo pacman -Sc
```

### 下载并安装内核

下载linux基本框架与内核并安装到主空间中。

```bash
pacstrap /mnt base linux linux-firmware base-devel
```

### 初始化系统

生成fstab文件，根据网上说法，该文件能够让系统启动时，自动挂载上面的几个分区。

```bash
$ genfstab -U /mnt >> /mnt/etc/fstab
```

> 使用`arch-chroot /mnt`可以切换到安装的新系统中。

**设置时区与时间**

```bash
$ ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
$ hwclock --systohc
```

**语言环境设置**

编辑文件`/etc/locale.gen`，将其中`en_US.UTF-8xxx`以及`zh_CNxxx`的注释取消，需要同时使用中文与英文。

```bash
# 运行命令完成本地化
$ locale-gen
```

创建文件件`/etc/locale.conf`，输入`LANG=en_US.UTF-8`。使用英文作为主要显示的语言，使用中文会出现显示异常问题。

**网络配置**

编辑文件`/etc/hostname`，在里面输入自己想设置的主机名。

**本地环路配置**

有些软件会读取`/etc/hosts`文件，这个文件内容是空的需要手动添加如下内容。

```shell
127.0.0.1        localhost
::1              localhost
127.0.1.1        myhostname
```

**配置root密码**

```bash
passwd
```

### 安装引导与微编码

```bash
$ pacman -S grub efibootmgr intel-ucode os-prober
```

**生成grub的配置文件**

```bash
$ grub-mkconfig -o /boot/grub/grub.cfg
```

> amd的cpu安装amd-ucode，现在os-prober默认是不主动发现其他系统的，需要编辑文件`/etc/default/grub`，将该语句的注释取消`GRUB_DISABLE_OS_PROBER=false`。然后重新运行grub的配置文件。注意，目标系统的引导区也必须挂载到`/mnt/boot`文件上，不然是没用的。

**安装grub条目**

```bash
# 查看本机架构
$ uname -m
# 本机为 x86_64
# 安装grub条目
$ grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=GRUB
```

> `--bootloader-id`可以不写，这个是在系统的BIOS界面选择那个启动引导时显示的名称，默认是以系统命名——Arch。

### 安装必要软件

主要是编辑器，网络连接工具，不凡新系统不自带网络连接工具的

```bash
$ pacman -S neovim zsh wpa_supplicant dhcpcd networkmanager net-tools
```

至此Arch linux 系统安装完毕。重启进入新系统。

## 基础环境

使用`networkmanager`连接网络，比`wpa_supplicant`方便。

```bash
# 扫描wifi
$ nmcli device wifi list
$ nmcli device wifi connect "SSID-Name" password "your password"
# 两个引号内填wifi名与密码
# 设置开机自启
$ systemctl enable NetworkManager
```

**ssh**

```bash
# install
$ pacman -S openssh

# auto start
$ systemctl enable sshd
```

**新用户**

```bash
# add new user: william
$ useradd -G root -m william
# set passwd for sudo command
$ passwd william
```

将新用户加入到sudo组中，编辑文件`/etc/sudoers`，找到`root ALL=(ALL) ALL`这一行，在下面添加`william ALL=(ALL) ALL`

### 驱动安装

**显卡**

```bash
# intel
$ sudo pacman -S mesa lib32-mesa vulkan-intel lib32-vulkan-intel
# nvidia
$ sudo pacman -S nvidia nvidia-settings lib32-nvidia-utils
# manage intel and nvidia gpu
$ yay -S optimus-manager optimus-manager-qt
```

**声卡**

```bash
$ sudo pacman -S alsa-utils alsa-tools pulseaudio-alsa pulseaudio
```

> `alsamixer`命令调出终端图形条件音量。

**电源管理**

```bash
$ sudo pacman -S mate-power-manager
```

> 需要进入图形化桌面才能运行`mate-power-manager`命令。

### 窗口环境

使用窗口管理软件来进行图形化显示，通过Xorg来启动。

```bash
# 安装服务组件
$ sudo pacman -S xorg xorg-server xorg-xinit
```

将文件`/etc/X11/xinit/xinitrc`复制到`~/.xinitrc`，编辑`~/.xinitrc`文件，将最后 **twm**开始一直到文件结尾的几行注释或者删掉。

#### 模拟终端

##### st

```bash
$ git clone https://git.suckless.org/st
```

下载后进入`st`文件夹，修改`confg.mk`文件

```shell
# 将这下面两个变量值修改如下
X11INC = /usr/include/X11
X11LIB = /usr/include/X11
```

通过`sudo make clean install`编译生成软件。将默认配置文件`config.def.h`删除，需先编译一遍。

##### alacritty

#### 快速启动软件

##### dmenu

```bash
$ git clone https://git.suckless.org/dmenu
```

操作同 **st**。

##### rofi

#### 窗口管理软件

##### i3wm

```bash
$ pacman -S i3-gaps polybar
```

在`~/.xinitrc`文件末尾添加`exec i3`启动。初次进入软件会根据指导生成配置文件`~/.config/i3/config`

基础按键

- `mod+Enter`：打开模拟终端st;
- `mod+d`：打开dmenu;
- `mod+R`：重新载入i3wm配置;
- `mod+e`：退出i3wm;

> 初次启动时，会在底部状态栏显示一个错误信息，那是因为没有装i3-status。这用polybar代替。去配置文件中将i3-status的配置注释掉就没有默认状态栏了。

polybar默认配置文件路径`/etc/polybar/config.ini`，复制该文件到`~/.config/polybar/`下，找到`[bar/example]`这一行，后面的example就是启用polybar的样式名，为区分，将其改成其他名字。如bar1，随后在终端中输入

```bash
$ polybar bar1 &
```

就能在后台运行bar1配置文件样式了。`killall polybar`可以杀死所有polybar进程。

> polybar 实现透明背景只需要在背景颜色前加入aa，例如，背景色为`#1e1e3e`时，修改成`#aa1e1e3e`就可以实现透明背景。

##### dwm

```bash
$ git clone https://git.suckless.org/dwm
```

另一种窗口管理软件，安装方式同 **st**，启动配置同 **i3wm**。

### 输入法

**字体**

```bash
# install chinese FONT
$ sudo pacman -S noto-fonts-cjk adobe-source-han-sans-cn-fonts adobe-source-han-serif-cn-fonts

# install code font
$ sudo pacman -S nerd-fonts-source-code-pro adobe-source-code-pro-fonts
```

通过`fc-list`可以查看所有安装的字体。

**fcitx5**

```bash
$ sudo pacman -S fcitx5-chinese-addons fcitx5-git fcitx5-gtk fcitx5-qt fcitx5-pinyin-zhwiki kcm-fcitx5
```

编辑文件`/etc/profile`，在最后添加如下内容

```shell
export INPUT_METHOD="fcitx5"
export XMODIFIERS="@im=fcitx5"
export GTK_IM_MODULE="fcitx5"
export QT_IM_MODULE="fcitx5"
```

同dmenu启动fcitx5-config-qt，在图形见面中添加输入法`pinyin`。可以在这个界面找到输入法字体大小的设置。

> 大部分需要一直运行的软件都可以在`~/.xinitrc`中启动，注：需要在执行窗口管理软件前启动。

### 解压

```bash
# test.tar.gz
$ tar -zxvf test.tar.gz
```

### Clash

下载了`Downloads/sub-web`文件，一个前端页面，将订阅生成配置，通过`yarn serve`开启。`Downloads/subconverter`对应前面前端的后端，运行里面的命令即可。安装了`clash`包，配置文件在`~/.config/clash/config.yaml`，通过前两个将订阅地址生成配置文件。同样下载了一个图形界面的clash，在`Downloads/'Clash for Windows-xxxx'`

但是依旧无法连接google.

### V2ray & V2rayA

另一个代理软件，通过pacman安装，后者是一个网页版前端，初次登陆需要设置账户，代理可用。

通常终端设置代理就是在配置文件中添加环境变量。以zsh为例。

```shell
# if the http port is 12345
export http_proxy="http://127.0.0.1:12345"
export https_proxy="http://127.0.0.1:12345"

# if the socket5 port is 12345
export http_proxy="socks5://127.0.0.1:12345"
export https_proxy="socks5://127.0.0.1:12345"
```

> 终端不会代理icmp，需要使用curl命令查看连通性。在v2rayA的git项目上的常见问题版块有描述。

> 使用该代理后，应该影响本地DNS，正常情况下访问网络没问题。但会让Ghelper无法使用，需要先启动一次该代理。随后关闭。Ghelper就可以使用。

### Linux

#### 进程

通过`ps aux`显示当前终端下所有进程信息。

#### 键盘映射

通过`xmodmap`来修改映射。编辑文件`~/.Xmodmap`，下面例子为交换`~`与`esc`，`capsLock`变成`ctrl_l`，`shift+Caps`映射为`capsLock`

```shell
clear lock
clear control
add control = Caps_Lock Control_L Control_R
keycode 66 = Control_L Caps_Lock NoSymbol NoSymbol
keysym Escape = grave asciitilde grave asciitilde
keysym grave asciitilde grave asciitilde = Escape
```

> 这样的键盘映射只会作用于生效配置时已经接入的键盘。后续在接入其他键盘时需要再次运行命令`xmodmap ~/.Xmodmap`才能够让新键盘也使用该映射。

## issus

pacman的数据锁文件放在`/var/lib/pacman/db.lck`，当中途停止pacman的命令时，下次使用需要删除该文件锁。

### U disk

可以使用安装过程中挂载主分区那样的方式直接将U盘挂载到某个文件。这里使用`udisks2`包进行挂载。使用手动挂载的方式。自动挂载需要编辑一个脚本文件。

通过`lsblk`或者`fdisk -l`查看U盘在系统中的设备名，本机应该是`/dev/sdc`（通过lsblk查看结果），若使用fdisk命令，应看到sdc1。

- `udisksctl mount -b /dev/sdc`：挂载U盘，默认挂载到文件`/run/media/william/xxx`，在xxx文件夹中有一个`System Volume Information`文件夹，应该是该工具包挂载生成的。实际上不存在U盘内部。xxx文件夹就是U盘内部了。

- `udisksctl unmount -b /dev/sdc`：取消挂载，也就是弹出U盘。

### arandr

设置多屏分辨率的一个GUI软件。

### alacritty-themes

一个修改alacritty终端主题的工具，通过标题名称调用。用npm安装。

### rofi

与dmenu一样的程序启动器，主题仓库`https://github.com/adi1090x/rofi`

### flameshot

一款开源截图软件，在rofi中使用`flameshot gui`开始截图。

### arch.icekylin

比较新的archlinux入门网站，
[地址](https://arch.icekylin.online/)

### zathura

一款linux下开源的pdf阅读器，能够支持多种格式，出了普通的pdf，还有EPUB等格式的支持，具体可以查看archwiki。

```shell
# for pdf read
$ sudo pacman -S zathura zathura-pdf-poppler
```

> the config file is located in ~/.config/zathura/zathurarc

### btop

一款终端资源检测工具，能够显示cpu，内存，后台运行进程等信息。通过`<esc>`呼出菜单可修改设置，查看帮助。

```shell
$ sudo pacman -S btop
```

### 外接显示器

通常情况下，外接显示器的接口是接在GPU上的，按照自己安装Arch linux的笔记流程，默认使用的是集成显卡来显示桌面。这种情况下，连接显示器是不会有显示效果的，需要将主要的显卡切换到nvidia显卡上。（当前电脑是一个intel核显，一个nvidia独显）。

之前是安装过`optimus-manager`软件，该软件能够实现在显卡之间的切换。但是，由于使用的`Xorg`来启动的窗口管理界面，不是桌面环境。在使用前后需要介入额外的命令。

```shell
# after startx command
$ prime-offload

# show info of current GPU
$ optimus-manager --status

# switch to nvidia GPU
$ optimus-manager --switch nvidia

# switch to integrated GPU
$ optimus-manager --switch integrated

# switch to hybrid model
# but need some config
$ optimus-manager --switch hybrid
```

**note**: 在结束xorg服务后，比如退出i3wm后，需要执行命令: `sudo prime-switch`，才能够成功切换显卡模式。

> 每次切换显卡都会自动退出窗口环境。

> glxinfo | egrep "OpenGL vendor|OpenGL renderer" 同样能偶查看当前使用的显卡名称。

### file manager

某些情况下需要用到图形化文件管理器。选择安装的是GNOME桌面环境的默认文件管理器`nautilus`，安装后通过命令`Files`打开。

在i3的环境中，每个软件在需要使用到文件管理器的情况下时，都是依据自身的设置绘制文件管理窗口。这就导致某些软件的窗口非常老旧。如`flameshot`软件。若需要像桌面环境那样每个软件都使用统一的管理。通过网上搜索。提到两个环境变量：`DE`, `XDG_CURRENT_DESKTOP`。暂时没去验证两个都要还是只要其中一个即可。

```shell
# in shell rc file.
# .zshrc for example.
# set to gnome, kde etc. the nautilus is gnome's default file manager
export DE='gnome'
export XDG_CURRENT_DESKTOP=gnome

# need add to PATH
export PATH = $PATH:$DE:$XDG_CURRENT_DESKTOP
```

### bleachbit

一款linux下的开源清理工具。

### picom

关于picom会给所有软件都添加透明效果问题，在arch都论坛中有提及，是一个随机bug。
有人提及一个解决方案如下。

**依据picom的设置，应该是默认为某种规则下的前端窗口软件添加透明背景。在使用中，
火狐浏览器右键时会又透明，其他情况下没有。谷歌浏览器完全没有透明背景等。
此时通过配置文件中为具体的软件配置透明参数。注意：一个软件需要配置两条**。

```shell
picom -bcCGfF -o 0.38 -O 200 -I 200 -t 0 -l 0 -r 3 -D2 -m 0.88 --config /dev/null &
```

上述方案在本地上是有参数错误的，近期发现picom在本地上配置文件曾经起过作用。
后面又无法生效的原因：应该是使用ranger的过程中无意修改来配置文件的名称，导致程序无法检索到配置文件。

现在picom配置生效，且配置来透明背景与毛玻璃效果。日期：2022/7/27

> 启用picom渲染后，无意解决来i3在程序启用浮动窗口模式时，通过鼠标拖动，修改窗口大小时。
> 会出现移动位置存在绿色花屏的情况。

### 中文设置

在安装过程中已经设置过中文对应的编码并生成来本地环境。只需要在`~/.xinitrc`文件中添加如下语句即可让支持中文的软件识别成中文环境。需要重启电脑才生效。

```shell
export LANG=zh_CN.UTF-8
export LANGUAGE=zh_CN:en_US
```

若在安装过程中没能够生成中文编码，需要先进入文件`/etc/locale.gen`，将中文编码前的注释去除。格式为`#zh_CN.xxx`，随后执行命令`locale-gen`。来生成本地编码。

[arch 官网教程：https://wiki.archlinux.org/title/Localization/Simplified_Chinese](https://wiki.archlinux.org/title/Localization/Simplified_Chinese) 

> 设置成中文环境后，能够解决在英文环境下，某些中文字体显示的是日文/繁体版本。例如`即`字。在英文环境就有显示错误问题。
> 
> 同时，设置成中文环境后，某型字体符号相对与英文环境会变大。最直接的就是nvim中的bufferline插件，当修改文件时会有一个绿色球体符号。该符号在中文环境中表现较大，英文环境中就比较小。

### 蓝牙音频

 所需软件包

```shell
$ sudo pacman -S bluez bluez-utils pulseaudio-bluetooth
```

启动蓝牙

```shell
$ sudo systemctl start bluetooth.service
```

初次连接蓝牙

```shell
# into bluetooth manager page
$ bluetoothctl
[bluetoothctl] power on # open bluetooth
[bluetoothctl] agnet on # open agent
[bluetoothctl] scan on # begin scan, scan off stop scan
[bluetoothctl] trust xx:xx:xx(device mac) # trust device
[bluetoothctl] pair xx:xx:xx(device mac) # pair device
[bluetoothctl] connect xx:xx:xx(device mac) # connect the device

[bluetoothctl] disconnect xx:xx:xx(device mac) # disconnect the device
```

后续连接蓝牙

```shell
# show trusted devices
$ bluetoothctl devices
# connect
$ bluetoothctl connect xx:xx:xx(device mac)
```

>  连接成功后可能需要切换音频，可通过图形化软件`pavucontrol`进行切换。
