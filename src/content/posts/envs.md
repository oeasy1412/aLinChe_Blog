---
title: 环境配置
published: 2025-05-02
description: 环境配置教程备份
tags: [git]
category: git
draft: false
---

# 系统级配置
## Ubuntu
```sh
## Basic
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo nano /etc/apt/sources.list
# 替换为清华源。注意：请务必将配置中的 jammy 替换为自己系统的实际版本代号，例如20.04是focal，22.04是jammy，24.04是noble
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-updates main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-backports main restricted universe multiverse
deb https://mirrors.tuna.tsinghua.edu.cn/ubuntu/ jammy-security main restricted universe multiverse

sudo apt update
sudo apt install -y fish
chsh -s $(which fish)
echo $SHELL

sudo timedatectl set-timezone Asia/Shanghai
date

sudo apt install -y vim git htop
git config --global user.name "aLinChe"
git config --global user.email "1129332011@qq.com"
git config --global core.editor vim
git config --global color.ui true

sudo apt install python3 python3-pip
python -m pip install --upgrade pip
pip config set global.index-url https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple
pip3 config list


## Docker
# https://docs.docker.com/desktop/setup/install/linux/ubuntu/
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done

# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker ${USER}
docker version

docker pull alinche/echo
docker images -a


## Nix
sh <(curl -L https://nixos.org/nix/install) --daemon
nix-channel --add https://github.com/nix-community/home-manager/archive/master.tar.gz home-manager
nix-channel --update
nix-shell '<home-manager>' -A install
vim ~/.config/home-manager/home.nix
#   nix = {
#     package = pkgs.nix;
#     settings.experimental-features = [ "nix-command" "flakes" ];
#     settings.access-tokens = "github.com=<your-github-access-token>";
#   };

#   programs = {
#     direnv = {
#       enable = true;
#       enableBashIntegration = true;
#       nix-direnv.enable = true;
#     };
    
#     fish.enable = true;
#     # bash.enable = true; # 使用 nix bash 管理
#   };
mv ~/.bashrc ~/.bashrc.backup
mv ~/.profile ~/.profile.backup

home-manager switch


## else
# disk
df -h /tmp
sudo vgdisplay
sudo lvdisplay
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv

lsblk
sudo fdisk /dev/sda
p -> d -> n -> w
sudo partprobe
sudo resize2fs /dev/sda2

# hostname
sudo hostnamectl set-hostname aLinChe
# netplan
sudo vim /etc/netplan/50-cloud-init.yaml
# systemctl
systemctl --user list-unit-files --type=service --state=enabled
```


## Arch
```sh
https://archlinux.org/download/ # archinstall

systemctl stop reflector.service
vim /etc/pacman.d/mirrorlist
Server = http://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch
Server = http://mirrors.hust.edu.cn/archlinux/$repo/os/$arch
Server = https://mirrors.hust.edu.cn/archlinux/$repo/os/$arch

pacman -Sy pacman-mirrorlist
cat /etc/pacman.d/mirrorlist.pacnew | grep China -A 42
cat /etc/pacman.d/mirrorlist.pacnew | grep China -A 42 > /etc/pacman.d/mirrorlist
vim /etc/pacman.d/mirrorlist

vim /etc/pacman.conf


fdisk /dev/sda # mnw
lsblk
mkfs.ext4 /dev/sda1
mount /dev/sda1 /mnt/

pacstrap -i /mnt base base-devel linux linux-firmware
genfstab -U -p /mnt > /mnt/etc/fstab
arch-chroot /mnt


echo ArchKK > /etc/hostname
pacman -S iw wpa_supplicant wireless_tools net-tools
pacman -S networkmanager
systemctl enable NetworkManager
pacman -S openssh
systemctl enable sshd
pacman -S dialog

ln -s -f /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
hwclock --systohc
date

vim /etc/locale.gen  # zh_CN.UTF-8
locale-gen
echo LANG=en_US.UTF-8 > /etc/locale.conf
cat /etc/locale.conf

useradd -m -G wheel -s /bin/bash kkArch
passwd kkArch
pacman -S fish
chsh -s /bin/fish kkArch

pacman -S grub
grub-install /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg

exit
reboot
```


## NixOS
```sh
https://github.com/oeasy1412/nixos_config/
```

