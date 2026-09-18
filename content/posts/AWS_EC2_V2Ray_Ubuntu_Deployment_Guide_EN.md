# 🚀 AWS EC2 Hands-On Deployment Guide: Setting Up a Dedicated V2Ray Node (Ubuntu)

## Introduction

After spending some time working with cloud servers, I decided to document the complete process of deploying a V2Ray node from scratch on AWS EC2. For people who are new to AWS, Linux, and V2Ray, the hardest part is often not any individual command, but understanding how all the steps fit together: Why do you need to configure a security group after creating an EC2 instance? Why can SSH connect successfully while the V2Ray client cannot? Why might a client show port `7890`, while AWS should not have port `7890` exposed? Why can V2Ray show as running while external connections still fail? These issues may seem unrelated, but they are actually connected to cloud-server networking, ports, firewalls, and application listeners.

This guide therefore does more than simply list a few commands. It follows the actual deployment sequence, covering the AWS account, EC2 instance creation, Ubuntu preparation, SSH access, security-group configuration, V2Ray installation, UUID generation, configuration-file editing, service startup, client configuration, and troubleshooting. You can use it as a hands-on guide from scratch, or return to the relevant section when you encounter a problem during deployment.

This guide uses Ubuntu Server LTS and AWS EC2 as the base environment. AWS console interfaces, instance types, free-tier policies, and V2Ray client interfaces may change over time, so do not worry if your screen looks slightly different from the examples here; simply locate the corresponding function. Keep in mind that AWS resources such as instances, storage, public IP addresses, and data transfer may incur charges. Before creating a server, check the free-tier eligibility and pricing that apply to your account. This guide is intended primarily for learning about Linux cloud servers, network-service deployment, and related technologies. Use these techniques in compliance with applicable laws, regulations, and the AWS Terms of Service.

---

## 1. Prerequisites: What Do You Need?

Before starting, you need a working AWS account, a computer with Internet access, and a secure place to store your AWS SSH private key. Windows users can use PowerShell or Windows Terminal, while macOS and Linux users can use the built-in Terminal application. On the server side, we will create an Ubuntu Server cloud instance on AWS EC2. On the client side, choose a client that supports the required protocol for your device and use case.

The entire deployment can be understood as the following chain:

```text
你的电脑
   │
   │ SSH / 客户端连接
   ▼
AWS EC2
   │
   ▼
Ubuntu Server
   │
   ▼
V2Ray
   │
   ▼
Internet
```

AWS EC2 provides the server environment, Ubuntu is the operating system running on the EC2 instance, V2Ray is the network service itself, and the AWS Security Group and Ubuntu firewall control whether external traffic can reach specific ports on the server.

在开始之前，还需要特别理解一个概念：**客户端本地代理端口和服务器端服务端口是两回事。**例如某些客户端可能在你的电脑本地使用 `127.0.0.1:7890` 作为代理入口，这个 `7890` 属于你自己的电脑，并不代表 AWS 服务器需要监听或者开放 `7890`。如果服务器上的 V2Ray 实际监听的是 `10086`，那么服务器需要处理的就是 `10086`，而不是客户端本地的 `7890`。

---

## 2. Creating an AWS EC2 Instance

After signing in to the AWS Management Console, enter `EC2` in the top search bar and open the EC2 service. Find “Launch instance.” When creating the instance, you can first set an instance name such as `My-V2Ray-Server`, `My-VPS`, or `Ubuntu-Server`. This name is mainly for your own identification and does not affect V2Ray.

Next, select the operating-system image, known as the AMI. This guide uses Ubuntu Server as an example, and it is generally preferable to choose a currently supported LTS release offered by the AWS console. LTS means Long Term Support, which is generally well suited to server environments. If you see releases such as Ubuntu 22.04 LTS or Ubuntu 24.04 LTS, choose according to current software compatibility and your needs. Do not assume that you must use exactly the version mentioned in an older tutorial.

