---
title: linux 配置普通 wifi 和企业级加密 wifi
date: 2019-12-21 18:51:09
tags: 
  - wifi
  - NetworkManager
  - 树莓派
categories:
  - linux
---

依然是在研究树莓派，带着到处插网线实在有点蠢，在家里和公司都把 wifi 配置好，开机自动连接网络就很省心了，再配合上一篇文章讲的内网穿透，更是 ssh ip 都不用费心找。

家里的 wifi 很好配置，因为就是普通的 WPA 个人级 wifi，而公司的 wifi 是 WPA2 企业级加密的 WIFI，费了一番功夫才找到正确的配置方法，记录在下面。

<!--more-->

虽然 ubuntu-18.04 使用 netplan 来管理网络，但是我们仍然可以用 NetworkManager 来管理网络，其在命令行下对应的命令是 nmcli。

## 启用 NetworkManager

NetworkManager 通过`NetworkManager.service`[systemd](https://wiki.archlinux.org/index.php/Systemd) 单元来 [控制](https://wiki.archlinux.org/index.php/Systemd_(简体中文)#使用单元)。 NetworkManager 守护进程启动后，会自动连接到任何可用的已经配置的**系统连接**。**用户连接**或未配置的连接需要通过`nmcli`或 applet 进行配置和连接。

NetworkManager 的全局配置文件位于`/etc/NetworkManager/NetworkManager.conf`。额外的配置文件可以放进 `/etc/NetworkManager/conf.d/` 文件夹。通常全局的默认配置不需要改动。

**开机自动启动 NetworkManager：**

```bash
systemctl enable NetworkManager
```

**立即启动 NetworkManager：**

```bash
systemctl start NetworkManager
```

## 连接普通 WIFI

**查看网络设备列表**

```bash
sudo nmcli d
```

注意，如果列出的设备状态是 unmanaged 的，说明网络设备不受 NetworkManager 管理，你需要清空 /etc/network/interfaces下的网络设置，然后重启。

**开启 WiFi**

```bash
sudo nmcli r wifi on
```

**扫描附近的 WiFi 热点**

```bash
sudo nmcli d wifi
```

**连接到指定的 WiFi 热点**

```bash
sudo nmcli d wifi connect "SSID" password "PASSWORD"
```

*请将 SSID 和 PASSWORD 替换成实际的 WiFi名称和密码*

连接成功后，下次开机，WiFi 也会自动连接。

## 添加 802.1x 认证 wifi

WPA2 企业级加密的 WIFI 是通过 802.1x 认证的，通过 nmcli 需要特殊的配置。

1. **扫描附近的 WiFi 热点，找到那个你想连接的**

   ```bash
   sudo nmcli d wifi
   ```

2. **添加一个新连接**

   其实是添加了一个配置文件到 /etc/NetworkManager/system-connections，NetworkManager 从这个目录下读取网络配置，每个 wifi 配置都会有独立的配置文件。

   ```bash
   sudo nmcli con add type wifi ifname <YOUR_WLAN> con-name <CUSTOM_CONNECTION_NAME> ssid <YOUR_SSID>
   ```

   > YOUR_WLAN：网卡名（如：wlan0）
   >
   > CUSTOM_CONNECTION_NAME：配置文件名，后面要编辑的时候以此为 id 指定要编辑的配置文件（建议与 wifi 的 SSID 同名就好）
   >
   > YOUR_SSID：wifi 的 SSID

   或者直接执行 `sudo nmcli con edit` 后根据提示一步步走，最后保存也是一样的。

3. **编辑这个刚创建出来的配置文件**

   ```bash
   sudo nmcli con edit id <CUSTOM_CONNECTION_NAME>
   
   -> set ipv4.method auto
   -> set 802-1x.eap peap
   -> set 802-1x.phase2-auth mschapv2
   -> set 802-1x.identity <YOUR_USERNAME>
   -> set 802-1x.password <YOUR_PASSWORD>
   -> set wifi-sec.key-mgmt wpa-eap
   -> set connection.autoconnect true
   -> save
   -> activate
   ```

   执行 `activate` 之后，应该就能连接上指定的 WPA2 (802.1X) 网络了

4. **查看已保存的连接**

   ```bash
   sudo nmcli c
   ```

   