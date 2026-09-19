---
title: AWS EC2 + Ubuntu: Deploying V2Ray
date: 2026-9-16
tags: ["AWS", "EC2", "Ubuntu", "V2Ray", "Linux", "Networking"]
description: A concise guide to deploying and configuring V2Ray on an AWS EC2 Ubuntu server.
---

# AWS EC2 + Ubuntu: A Practical Guide to Deploying V2Ray

## Introduction

This guide documents a complete, from-scratch deployment workflow for running V2Ray on an Ubuntu server hosted by AWS EC2. For beginners, the difficult part is often not any single command, but understanding how the different layers fit together: the EC2 instance, SSH access, AWS security groups, the Ubuntu firewall, listening ports, V2Ray itself, and the client. For example, SSH may work while the V2Ray client cannot connect, or a client may use `127.0.0.1:7890` locally even though the server-side V2Ray service listens on `1080`. These are not contradictory situations; they involve different network layers and different ports.

Rather than presenting a collection of commands, this article follows the deployment process in order: creating an EC2 instance, preparing Ubuntu, connecting through SSH, configuring the security group, installing V2Ray, generating a UUID, editing the configuration, starting and checking the service, configuring a client, and troubleshooting common connection problems. You can read it from beginning to end as a hands-on tutorial or jump directly to the section relevant to a problem you are troubleshooting.

The baseline environment is Ubuntu Server LTS on AWS EC2. AWS changes its console interface, instance types, pricing, and free-tier policies over time, and V2Ray clients may also change their interfaces. If the labels in your console differ slightly from those shown here, look for the corresponding function rather than following the wording literally. AWS resources such as instances, storage, public IP addresses, and data transfer may incur charges, so check the pricing and eligibility that apply to your account before creating a server. This article is primarily intended for learning about Linux cloud servers, network services, and server administration. Use cloud resources and network services in accordance with applicable laws, regulations, and the AWS Terms of Service.

> **Note:** The V2Ray configuration used in this guide is intentionally basic and is presented as a learning example. Production deployments may require additional security measures and a different architecture, such as TLS and a reverse proxy.

---

## 1. Prerequisites

Before you begin, you need an AWS account, a computer with Internet access, and a secure location for storing your AWS SSH private key. Windows users can use PowerShell or Windows Terminal, while macOS and Linux users can use the built-in terminal. On the server side, this guide uses AWS EC2 with Ubuntu Server. On the client side, use a client that supports the protocol and transport settings described in this guide.

The entire deployment process can be simplified into the following chain:

```text
Your computer
   │
   │ SSH / Client connection
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

AWS EC2 provides the server environment, Ubuntu is the operating system running on EC2, V2Ray is the network service actually running on the server, and the AWS security group and Ubuntu firewall control whether external networks can access specified ports on the server.

Before starting, it is important to understand one concept: **the client's local proxy port and the server's service port are two different things.** For example, some clients may use `127.0.0.1:7890` as the local proxy entry point on your computer. That `7890` belongs to your own computer; it does not mean that the AWS server needs to listen on or expose `7890`. If V2Ray on the server actually listens on `1080`, the server needs to listen on `1080`; it does not need to expose the client's local `7890`.

---

## 2. Creating an AWS EC2 Instance

After signing in to the AWS Management Console, enter `EC2` in the search bar at the top, open the EC2 service page, and find **Launch instance**. When creating the instance, you can first set an instance name such as `My-V2Ray-Server`, `My-VPS`, or `Ubuntu-Server`. This name is mainly for your own identification and does not affect V2Ray later.

Next, choose the operating system image, or AMI. This article uses Ubuntu Server as an example, and it is generally preferable to choose a stable LTS version currently offered in the AWS console. LTS means Long Term Support, which is usually more suitable for server environments. If you see versions such as Ubuntu 22.04 LTS or Ubuntu 24.04 LTS, choose one based on current software compatibility and your actual needs. Do not assume you must use exactly the version mentioned in an older tutorial.

For the instance type, if you are only learning Linux, testing V2Ray, or running very lightweight services, you can choose a small instance type currently offered by AWS. However, AWS instance types and free-tier policies change. The `t2.micro`, which appears frequently in older tutorials, is not necessarily free at all times, in all regions, or for all accounts. When creating the instance, use the eligibility and pricing shown in your own AWS console as the reference.

Next, create an SSH key pair. Find the key pair option, create a new key pair such as `MyEC2Key`, choose an appropriate key format, and download it. For Linux/macOS/Windows OpenSSH environments, `.pem` is a common format. After downloading it, store it securely, for example in a dedicated AWS key directory on your computer. This file is extremely important because you will need it to log in to the server via SSH. Do not upload the `.pem` file to GitHub, public cloud storage, forums, or any other public location, and never commit it to a code repository.

After configuring the key, move on to the network settings. The AWS security group deserves particular attention. You can think of a security group as a network firewall around the EC2 instance. It determines which sources can access which ports on the server. SSH uses TCP port 22 by default, so your computer must be allowed to access the server through TCP 22. If V2Ray later listens on TCP 1080, external access to TCP 1080 must also be allowed according to your actual use case.

If you are only using SSH to manage the server yourself, it is better not to leave SSH open to the entire world. For example, you can restrict the source of TCP 22 to your current public IP and use `/32` to represent a single IP address. If your public IP changes frequently, adjust the rule as needed, but do not permanently allow all sources simply for convenience.

Pay particular attention to not opening large numbers of unnecessary ports. If the server only needs SSH and V2Ray, there is usually no need to open a collection of unused ports such as `8080`, `8888`, `7890`, and `12345`. The principle should be “open only what is needed,” rather than “open everything first and deal with it later.”

Finally, review the instance name, Ubuntu image, instance type, key pair, network settings, and storage configuration. Once everything is correct, click “Launch instance.” Wait until the instance status becomes `Running`, then open the instance details and find `public IPv4 address`. This is the public IPv4 address you may use later for SSH access and client connections.

---

## 3. Connecting to Ubuntu via SSH

Once the EC2 instance has been created and is in the `Running` state, you can use SSH to log in. SSH is one of the most common ways to remotely manage Linux servers. In practice, you open a terminal on your own computer and connect over the Internet to the Ubuntu system on AWS.

If you are using Windows, open PowerShell or Windows Terminal. Assuming your key file is named `MyEC2Key.pem` and the server's public IP is `203.0.113.10`, the command is:

```bash
ssh -i "MyEC2Key.pem" ubuntu@203.0.113.10
```

If the key file is on the Windows desktop, you can use something like:

```powershell
ssh -i "C:\Users\Administrator\Desktop\MyEC2Key.pem" ubuntu@203.0.113.10
```

macOS or Linux users can first enter the directory containing the key, run `chmod 400 MyEC2Key.pem` to restrict the private key permissions, and then run:

```bash
ssh -i "MyEC2Key.pem" ubuntu@203.0.113.10
```

The `ubuntu` username is important because official Ubuntu EC2 images typically use `ubuntu` as the default SSH user. In other words, do not follow generic VPS tutorials and use `root@server-ip` for the first connection. With an Ubuntu EC2 image, you will normally use `ubuntu@server-ip` and then use `sudo` when administrator privileges are required.

During the first SSH connection, the terminal may display a message such as `The authenticity of host ... can't be established`. SSH is asking whether you trust the server. If you have confirmed that this is the EC2 instance you just created, enter `yes` to continue. After a successful login, you will typically see a prompt such as `ubuntu@ip-xxx-xxx-xxx-xxx:~$`. Once you see this prompt, you are operating inside the Ubuntu server on AWS rather than on your own computer.

If the SSH connection fails, check three things first. First, confirm that the EC2 instance is still in the `Running` state and that you are using its current public IP. Second, check whether the security group allows TCP port 22. Third, verify the `.pem` file path and key. If you see `Permission denied (publickey)`, focus on the username, key, and the key pair used when the instance was created.

---

## 4. Updating Ubuntu and Preparing the Environment

After successfully logging in to Ubuntu, it is recommended to update the package list and upgrade the system first. The most common command is `sudo apt update`, which updates the package index, followed by `sudo apt upgrade -y` to install available updates. You can therefore run:

```bash
sudo apt update && sudo apt upgrade -y
```

After the system update is complete, you can check the Ubuntu version with `lsb_release -a`. Use `uname -a` to view the Linux kernel, `free -h` to check memory usage, and `df -h` to check disk space. These commands are not required for V2Ray, but they are worth learning if you plan to use Linux cloud servers long term.