For the instance type, a small instance may be sufficient if you are only learning Linux, testing V2Ray, or running very lightweight services. However, AWS instance types and free-tier policies can change. The frequently mentioned `t2.micro` in older tutorials does not mean it will necessarily be free at all times, in all regions, or for every account. Use the eligibility and pricing shown in your own AWS console as the source of truth.

Next, create an SSH Key Pair. Find the Key pair option, create a new key pair such as `MyEC2Key`, choose the appropriate key format, and download it. For Linux, macOS, and Windows OpenSSH environments, `.pem` is commonly used. Store the file securely, for example in a dedicated AWS key directory on your computer. This file is important because you will need it to connect to the server via SSH. Never upload the `.pem` file to GitHub, a public cloud drive, a forum, or any other public location, and never commit it to a code repository.

After configuring the key pair, move on to networking. The Security Group is especially important. You can think of it as a network firewall around the EC2 instance that determines which sources can access which server ports. SSH normally uses TCP port 22, so your computer must be allowed to reach the server over TCP 22. If V2Ray later listens on TCP 10086, external access to TCP 10086 must also be permitted according to your actual use case.

If you are using SSH only to administer the server yourself, avoid leaving SSH open to the entire Internet indefinitely. For example, you can restrict TCP 22 to your current public IP and use `/32` to represent a single IP address. If your public IP changes frequently, adjust the rule as needed, but do not permanently allow all sources simply for convenience.

Do not open large numbers of unnecessary ports. If the server only needs SSH and V2Ray, there is usually no reason to expose ports such as `8080`, `8888`, `7890`, or `12345`. A good server-management principle is: open what you need, rather than opening everything first and deciding later.

Finally, review the instance name, Ubuntu image, instance type, key pair, network settings, and storage configuration. Once everything is correct, click “Launch instance.” Wait for the instance state to become `Running`, then open its details and find `Public IPv4 address`. This is the public IPv4 address you may use later for SSH access and client connections.

---

## 3. Connecting to Ubuntu via SSH

EC2 创建完成并进入 `Running` 状态以后，就可以使用 SSH 登录服务器。SSH 是 Linux 服务器最常见的远程管理方式，你实际上是在自己的电脑上打开一个终端，然后通过互联网连接到 AWS 中的 Ubuntu 系统。

如果你使用 Windows，可以打开 PowerShell 或 Windows Terminal。Assume your key file is named `MyEC2Key.pem`，and the server public IP is `203.0.113.10`，那么命令可以写成：

```bash
ssh -i "MyEC2Key.pem" ubuntu@203.0.113.10
```

If the key is on the Windows desktop, you can use a command such as:

```powershell
ssh -i "C:\Users\Administrator\Desktop\MyEC2Key.pem" ubuntu@203.0.113.10
```

macOS 或 Linux 用户可以先进入密钥所在目录，then run `chmod 400 MyEC2Key.pem` to restrict the private-key permissions，再执行：

```bash
ssh -i "MyEC2Key.pem" ubuntu@203.0.113.10
```

这里的 `ubuntu` 非常重要，因为 Ubuntu 官方 EC2 镜像通常使用 `ubuntu` 作为默认 SSH 用户。也就是说，第一次连接时不要按照一些普通 VPS 教程写成 `root@服务器IP`。如果使用的是 Ubuntu EC2 镜像，通常应该使用 `ubuntu@服务器IP`，然后在需要管理员权限时通过 `sudo` 执行命令。

第一次 SSH 连接时，终端可能会出现类似 `The authenticity of host ... can't be established` 的提示，这是 SSH 在询问你是否信任这台服务器。如果确认这是自己刚刚创建的 EC2 实例，可以输入 `yes` 继续。After a successful login, you will usually see something like `ubuntu@ip-xxx-xxx-xxx-xxx:~$` 的命令提示符。Seeing this prompt means you are now operating inside the Ubuntu server on AWS rather than on your local computer.