## Windows
```sh
## LTSC is all you need.
https://next.itellyou.cn/Original/

## WSL2
“控制面板”->“程序”->“启用或关闭Windows功能”中勾选“适用于 Linux 的 Windows 子系统”和“虚拟机平台”
重启计算机。重启后，以管理员身份打开PowerShell：
wsl --set-default-version 2
通过 Microsoft Store 搜索"Ubuntu"点击"获取"进行安装
# 法2：wsl --list --online (wsl --install -d Ubuntu-22.04)

## WSL迁移到D盘
wsl -l -v
wsl --shutdown
wsl --export Arch D:\WSL2\Arch\arch.tar
wsl --unregister Arch
wsl --import D:\WSL2\Arch D:\WSL2\Arch\Arch.tar --version 2
del D:\WSL2\Arch\Arch.tar
Arch.exe config --default-user [your name]
# 同理 wsl --import Ubuntu-22.04 D:\WSL2\Ubuntu-22.04 D:\WSL2\Ubuntu-22.04\Ubun.tar --version 2

## disk 硬盘盒启动 + Ventoy安装系统
diskpart
list disk              # 确认新硬盘是磁盘1（根据容量931GB判断）
select disk 1          # 选中您的机械硬盘（⚠️绝对不要选错磁盘！）
clean                  # 删除所有分区（新硬盘可安全执行）
convert gpt            # 转换为GPT分区表（兼容大硬盘）
create partition primary # 创建主分区（默认占用全部空间）
format fs=ntfs quick   # 快速格式化为NTFS文件系统
assign letter=E        # 手动分配驱动器号为E（可替换为F,G等）
exit
```



# 软件级配置
## Conda
```sh
## python
https://www.python.org/downloads/ # Windows
## conda
## 安装
curl -O https://mirrors.tuna.tsinghua.edu.cn/anaconda/archive/Anaconda3-2025.06-1-Linux-x86_64.sh
chmod +x Anaconda3-2025.06-1-Linux-x86_64.sh
bash ./Anaconda3-2025.06-1-Linux-x86_64.sh

conda info --envs
## d2l
conda create -n d2l-env python=3.11
conda activate d2l-env
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/pytorch/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/
pip install jupyter torch torchvision
pip install d2l
conda install ipykernel
python -m ipykernel install --user --name d2l-env --display-name "d2l"
# rm
conda remove --name d2l-env --all
conda env list
# ​​导出环境配置​​
conda activate d2l-env
conda env export > d2l-env.yml
# 复用
# conda env create -f d2l-env.yaml --name new_torch_env
conda env create -n NewProject --file d2l-env.yml
conda activate NewProject
conda config --show
conda env list

## ROCm
https://github.com/likelovewant/ROCmLibs-for-gfx1103-AMD780M-APU/ # 鸡哥无界14X
conda create -n ROCm python=3.11
conda activate ROCm
# pip install torch==2.7.1 torchvision==0.22.1 torchaudio==2.7.1 --index-url https://download.pytorch.org/whl/rocm6.3
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu118
pip install jupyter d2l
conda install ipykernel
python -m ipykernel install --user --name ROCm --display-name "ROCm"

## jupyter
jupyter kernelspec list
jupyter nbconvert --to script a.ipynb # html markdown pdf
jupyter notebook list
jupyter notebook --ip=0.0.0.0 --port=8888 --allow-root
```


## CUDA
```sh
## nvidia 驱动
ubuntu-drivers devices
# 安装推荐驱动（或手动指定版本）
sudo ubuntu-drivers autoinstall
sudo reboot
nvidia-smi

https://developer.nvidia.com/cuda-downloads/ # is all you need
## PATH
sudo ln -s /usr/local/cuda-12.2 /usr/local/cuda
export PATH=/usr/local/cuda/bin${PATH:+:${PATH}}
export LD_LIBRARY_PATH=/usr/local/cuda/lib64${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
export PATH="$HOME/anaconda3/bin:$PATH"

## 重装
# 清理旧驱动
sudo apt purge '^nvidia-.*' && sudo apt autoremove
# 安装推荐版本（如575 => cuda12.9）
sudo apt install nvidia-driver-575
# 禁用 Nouveau 驱动
echo "blacklist nouveau" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
sudo update-initramfs -u
sudo reboot

sudo nvidia-smi -pm 1
nvidia-smi
watch -n 1 nvidia-smi

## e.g. GTX1060 特化版本
# 安装 535 驱动
sudo apt install -y nvidia-driver-535
# 重建内核模块
sudo dpkg-reconfigure nvidia-dkms-535
sudo update-initramfs -u
# 添加 CUDA 12.2 官方仓库
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-ubuntu2204.pin
sudo mv cuda-ubuntu2204.pin /etc/apt/preferences.d/cuda-repository-pin-600
# 下载并安装CUDA 12.2的本地仓库包
wget https://developer.download.nvidia.com/compute/cuda/12.2.0/local_installers/cuda-repo-ubuntu2204-12-2-local_12.2.0-535.54.03-1_amd64.deb
sudo dpkg -i cuda-repo-ubuntu2204-12-2-local_12.2.0-535.54.03-1_amd64.deb
# 将密钥环复制到正确位置
sudo cp /var/cuda-repo-ubuntu2204-12-2-local/cuda-*-keyring.gpg /usr/share/keyrings/
sudo apt update
# 安装CUDA 12.2工具包（不包含驱动，因为已单独安装）
sudo apt install -y cuda-toolkit-12-2

sudo ln -s /usr/local/cuda-12.2 /usr/local/cuda

conda install -c nvidia cudnn cudatoolkit=12.2
```


