---
layout: post
title: "ssh证书登录"
date: 2016-09-12
categories: config
---

* 证书登录

1. 客户端生成证书: 私钥与公钥
2. 服务器添加公钥

* 客户端生成私钥与公钥

`ssh-keygen -t rsa`

rsa是一种密码的算法，还有dsa可以选择。rsa是登录常用的

密码对生成后会在用户目录下的`.ssh/`目录下，

- `id\_rsa`： 私钥
- `id_rsa.pub`: 公钥

* ssh服务端配置

ssh配置如下：
```bash
vim /etc/ssh/sshd_config

# 禁用root账号登录，非必要。但为了安全，请禁用此项
PermitRootLogin no

# 是否让sshd去检查用户home目录或者相关档案的权限数据
# 这是为了担心使用者将某些重要档案的权限设错，可能会导致一些问题
# 例如使用者的~/.ssh/权限设错时，某些特殊情况下会不许用户登录
StrictModes no

# 是否允许用户自行使用成对的密钥系统进行登录行为。仅针对Version 2.
# 至于自制的公钥数据就放置与用户目录下的 .ssh/authorized\_keys内
RSAAuthentication yes
PublicAuthentication yes
AuthorizedKeysFile %h/.ssh/authorized_keys

# 有了证书登录，禁用密码登录
PasswordAuthentication no
```

ssh服务器配置完成后，将客户端上生成的公钥上传到服务器端的 authorized_keys

在客户端中执行:

```bash
scp ~/.ssh/id_rsa.pub <ssh_username>@<ssh_server_ip>:<ssh_server_port>:~/.ssh/authorized_keys
```

最后需要重启ssh服务器

```bash
/etc/init.d/ssh restart
```

* 客户端使用私钥登录ssh服务器

ssh命令

```bash
ssh -i ~/.ssh/id_rsa <ssh_username>@<ssh_server_ip>:<ssh_server_port>
```

scp命令

```bash
scp -i ~/.ssh/id_rsa <to-local-file-name> <ssh_username>:<ssh_server_ip>:<ssh_server_port>:<to-remote-file-name>
```

每次敲命令，都需要指定私钥。可以把私钥的路径加入到ssh客户端的默认配置里。
修改/etc/ssh/ssh\_config
```config
# 默认情况下已经加入私钥路径了。
IdentityFile ~/.ssh/id_rsa

# 如果有其它路径，还要以加入
Identityfile <path-to-other>
```