If the system indicates that certain services need to be restarted or recommends a reboot after a kernel update, follow the prompt. If you need to reboot the server, run `sudo reboot`, wait a few minutes, and reconnect via SSH. Note that if your EC2 instance does not have a fixed public IP, its public IP may change after the instance is stopped and started again. Before reconnecting, it is therefore best to confirm the current public IPv4 address in the AWS console.

---

## 5. Installing V2Ray

Once the Ubuntu environment is ready, you can install V2Ray. V2Ray is a server-side network service, so further configuration is required after installation before it can work as intended. Before installing, make sure `curl` is available; if not, run `sudo apt install curl -y`.

V2Fly provides a common installation-script method. You can use:

```bash
bash <(curl -L https://raw.githubusercontent.com/v2fly/fhs-install-v2ray/master/install-release.sh)
```

Wait for the installer to finish. After installation, you can run `v2ray version` to check the version. If the system outputs a version number normally, the program has generally been installed successfully.

On Ubuntu, V2Ray can usually be managed as a systemd service. systemd is an important service-management mechanism in Linux, so you do not need to manually run V2Ray every time. Instead, you can control it with `systemctl`. For example, start it with `sudo systemctl start v2ray`, stop it with `sudo systemctl stop v2ray`, restart it with `sudo systemctl restart v2ray`, and check its status with `sudo systemctl status v2ray`.

If you want V2Ray to start automatically after the server reboots, run `sudo systemctl enable v2ray`. You can also use `sudo systemctl enable --now v2ray`, which enables automatic startup and starts the service immediately.

Keep in mind that **successful installation does not mean successful configuration**. A newly installed V2Ray is not necessarily listening on the port you need. You still need to generate a UUID, modify the configuration file, and verify that the service can start normally.

---

## 6. Generating a UUID

V2Ray configuration typically requires a UUID as a client authentication credential. Linux can generate a UUID without any additional tools. Run:

```bash
cat /proc/sys/kernel/random/uuid
```

The terminal will output a string similar to:

```text
xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

The actual generated value will be different from the example. Copy the UUID generated on your own server and store it securely. You will need to enter exactly the same UUID when configuring the client later.

Although a UUID cannot be used to log in to the server in the same way as an SSH private key, it is still client authentication information. Therefore, it is not recommended to publicly disclose the server IP, port, and UUID together. If you suspect that the UUID has been exposed, generate a new UUID and update both the server-side and client-side configurations.

---

## 7. Configuring V2Ray

After installation, you need to edit the V2Ray configuration file. With common installation methods, the configuration file is located at `/usr/local/etc/v2ray/config.json`, and you can edit it with Nano:

```bash
sudo nano /usr/local/etc/v2ray/config.json
```

Nano is a common terminal text editor in Ubuntu. Once inside, use the arrow keys to move the cursor, `Ctrl + K` to delete the current line, `Ctrl + O` to save the file and then press Enter to confirm, and finally `Ctrl + X` to exit.

This article first uses a relatively basic VMess + WebSocket configuration as an experimental example. Replace `YOUR-UUID` below with the actual UUID you generated:

```json
{
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 1080,
      "protocol": "vmess",
      "settings": {
        "clients": [
          {
            "id": "YOUR-UUID",
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

The most important parameters in this configuration are `port`, `protocol`, `id`, `network`, and `path`. `port` determines which port the V2Ray service listens on; here it is `1080`. `protocol` specifies VMess; `id` is the UUID; `network` is set to `ws`, meaning WebSocket; and `path` is the WebSocket path.

For example, if the server configuration uses `/your-path`, the client must also use `/your-path`. If the server uses `/your-path` while the client enters `/v2ray`, the connection will not work with this configuration even if the IP address, port, and UUID are all correct.

JSON syntax also requires careful attention. JSON is strict about brackets, quotation marks, and commas. For example, the following configuration is incorrect:

```json
{
  "port": 1080,
  "protocol": "vmess",
}
```

The problem is the extra comma after the last field. The correct form is:

```json
{
  "port": 1080,
  "protocol": "vmess"
}
```

After modifying the configuration file, validate it before restarting the service.

---

## 8. Checking the Configuration and Starting V2Ray

If `jq` is already installed, you can use it to check the JSON format. If it is not installed, run `sudo apt install jq -y`, then run:

```bash
sudo jq . /usr/local/etc/v2ray/config.json
```

If the JSON is valid, `jq` will reformat and output the configuration. If there is a syntax error, it will indicate the relevant error location.

You can also test the configuration using commands supported by your current V2Ray version. For example, some versions support:

```bash
v2ray test -config /usr/local/etc/v2ray/config.json
```

If the command format differs in your version, use `v2ray help` to see the commands supported by the current version.

Once you have confirmed that there are no obvious configuration errors, restart V2Ray:

```bash
sudo systemctl restart v2ray
```

Then check:

```bash
sudo systemctl status v2ray
```

If you see:

```text
Active: active (running)
```

the V2Ray service is running.

If it shows `failed`, do not keep running restart repeatedly. Check the logs instead. The most common command is:

```bash
sudo journalctl -u v2ray --no-pager -n 100
```

To monitor the logs in real time, use:

```bash
sudo journalctl -u v2ray -f
```

Then perform a client connection test. This allows you to observe whether the server receives the connection and what errors occur.

---

## 9. Checking Whether Port 1080 Is Actually Listening

A V2Ray service showing `active (running)` does not necessarily mean that the port is listening correctly, so you should also verify it at the Linux system level.

Run:

```bash
sudo ss -lntp | grep 1080
```

If you see something like:

```text
LISTEN 0 4096 0.0.0.0:1080 0.0.0.0:*
```

This indicates that the system is listening on TCP port 1080.

If there is no output, no program is currently listening on this port. At this point, check the V2Ray service status and configuration file again instead of continuing to modify the AWS security group.

This is important because a network connection actually passes through multiple layers:

```text
client
   ↓
Internet
   ↓
AWS security group
   ↓
EC2 Ubuntu
   ↓
Ubuntu firewall
   ↓
1080 port
   ↓
V2Ray
```

A problem at any layer can cause the client connection to fail.
---

## 10. Configuring the AWS security group

Now that the server-side configuration is complete and V2Ray is confirmed to be listening on `1080`, you need to make sure AWS allows external connections to this port.

Open the AWS EC2 console, find the corresponding instance, and open its security group. Under `Inbound rules`, confirm that SSH TCP port 22 and the TCP 1080 port used by V2Ray are configured according to your actual needs.

For example:

```text
Type: SSH
Protocol: TCP
Port: 22
Source: your management IP
```

And:

```text
Type: Custom TCP
Protocol: TCP
Port: 1080
Source: Set it according to your actual access needs
```

If the V2Ray service needs to be accessed from the public Internet, the corresponding service port must allow the appropriate public sources. If it is only used from a specific network environment, you can restrict the sources further.

Again, **do not open port 7890 in AWS just because the client software uses 7890.** Suppose v2rayN on your Windows computer uses `127.0.0.1:7890` as its local proxy port. This means:

```text
Your computer
   │
   └── 127.0.0.1:7890
```

Here, `127.0.0.1` refers to the current computer itself, and 7890 is the client's local listening port. It has no direct port-to-port relationship with the V2Ray service on AWS EC2.

If V2Ray on EC2 is configured to use `1080`, the AWS security group should concern itself with TCP 1080, not TCP 7890. Therefore, the practice found in many older tutorials of opening both `1080` and `7890` is unnecessary. In particular, do not expose 7890 to the entire Internet just because it is a local client port.

---

## 11. Ubuntu UFW Firewall

In addition to the AWS security group, Ubuntu itself can also have a firewall configured. A common tool is UFW, whose current status can be checked with `sudo ufw status`.

If you plan to enable UFW, allow SSH first. Otherwise, enabling the firewall can lock you out of the server. Run:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 1080/tcp
```

Then enable it:

```bash
sudo ufw enable
```

Finally, check:

```bash
sudo ufw status
```

If the corresponding ports are shown as allowed, UFW is working according to the current configuration.

It is important to understand that the AWS security group and UFW are firewalls at different layers:

```text
Internet
    ↓
AWS security group
    ↓
EC2
    ↓
Ubuntu UFW
    ↓
V2Ray
```

So even if the AWS security group has allowed port 1080, if Ubuntu UFW blocks it, the connection will still fail; and vice versa. If AWS does not allow it, access will still fail even if Ubuntu UFW permits the connection.

---

## 12. Configuring the v2rayN Client

Once the server side is complete, you can configure the corresponding server in the Windows client. Menu names may vary between v2rayN versions, but the core parameters are essentially the same. When adding a server, enter the server's public IP, port, UUID, transport method, and WebSocket path.

Suppose the server information is as follows:

```text
Address:
your AWS public IPv4

Port:
1080

Protocol:
VMess

UUID:
UUID generated by the server

network:
WebSocket / ws

path:
/your-path
```

Enter the current AWS EC2 public IPv4 address in Address. Port must match `1080` in the server configuration, UUID must exactly match the UUID in the server configuration file, network should be set to WebSocket, and path should be `/your-path`.

The relationship between the client and server can be understood simply as:

```text
Server:

Port = 1080
UUID = A
network = ws
path = /your-path


Client:

Port = 1080
UUID = A
network = ws
path = /your-path
```

In other words, the address, port, UUID, transport method, path, and other connection parameters must correspond correctly.

---

## 13. Troubleshooting: Check the Layers in Order

If the client cannot connect, do not immediately reinstall V2Ray. Troubleshoot the stack layer by layer, starting with the external network and moving inward to the service and client configuration.

First, open the AWS console and confirm that the EC2 instance is still `Running`, then verify that the current public IPv4 is correct. If the instance was stopped and started again without a fixed public IP, the public IP may have changed, and the client will fail immediately if it continues using the old IP.

Then check the AWS security group and confirm that TCP 1080 is allowed. If UFW is enabled, also run `sudo ufw status` to confirm that the local Ubuntu firewall is not blocking the port.

Next, log in to the server and run `sudo systemctl status v2ray` to check the service status. If the service is not running, use `sudo journalctl -u v2ray --no-pager -n 100` to view the logs.

Then run `sudo ss -lntp | grep 1080` to confirm that 1080 is actually listening. If it is not listening, the problem is usually in the V2Ray configuration or service startup stage. If it is listening, continue checking the AWS network layer and client configuration.

Finally, check the client parameters, including the server's public IP, port, UUID, WebSocket settings, and path. UUID and path in particular can easily cause connection failures when copied incorrectly.

You can troubleshoot in the following order:

```text
Is the EC2 running
        ↓
Is the public IP correct
        ↓
Does the AWS security group allow it
        ↓
Does Ubuntu UFW allow it
        ↓
Is V2Ray running
        ↓
Is 1080 listening
        ↓
Are the UUIDs the same
        ↓
Is the WebSocket configuration consistent
        ↓
Are the paths the same
        ↓
Client testing again
```

This approach is usually more effective than repeatedly changing configuration values.

---

## 14. Common Error Analysis

If the client reports `Connection refused`, it usually means the target server is reachable but the corresponding port is not accepting connections normally. Focus on whether V2Ray is running and whether `1080` is listening. If you see `Connection timeout`, pay more attention to the public IP, AWS security group, Ubuntu firewall, and network path.

If V2Ray is running normally on the server but the client reports an authentication-related error, focus on the UUID. The client's UUID must exactly match the `id` in the server configuration file.

When using WebSocket, the client and server path must also match. For example, if the server uses `/your-path` while the client enters `/v2ray`, the connection will fail.

If the V2Ray service fails to start, check the following first:

```bash
sudo journalctl -u v2ray --no-pager -n 100
```

If the logs contain errors related to JSON, configuration fields, or ports, return to `/usr/local/etc/v2ray/config.json` and inspect it. Do not change a dozen parameters at once. It is better to change one thing at a time, then recheck the configuration, restart the service, and observe the logs.

---

## 15. Security Recommendations

If you are only learning and experimenting temporarily, the basic configuration above can help you understand the relationship between EC2, Ubuntu, V2Ray, ports, and clients. However, if you plan to run the server long term, you should consider security in greater depth.

First, never expose your SSH private key. If someone obtains the `.pem` file, it can create a serious server security risk. Second, do not casually publish authentication information such as the UUID and server address. Third, restrict access to the SSH port as much as possible and do not open ports that have no practical purpose.

On the system side, regularly run `sudo apt update && sudo apt upgrade -y` to update Ubuntu packages, and periodically check the V2Ray service status, disk space, and system logs. Once the server has been running for a while, also monitor AWS billing and resource usage to avoid unexpected charges from forgotten test instances, disks, public IPs, or other resources.

If the server is intended for long-term use, you can further build a more complete network service architecture using a domain, TLS, Nginx, or Caddy. However, this is advanced material. It is not recommended to add every component during your first deployment, because once something goes wrong, it becomes much harder to determine which layer is responsible.

---

## 16. Domains, TLS, and Reverse Proxy: Advanced Topics

The basic architecture can be understood as:

```text
client
   ↓
public IP
   ↓
EC2
   ↓
V2Ray
```

With further upgrades, it can become:

```text
client
   ↓
domain name
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

This architecture involves additional knowledge, including DNS resolution, domain management, TLS certificates, HTTPS, reverse proxies, and web server configuration. These are not only V2Ray concepts; they are also common technologies used when deploying websites and API services.

For example, if you later want to deploy a personal blog, API, Web service, and other applications on the same EC2 instance, you can continue learning technologies such as Nginx, Caddy, Docker, and Cloudflare DNS. You will then have learned more than just “how to install V2Ray”; you will have developed a complete deployment approach that can be transferred to personal websites, API services, Docker projects, and other cloud-server applications.

---

## 17. Common Linux and V2Ray Commands

When managing the server later, you do not need to search for commands every time. The following basic commands can be used as a daily maintenance reference.

Update the system:

```bash
sudo apt update && sudo apt upgrade -y
```

Check the Ubuntu version:

```bash
lsb_release -a
```

Check memory:

```bash
free -h
```

Check disk space:

```bash
df -h
```

Check ports:

```bash
sudo ss -lntp
```

Check V2Ray status:

```bash
sudo systemctl status v2ray
```

Start V2Ray:

```bash
sudo systemctl start v2ray
```

Stop V2Ray:

```bash
sudo systemctl stop v2ray
```

Restart V2Ray:

```bash
sudo systemctl restart v2ray
```

Enable startup on boot:

```bash
sudo systemctl enable v2ray
```

View recent logs:

```bash
sudo journalctl -u v2ray --no-pager -n 100
```

View logs in real time:

```bash
sudo journalctl -u v2ray -f
```

Check a specific port:

```bash
sudo ss -lntp | grep 1080
```

Check UFW:

```bash
sudo ufw status
```

---

## 18. Complete Deployment Checklist

If you have completed the procedure in this article, use the following checklist for a final review:

```text
AWS:

[✓] AWS account is working
[✓] EC2 instance has been created
[✓] Ubuntu Server is running normally
[✓] SSH key has been saved
[✓] public IPv4 has been confirmed
[✓] security group has been configured


SSH:

[✓] SSH login works
[✓] Using the `ubuntu` user
[✓] PEM private-key path is correct


Ubuntu:

[✓] System has been updated
[✓] `curl` is installed
[✓] UFW is configured correctly (if enabled)


V2Ray:

[✓] V2Ray has been installed
[✓] UUID has been generated
[✓] `config.json` has been modified
[✓] JSON format is valid
[✓] V2Ray service is running normally
[✓] Port 1080 is listening


Client:

[✓] AWS public IP is correct
[✓] Port = 1080
[✓] UUID matches the server
[✓] network = WebSocket
[✓] path = /your-path


network:

[✓] AWS security group allows the service port
[✓] Ubuntu firewall allows the service port
[✓] SSH works normally
```

---

## Conclusion

From creating an AWS EC2 server to successfully connecting the client, what you really need to understand is not a single software installation command, but an entire cloud-server deployment mindset. First, obtain a cloud server through AWS EC2 and remotely access Ubuntu using SSH. Then update the system and install V2Ray, configure the service using the UUID, port, protocol, and transport method, control network access through the AWS security group and Ubuntu firewall, and finally enter the corresponding server parameters in the client.

The entire process can be condensed into:

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
AWS security group
   ↓
Client
```

The concept of “ports” is particularly important. The server-side V2Ray `1080` is a server listening port, while ports such as `7890` used by client software are usually local proxy ports. They are not the same thing. Once you understand the relationship between server ports, client ports, the AWS security group, and Ubuntu UFW, many seemingly complicated network problems become much easier to troubleshoot.

If you want to go further, you can build on this foundation by learning about domains, DNS, TLS, Nginx, Caddy, Docker, Cloudflare, and Linux server security. You will then have learned more than just “how to install V2Ray”; you will have a complete deployment approach that can be applied to personal websites, API services, Docker projects, and other cloud-server applications.

Finally, remember that the use of cloud servers and network services should comply with the laws and regulations of your jurisdiction, the AWS Terms of Service, and other applicable rules. This article is primarily intended for learning about AWS EC2, Ubuntu, Linux service management, and network configuration.