## tailscale
```sh
https://tailscale.com/
## 安装Tailscale
### Linux
curl -fsSL https://tailscale.com/install.sh | sh
### Windows
https://mirrors.scutosc.cn/repository/tailscale/stable/tailscale-setup-latest.exe
### MacOS
https://mirrors.scutosc.cn/repository/tailscale/stable/Tailscale-latest-macos.pkg
### Andriod
release: https://github.com/tailscale/tailscale-android/releases/
you can also accelerate your download here: https://hub.samuka007.com/tailscale/tailscale-android/releases （x

## 如何使用
### Linux
# 将返回的链接COPY到浏览器打开，在浏览器登录验证
sudo tailscale up
# 打开开机自启
sudo systemctl enable tailscaled
### Windows
- Windows点击右下角的 `∧` ，点击 `Log in`
### MacOS
- 副歌v我50（x
### Andriod
- 直接打开登录

# 查看组网情况
tailscale status
# IP  Name  User  OS status

# More 
tailscale set
tailscale configure
tailscale file
scp A B # from A to B
tailscale ip
tailscale netcheck
tailscale configure

## handscale 
sudo apt install iproute2
wget https://github.com/juanfont/headscale/releases/download/v0.26.0/headscale_0.26.0_linux_amd64.deb
sudo dpkg -i headscale_0.26.0_linux_amd64.deb

sudo headscale users create kkubun
# 生成预授权密钥
sudo headscale preauthkeys create --reusable -e 180d -u <用户ID>
sudo headscale preauthkeys ls -u kkubun
# 为整个组织创建密钥
headscale preauthkeys create -u 0 -e 180d -r -o

sudo headscale nodes ls
sudo headscale nodes list-routes
sudo headscale users ls


tailscale up --login-server=http://<headscale-server>:8080 --authkey <key>

## 流量出口
## server 
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p  # 立即生效
# 添加流量伪装规则（替换 youreth0 为实际外网网卡名）
sudo iptables -t nat -A POSTROUTING -o youreth0 -j MASQUERADE
# 保存规则（若使用 netfilter-persistent）
sudo netfilter-persistent save

echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

sudo tailscale up --advertise-route=192.168.199.0/24
sudo tailscale up --advertise-exit-node
sudo tailscale set --advertise-exit-node

![Tailscale 管理后台](https://login.tailscale.com/admin/machines) ​Edit route settings​ -> ​​Use as exit node​​
# ip route show default 
# 停止宣告出口节点​
sudo tailscale up --advertise-exit-node=false

# client 
sudo tailscale up --exit-node=<节点IP或主机名> --exit-node-allow-lan-access
# 清除出口节点配置
sudo tailscale up --exit-node=""
```




## Termux
```sh
https://github.com/termux/termux-app/
https://github.com/termux/termux-api/

vim $PREFIX/etc/apt/sources.list # 自带nano
deb https://mirrors.tuna.tsinghua.edu.cn/termux/apt/termux-main stable main
# termux-change-repo # 使用Termux提供的交互式工具更换镜像源
# pkg install tsinghua.sources
# pkg install ustc.sources

pkg update && pkg upgrade
pkg install vim openssh termux-services
# pkg install htop clang (可选)

whoami
passwd
ifconfig
sshd
# ssh 安卓用户名@IP -p 8022
pkg i termux-services
sv-enable ssh-agent
sv-enable sshd # sv status sshd
vim $PREFIX/etc/termux-login.sh
if tty -s; then
    if pgrep -x "sshd" > /dev/null; then
        echo "sshd is already running."
    else
        sshd
        echo "Started sshd."
    fi
    termux-wake-lock # 阻止手机休眠时CPU和网络被挂起
    echo "Welcome Termux!"
fi

vim $PREFIX/etc/ssh/sshd_config
ClientAliveInterval 60
ClientAliveCountMax 3

# scp 传输文件
scp ./main.cpp myHand:~/cpp
# 在您的电脑上创建或编辑SSH客户端配置文件
Host myHand
  HostName 192.168.1.100 # 请替换为您手机的实际局域网IP
  Port 8022
  User u0_a118 # 请替换为Termux中`whoami`命令返回的用户名
  ServerAliveInterval 60
  ServerAliveCountMax 3
  IdentityFile C:\Users\Name\.ssh\myHand_id_rsa # 使用密钥认证(推荐)

termux-setup-storage # 允许访问手机的存储空间

termux-vibrate

termux-media-player play "storage/shared/Music/a.mp3"

termux-camera-info
termux-camera-photo storage/shared/a.jpg

termux-microphone-record -f storage/shared/a.m4a -l 0

ls storage/shared/Android/data/
am start com.tencent.mobileqq/.activity.SplashActivity

termux-battery-status
termux-notification
```


## 图吧工具箱
```sh
https://tbtool.dawnstd.cn/
```

## MinGW-w64
```sh
https://winlibs.com/
```