If SSH fails, check three things first.第一，Confirm that the EC2 instance is still `Running` and that you are using its current public IP；第二，check whether the Security Group allows TCP port 22；第三，and verify the `.pem` path and key are correct。If you see `Permission denied (publickey)`, focus on the username, key, and the Key Pair used when the instance was created.

---

## 4. Updating Ubuntu and Preparing the Environment

After successfully logging in to Ubuntu, update the package list and upgrade the system first.最常用的命令就是 `sudo apt update`，它负责更新软件包索引；之后使用 `sudo apt upgrade -y` 安装可用的软件更新。因此可以直接执行：

```bash
sudo apt update && sudo apt upgrade -y
```

After the update finishes, you can check the current Ubuntu version，执行 `lsb_release -a`；check the Linux kernel可以使用 `uname -a`；check memory usage可以使用 `free -h`；check disk space则可以使用 `df -h`。These commands are not required for V2Ray, but they are worth learning if you plan to manage Linux cloud servers long term.

If the system recommends restarting services or rebooting after a kernel update, follow the prompt.If you need to reboot the server, run `sudo reboot`，wait a few minutes, and reconnect via SSH.If your EC2 instance does not have a persistent public IP, its public IP may change after the instance is stopped and started again，所以重新连接前最好回 AWS 控制台确认当前 Public IPv4 address。

---

## 5. Installing V2Ray

Ubuntu 基础环境准备完成之后，就可以安装 V2Ray。V2Ray is a server-side network service, so installation alone is not enough; it must be configured before it can work as intended.Before installation, make sure the system has `curl`，如果没有，可以执行 `sudo apt install curl -y`。

V2Fly 项目提供了常见的安装脚本方式，可以使用：

```bash
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
```

Wait for the installer to finish.安装结束后，可以尝试执行 `v2ray version` to check the version。如果系统能够正常输出版本号，通常说明程序已经安装成功。

On Ubuntu, V2Ray can normally be managed as a systemd service.systemd 是 Linux 中非常重要的服务管理机制，因此以后你不需要每次都手动运行 V2Ray 程序，而是可以通过 `systemctl` 控制它。例如start the service with `sudo systemctl start v2ray`，stop it with `sudo systemctl stop v2ray`，restart it with `sudo systemctl restart v2ray`，check its status with `sudo systemctl status v2ray`。

To have V2Ray start automatically after a server reboot, run `sudo systemctl enable v2ray`。You can also use `sudo systemctl enable --now v2ray`，which enables automatic startup and starts the service immediately.

Remember: **successful installation does not mean successful configuration**.A newly installed V2Ray does not necessarily mean it is already listening on the port you need.Next, generate a UUID, edit the configuration file, and verify that the service can start correctly.

---

## 6. Generating a UUID

V2Ray 配置通常需要一个 UUID 作为客户端身份凭证。Linux 本身就可以生成 UUID，不需要额外安装工具。执行：

```bash
cat /proc/sys/kernel/random/uuid
```

The terminal will output a string similar to:

```text
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

The actual value will be different from the example.Copy the UUID generated by your server and store it securely.You will need to enter exactly the same UUID when configuring the client.

Although a UUID cannot be used to log in to the server like an SSH private key, it is still client authentication information, so avoid publicly publishing the server IP, port, and UUID together.If you suspect the UUID has been exposed, generate a new UUID and update both the server and client configurations.

---

## 7. Configuring V2Ray

After installation, edit the V2Ray configuration file.常见安装方式下，配置文件位于 `/usr/local/etc/v2ray/config.json`，可以使用 Nano 编辑：

```bash
sudo nano /usr/local/etc/v2ray/config.json
```

Nano 是 Ubuntu 中非常常见的终端文本编辑器。Inside Nano, use the arrow keys to move the cursor，`Ctrl + K` delete the current line，`Ctrl + O` save the file and press Enter to confirm，then press `Ctrl + X` to exit.

This guide first uses a basic VMess + WebSocket configuration as an example.Replace `你的UUID` below with the actual UUID you generated:

```json
{
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 10086,
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "你的UUID",
            "alterId": 0
          }
        ]
      },
      "streamSettings": {
        "network": "ws",
        "wsSettings": {
          "path": "/your-path"
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "direct"
    }
  ]
}
```

The most important parameters in this configuration are `port`、`protocol`、`id`、`network` 和 `path`。`port` determines which port V2Ray listens on，这里使用的是 `10086`；`protocol` indicates VMess；`id` is the UUID；`network` 设置为 `ws`，indicates WebSocket；`path` is the WebSocket path。

For example, if the server uses `/your-path`, the client must also use `/your-path`.If the server uses `/your-path` but the client uses `/v2ray`, the connection will not work with this configuration even if the IP, port, and UUID are all correct.

You also need to pay attention to JSON syntax. JSON is strict about brackets, quotation marks, and commas.例如下面这样的配置就存在错误：

```json
{
  "port": 10086,
  "protocol": "vmess",
}
```

因为最后一个字段后面多了一个逗号。正确写法应该是：

```json
{
  "port": 10086,
  "protocol": "vmess"
}
```

After editing the configuration, do not simply assume it is correct; validate it first.

---

## 8. Validating the Configuration and Starting V2Ray

If `jq` is installed, you can use it to validate the JSON format.If it is not installed, run `sudo apt install jq -y`，然后运行：

```bash
sudo jq . /usr/local/etc/v2ray/config.json
```

If the JSON is valid, `jq` will format and output it; if there is a syntax error, it will indicate the relevant location.

You can also test the configuration using the commands supported by your current V2Ray version.例如某些版本可以使用：

```bash
v2ray test -config /usr/local/etc/v2ray/config.json
```

如果你的版本命令格式有所不同，可以通过 `v2ray help` 查看当前版本支持的命令。

After confirming that there are no obvious configuration errors, restart V2Ray:

```bash
sudo systemctl restart v2ray
```

Then check:

```bash
sudo systemctl status v2ray
```

If you see something like:

```text
Active: active (running)
```

the V2Ray service is running.

If it shows `failed`, do not keep restarting it repeatedly; check the logs instead.最常用的命令是：

```bash
sudo journalctl -u v2ray --no-pager -n 100
```

To watch the logs in real time, use:

```bash
sudo journalctl -u v2ray -f
```

Then test the client connection and observe whether the server receives the connection and what errors appear.

---

## 9. Checking Whether Port 10086 Is Actually Listening

A V2Ray service showing `active (running)` does not necessarily mean the expected port is listening, so verify it at the Linux system level.

执行：

```bash
sudo ss -lntp | grep 10086
```

If you see something like:

```text
LISTEN 0 4096 0.0.0.0:10086 0.0.0.0:*
```

the system is listening on TCP 10086.

If there is no output, no process is currently listening on that port.At that point, check the V2Ray service status and configuration rather than continuing to change the AWS Security Group.

This is important because a network connection passes through multiple layers:

```text
客户端
   ↓
互联网
   ↓
AWS Security Group
   ↓
EC2 Ubuntu
   ↓
Ubuntu 防火墙
   ↓
10086 端口
   ↓
V2Ray
```

任何一层出现问题，都可能导致客户端连接失败。

---

## 10. Configuring the AWS Security Group

Now that the server is configured and V2Ray is confirmed to be listening on `10086`, make sure AWS allows external connections to this port.

Open the AWS EC2 console, find the corresponding instance, and open its Security Group.在 `Inbound rules` 中确认 SSH 的 TCP 22 端口以及 V2Ray 使用的 TCP 10086 端口符合你的实际需求。

例如：

```text
Type: SSH
Protocol: TCP
Port: 22
Source: 你的管理 IP
```

以及：

```text
Type: Custom TCP
Protocol: TCP
Port: 10086
Source: 根据你的实际访问需求设置
```

If the V2Ray service needs to be reachable from the public Internet, the service port must allow the required public sources; if it is used only from a specific network environment, you can restrict the source further.

Again, **do not open port 7890 in AWS simply because the client software uses 7890.**假设你的 Windows 电脑上 V2rayN 使用 `127.0.0.1:7890` 作为本地代理端口，那么它表示：

```text
你的电脑
   │
   └── 127.0.0.1:7890
