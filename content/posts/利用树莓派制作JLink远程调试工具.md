---
title: "利用树莓派制作JLink远程调试工具"
date: 2025-09-01T10:12:39+08:00
description: "用树莓派搭建 JLink 远程调试服务"
categories: ["实用工具"]
tags: ["树莓派", "JLink", "远程调试", "Keil"]
---

# 利用树莓派制作JLink远程调试工具

## 1.树莓派烧录操作系统

首先去树莓派官网 https://www.raspberrypi.com/software/ 安装树莓派烧录操作系统工具。

![](/images/利用树莓派制作JLink远程调试工具/树莓派官网.png)

电脑插入sd卡，打开安装好的软件，选择使用的树莓派型号、安装的操作系统和sd卡对应的磁盘。这里选择安装Ubuntu操作系统。

![](/images/利用树莓派制作JLink远程调试工具/安装操作系统0.png)

![](/images/利用树莓派制作JLink远程调试工具/安装操作系统1.png)

![](/images/利用树莓派制作JLink远程调试工具/安装操作系统2.png)

![](/images/利用树莓派制作JLink远程调试工具/安装操作系统3.png)

## 2.配置树莓派环境

写入校准通过后，将sd卡插入树莓派，连接电源适配器上电启动，使用HDMI接口连接显示器，使用USB接口连接键盘。

### 2.1登录用户

等待树莓派跑完bootloader启动流程后，登录默认用户。对于树莓派上的Ubuntu Server官方镜像，默认的用户名是ubuntu，密码是ubuntu。

### 2.2连接wifi

在Ubuntu中Netplan是默认的网络配置管理工具。

首先需要确认无线网卡是否被系统识别，然后需要修改Netplan配置文件，将wifi的账号和密码添加进去，最后应用新的配置文件。

输入以下指令查看树莓派的无线网卡是否被系统识别。

```shell
ip addr
```

使用 `nano` 编辑器打开配置文件

```shell
sudo nano /etc/netplan/50-cloud-init.yaml
```

修改文件内容，在文件中添加 `wifis` 部分。请特别注意缩进（使用空格，不能使用 Tab 键）。

```shell
network:
    version: 2
    ethernets:
        eth0:
            dhcp4: true
            optional: true
    wifis:
        wlan0: # 请确保这是你的正确无线接口名
            dhcp4: true
            optional: true
            access-points:
                "你的Wi-Fi名称": # 请用你的真实Wi-Fi名替换，保留引号
                    password: "你的Wi-Fi密码" # 请用你的真实密码替换
```

在 `nano` 编辑器中修改完成后，按 `Ctrl+X`，然后按 `Y` 确认保存，最后按 `Enter` 确认文件名。

然后输入指令应用新的配置文件。

```shell
sudo netplan apply
```

最后检查wifi是否连接成功。

```shell
ip addr show wlan0
# 你应该能看到一个 'inet' 地址，类似 192.168.1.123
ping -c 4 www.baidu.com
# 测试网络是否通畅
```

## 3.通过ssh远程控制树莓派

### 3.1在树莓派上安装并启用SSH服务

Ubuntu系统默认可能没有安装SSH服务器端。请在你的树莓派命令行终端中依次执行以下命令：

1.**更新软件包列表**（确保获取最新的软件信息）：

```shell
sudo apt update
```

2.**安装OpenSSH服务器**：

```shell
sudo apt install openssh-server
```

3.**确保SSH服务已启动并设置开机自启**

安装后，SSH服务通常会自动启动。你可以通过以下命令检查其状态：

```shell
sudo systemctl status ssh
```

如果看到 `Active: active (running)` 就表示服务正在运行。

**4.如果SSH没有激活**

输入命令启用ssh开机自启

```shell
sudo systemctl enable ssh
```

### 3.2配置树莓派运行允许使用密钥连接

编辑SSH服务器的配置文件

```shell
sudo nano /etc/ssh/sshd_config
```

找到以下变量，前俩配置成yes，第三个注释掉

```
PubkeyAuthentication yes
PasswordAuthentication yes
# AuthenticationMethods publickey,password
```

ctrl+o保存文件，按enter确认，ctrl+x退出nano编辑器

输入命令，确保没有和上面三个配置冲突的地方

```shell
sudo cat /etc/ssh/sshd_config.d/*
```

如果有的话找到注释掉

重启SSH服务器

```shell
sudo systemctl restart ssh
```

### 3.3获取树莓派的IP地址

需要知道你的树莓派在局域网中的IP地址，才能从其他电脑连接它。在树莓派终端中输入以下命令：

```shell
ip addr
```

记下wlan0的IP地址

### 3.4电脑上使用SSH连接

打开putty.exe软件，输入树莓派的IP地址，选择SSH连接方式，点击open，然后输入树莓派上的用户名和密码，就可以连接成功。

## 4.树莓派安装JLink环境

在树莓派终端里首先更新一下

```shell
sudo apt -y update && sudo apt -y upgrade
```

安装J-Link软件

```shell
cd ~/Downloads
wget --post-data 'accept_license_agreement=accepted&non_emb_ctr=confirmed&submit=Download+software'   https://www.segger.com/downloads/jlink/JLink_Linux_arm64.deb
sudo dpkg -i JLink_Linux_arm64.deb
sudo apt -y update && sudo apt -y upgrade
```

## 5.使用Keil连接到仿真器

树莓派端运行JLink的远程服务功能

```shell
JLinkRemoteServerCLExe -Port 19020
```

PC端配置Keil设置，打开Keil-Options for Target-Debug-JLink Settings，然后按照下图设置，其中IP地址改成树莓派的地址。

![](/images/利用树莓派制作JLink远程调试工具/Keil配置.png)

点击Connect连接测试，识别到JLink设备即成功。