```

Here, `127.0.0.1` refers to the local computer itself, and 7890 is the client's local listening port.It does not directly correspond to the V2Ray server port on AWS EC2.

If V2Ray on EC2 is configured to use `10086`, the AWS Security Group should concern itself with TCP 10086, not TCP 7890.因此原先很多教程中同时开放 `10086` 和 `7890` 的做法并不是必要的，尤其不要为了客户端本地端口而把 7890 暴露给整个互联网。

---

## 11. Ubuntu UFW Firewall

除了 AWS Security Group 以外，Ubuntu can also have its own firewall.常见工具是 UFW，可以使用 `sudo ufw status` 查看当前状态。

If you plan to enable UFW, allow SSH first; otherwise, you can easily lock yourself out of the server after enabling the firewall.可以先执行：

```bash
sudo ufw allow 22/tcp
sudo ufw allow 10086/tcp
```

然后再启用：

```bash
sudo ufw enable
```

最后检查：

```bash
sudo ufw status
```

If the corresponding ports are shown as allowed, UFW is operating according to the current configuration.

需要理解的是，AWS Security Group 和 UFW 是两个不同层级的防火墙：

```text
Internet
    ↓
AWS Security Group
    ↓
EC2
    ↓
Ubuntu UFW
    ↓
V2Ray
```

因此即使 AWS Security Group 已经放行 10086，如果 Ubuntu UFW 阻止了它，连接依然可能失败；反过来也一样。如果 AWS 没有放行，即使 Ubuntu UFW 允许，也无法从公网正常访问。

---

## 12. Configuring the V2rayN Client

Once the server side is complete, configure the corresponding server in the Windows client.Menu names may vary between V2rayN versions, but the core parameters are generally the same.When adding a server, enter its public IP, port, UUID, transport method, and WebSocket Path.

假设服务器信息如下：

```text
Address:
你的 AWS 公网 IPv4

Port:
10086

Protocol:
VMess

UUID:
服务器生成的 UUID

Network:
WebSocket / ws

Path:
/your-path
```

Set Address to the EC2 instance's current public IPv4 address, Port to `10086` to match the server configuration, UUID to exactly the same UUID in the server configuration, Network to WebSocket, and Path to `/your-path`.

客户端与服务器之间可以简单理解为：

```text
服务器：

Port = 10086
UUID = A
Network = ws
Path = /your-path


客户端：

Port = 10086
UUID = A
Network = ws
Path = /your-path
```

In other words, the address, port, UUID, transport method, and Path must match between the two sides.

---

## 13. Cannot Connect? Troubleshoot in This Order

If the client cannot connect, do not immediately reinstall V2Ray.When a network service has problems, the most effective approach is to troubleshoot layer by layer from the outside inward.

First, confirm in the AWS console that the EC2 instance is still `Running`, then verify that the current Public IPv4 is correct.如果实例曾经停止并重新启动，而没有配置固定公网 IP，那么公网 IP 可能已经发生变化，客户端继续使用旧 IP 就会直接连接失败。

Then check the AWS Security Group and confirm that TCP 10086 is allowed.If UFW is enabled, run `sudo ufw status` and make sure the local Ubuntu firewall is not blocking the port.

Next, log in to the server and run `sudo systemctl status v2ray` to confirm the service status.如果服务没有运行，则执行 `sudo journalctl -u v2ray --no-pager -n 100` 查看日志。

then run `sudo ss -lntp | grep 10086`，确认 10086 是否真正处于监听状态。如果没有监听，那么问题通常发生在 V2Ray 配置或者服务启动阶段；如果已经监听，那么继续检查 AWS 网络层以及客户端配置。

Finally, check the client parameters, including the server public IP, port, UUID, WebSocket, and Path.UUID and Path are especially prone to copy-and-paste errors that can cause connection failures.

可以按照下面这个顺序排查：

```text
EC2 是否 Running
        ↓
公网 IP 是否正确
        ↓
AWS Security Group 是否允许
        ↓
Ubuntu UFW 是否允许
        ↓
V2Ray 是否运行
        ↓
10086 是否监听
        ↓
UUID 是否一致
        ↓
WebSocket 是否一致
        ↓
Path 是否一致
        ↓
客户端再次测试
```

这样排查通常比不断修改配置更加有效。

---

## 14. Common Errors and Their Causes

If the client reports `Connection refused`, it usually means the destination server is reachable but the target port is not accepting connections.At this point, check whether V2Ray is running and whether `10086` is listening.If you get `Connection timeout`, focus on the public IP, AWS Security Group, Ubuntu firewall, and network path.

If V2Ray is running normally on the server but the client reports an authentication error, check the UUID first.客户端的 UUID 必须与服务器配置文件中的 `id` 完全一致。

If you use WebSocket, the client and server Path must also match.例如服务器使用 `/your-path`，客户端却填写 `/v2ray`，就会出现连接问题。

If V2Ray fails to start, check:

```bash
sudo journalctl -u v2ray --no-pager -n 100
```

If the logs show JSON, configuration-field, or port-related errors, return to `/usr/local/etc/v2ray/config.json` and inspect it.Do not change a dozen parameters at once. Ideally, change one thing at a time, validate the configuration, restart the service, and observe the logs.

---

## 15. Security Recommendations

If you are only learning and experimenting temporarily, the basic configuration above can help you understand the relationship between EC2, Ubuntu, V2Ray, ports, and the client.If you plan to operate the server long term, however, you should consider additional security measures.

First, do not expose your SSH private key.`.pem` 文件一旦被别人获得，就可能造成严重的服务器安全问题。Second, do not casually publish authentication information such as the UUID and server address.Third, restrict SSH access by source whenever possible, and do not expose ports that have no practical purpose.

On the system side, periodically run `sudo apt update && sudo apt upgrade -y` to update Ubuntu packages, and regularly check the V2Ray service status, disk space, and system logs.For long-running servers, also monitor your AWS billing and resource usage so that forgotten test instances, storage, public IPs, or other resources do not generate unexpected charges.

For a long-term deployment, you can also build a more complete network-service architecture using a domain, TLS, Nginx, or Caddy.不过这已经属于进阶内容，不建议第一次部署的时候一次性把所有组件全部加入，否则出现问题之后很难判断具体是哪一层出现了故障。

---

## 16. Domains, TLS, and Reverse Proxying: Advanced Topics

基础架构通常可以理解为：

```text
客户端
   ↓
公网 IP
   ↓
EC2
   ↓
V2Ray
```

如果继续升级，可以变成：

```text
客户端
   ↓
域名
   ↓
DNS
   ↓
EC2
   ↓
Nginx / Caddy
   ↓
TLS / HTTPS
   ↓
WebSocket
   ↓
V2Ray
```

This architecture involves additional concepts such as DNS resolution, domain management, TLS certificates, HTTPS, reverse proxying, and Web server configuration.These are not only V2Ray concepts; they are also common technologies used to deploy websites and API services.

例如以后你想在同一台 EC2 上部署个人博客、API、Web 服务和其他应用，就可以进一步学习 Nginx、Caddy、Docker、Cloudflare DNS 等技术。这样你掌握的就不再只是“如何安装一个 V2Ray”，而是一套可以迁移到个人网站、API 服务、Docker 项目以及其他云服务器应用中的完整部署思路。

---

## 17. Common Linux and V2Ray Commands

When managing the server later, you do not need to search for these commands every time. The following basic commands can serve as a day-to-day reference.

更新系统：

```bash
sudo apt update && sudo apt upgrade -y
```

查看 Ubuntu 版本：

```bash
lsb_release -a
```

查看内存：

```bash
free -h
```

查看磁盘：

```bash
df -h
```

查看端口：

```bash
sudo ss -lntp
```

查看 V2Ray 状态：

```bash
sudo systemctl status v2ray
```

启动 V2Ray：

```bash
sudo systemctl start v2ray
```

停止 V2Ray：

```bash
sudo systemctl stop v2ray
```

重启 V2Ray：

```bash
sudo systemctl restart v2ray
```

设置开机启动：

```bash
sudo systemctl enable v2ray
```

查看最近日志：

```bash
sudo journalctl -u v2ray --no-pager -n 100
```

实时查看日志：

```bash
sudo journalctl -u v2ray -f
```

查看指定端口：

```bash
sudo ss -lntp | grep 10086
```

查看 UFW：

```bash
sudo ufw status
```

---

## 18. Complete Deployment Checklist

If you have completed the guide, use the following checklist for a final review:

```text
AWS：

[✓] AWS 账号正常
[✓] EC2 实例已经创建
[✓] Ubuntu Server 正常运行
[✓] SSH Key 已保存
[✓] Public IPv4 已确认
[✓] Security Group 已配置


SSH：

[✓] 可以使用 SSH 登录
[✓] 使用 ubuntu 用户
[✓] PEM 私钥路径正确


Ubuntu：

[✓] 系统已经更新
[✓] curl 已安装
[✓] UFW 配置正确（如果启用）


V2Ray：

[✓] V2Ray 已安装
[✓] UUID 已生成
[✓] config.json 已修改
[✓] JSON 格式正确
[✓] V2Ray 服务正常运行
[✓] 10086 正在监听


客户端：

[✓] AWS 公网 IP 正确
[✓] Port = 10086
[✓] UUID 与服务器一致
[✓] Network = WebSocket
[✓] Path = /your-path


网络：

[✓] AWS Security Group 允许服务端口
[✓] Ubuntu 防火墙允许服务端口
[✓] SSH 可以正常使用
```

---

## Conclusion

From creating an AWS EC2 server to getting the client connected successfully, the key skill is not memorizing a single software-installation command but understanding the overall process of deploying and managing a cloud server.首先通过 AWS EC2 获得一台云服务器，然后使用 SSH 远程进入 Ubuntu；接着更新系统并安装 V2Ray，再通过 UUID、端口、协议和传输方式配置服务；随后通过 AWS Security Group 和 Ubuntu 防火墙控制网络访问，最后在客户端填写与服务器对应的参数。

The entire process can be summarized as:

```text
AWS EC2
   ↓
Ubuntu
   ↓
SSH
   ↓
V2Ray
   ↓
config.json
   ↓
UUID + Port + Protocol + WebSocket
   ↓
AWS Security Group
   ↓
客户端
```

One of the most important concepts to understand is the idea of a “port.”The server-side V2Ray `10086` is a listening port on the server, while ports such as `7890` used by client software are usually local proxy ports. They are not the same thing.Once you understand the relationship between server ports, client ports, the AWS Security Group, and Ubuntu UFW, many seemingly complicated network problems become much easier to troubleshoot.

如果以后准备继续深入，可以在这个基础上学习域名、DNS、TLS、Nginx、Caddy、Docker、Cloudflare 以及 Linux 服务器安全等内容。这样你掌握的就不再只是“如何安装一个 V2Ray”，而是一套可以迁移到个人网站、API 服务、Docker 项目以及其他云服务器应用中的完整部署思路。

Finally, remember that the use of cloud servers and network services should comply with applicable local laws, AWS Terms of Service, and other relevant rules. This guide is primarily intended for learning about AWS EC2, Ubuntu, Linux service management, and network configuration.
