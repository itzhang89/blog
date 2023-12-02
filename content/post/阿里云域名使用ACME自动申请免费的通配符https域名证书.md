---
categories:
    - devops
date: "2023-12-02T10:18:34+08:00"
description: ""
image: ""
slug: a-li-yun-yu-ming-shi-yong-acmezi-dong-shen-qing-mian-fei-de-tong-pei-fu-httpsyu-ming-zheng-shu
tags:
    - nginx
title: 阿里云域名使用ACME自动申请免费的通配符https域名证书
---
# 使用ACME自动申请免费的通配符https域名证书

以阿里云域名为例，比如 example.com。 我这里是 debian 12 的服务器为例。

## 安装ACME客户端

1. 直接在线安装（能够正常访问GitHub）

   ```bash
   curl https://get.acme.sh | sh
   ```

2. 国内用户可以参考如下 [acme.sh: An ACME Shell script, a certbot client: acme.sh](https://gitee.com/neilpang/acme.sh)

## 阿里云中创建子账号

从 [RAM 访问控制](https://ram.console.aliyun.com/users)，中新建一个新用户，授权 【OpenAPI 调用访问】。创建好新用户之后，添加 [AliyunHTTPDNSFullAccess](https://ram.console.aliyun.com/policies/AliyunHTTPDNSFullAccess/System/content) 和 [ AliyunDNSFullAccess](https://ram.console.aliyun.com/policies/AliyunDNSFullAccess/System/content)  2个权限。

保存新建用户的 AccessKey ID 和 AccessKey Secret 信息，具体如下

```
用户登录名称 xxxxxDNS@xxxxxxxx.onaliyun.com
AccessKey ID  xxxxxx
AccessKey Secret xxxxxxx
```

## 在安装了 acme.sh 中的主机中配置如下的环境变量

将获取到的AccessKey 和 Secret 写到acme.sh.env配置文件里面

```bash
root@xxx:~# cat /root/.acme.sh/acme.sh.env
export LE_WORKING_DIR="/root/.acme.sh"
alias acme.sh="/root/.acme.sh/acme.sh"
export Ali_Key="xxxx"
export Ali_Secret="xxxxx"
```

然后在 ~/.bashrc 中添加 `[ -f /root/.acme.sh/acme.sh.env ] && . /root/.acme.sh/acme.sh.env` 命令，最后运行 `source ~/.bashrc` 使得当前配置生效。

## 生成证书

将切换CA机构到letsencrypt:

```
acme.sh --set-default-ca --server letsencrypt
```

执行命令

```
acme.sh —issue —dns dns_ali -d .xxx.com -d '.xxx.com'
```

顺利的在终端可以看到类似的输出信息

```
-----END CERTIFICATE-----
[Fri Dec  1 21:42:29 CST 2023] Your cert is in: /root/.acme.sh/example.com_ecc/example.com.cer
[Fri Dec  1 21:42:29 CST 2023] Your cert key is in: /root/.acme.sh/example.com_ecc/example.com.key
[Fri Dec  1 21:42:29 CST 2023] The intermediate CA cert is in: /root/.acme.sh/example.com_ecc/ca.cer
[Fri Dec  1 21:42:29 CST 2023] And the full chain certs is there: /root/.acme.sh/example.com_ecc/fullchain.cer
```

## 自动配置到nginx

我这里需要让生产的证书提供给这台主机的上的nginx服务使用

```bash
acme.sh --install-cert -d example.com \
--key-file       /etc/nginx/cert/example.com.key  \
--fullchain-file /etc/nginx/cert/example.com.pem \
--reloadcmd     "service nginx force-reload"
```



## 设置定时检查

Let's Encrypt 的证书有效期是 90 天的，你需要定期 `renew` 重新申请，这部分 `acme.sh` 以及帮你做了，在安装的时候往 crontab 增加了一行每天执行的命令 `acme.sh --cron`:

```
$ crontab -l
0 0 * * * "/root/.acme.sh/acme.sh" --cron --home "/root/.acme.sh" > /dev/null
```

这样就是正常的：

```
[Fri Dec 23 11:50:30 CST 2016] Renew: 'www.your-app.com'
[Fri Dec 23 11:50:30 CST 2016] Skip, Next renewal time is: Tue Feb 21 03:20:54 UTC 2017
[Fri Dec 23 11:50:30 CST 2016] Add '--force' to force to renew.
[Fri Dec 23 11:50:30 CST 2016] Skipped www.your-app.com
```

`acme.sh --cron` 命令执行以后将会 **申请新的证书** 并放到相同的文件路径。由于前面执行 `--installcert` 的时候告知了重新 Nginx 的方法，`acme.sh` 也同时会在证书更新以后重启 Nginx。

最后走一下 acme.sh --cron 的流程看看能否正确执行

```
acme.sh --cron -f
```

这个过程应该会得到这样的结果，并在最后重启 Nginx (不需要输入密码)

```
[Tue Dec 27 14:28:09 CST 2016] Renew: 'www.your-app.com'
[Tue Dec 27 14:28:09 CST 2016] Single domain='www.your-app.com'
[Tue Dec 27 14:28:09 CST 2016] Getting domain auth token for each domain
[Tue Dec 27 14:28:09 CST 2016] Getting webroot for domain='www.your-app.com'
[Tue Dec 27 14:28:09 CST 2016] _w='/home/ubuntu/www/your-app/current/public/'
[Tue Dec 27 14:28:09 CST 2016] Getting new-authz for domain='www.your-app.com'
[Tue Dec 27 14:28:16 CST 2016] The new-authz request is ok.
[Tue Dec 27 14:28:16 CST 2016] www.your-app.com is already verified, skip.
[Tue Dec 27 14:28:16 CST 2016] www.your-app.com is already verified, skip http-01.
[Tue Dec 27 14:28:16 CST 2016] www.your-app.com is already verified, skip http-01.
[Tue Dec 27 14:28:16 CST 2016] Verify finished, start to sign.
[Tue Dec 27 14:28:19 CST 2016] Cert success.
... 省略
[Fri Dec 23 11:09:02 CST 2016] Your cert is in  /home/ubuntu/.acme.sh/www.your-app.com/www.your-app.com.cer 
[Fri Dec 23 11:09:02 CST 2016] Your cert key is in  /home/ubuntu/.acme.sh/www.your-app.com/www.your-app.com.key 
[Fri Dec 23 11:09:04 CST 2016] The intermediate CA cert is in  /home/ubuntu/.acme.sh/www.your-app.com/ca.cer 
[Fri Dec 23 11:09:04 CST 2016] And the full chain certs is there:  /home/ubuntu/.acme.sh/www.your-app.com/fullchain.cer 
[Tue Dec 27 14:28:22 CST 2016] Run Le_ReloadCmd: sudo service nginx force-reload
 * Reloading nginx nginx                                                                                                                                     [ OK ] 
[Tue Dec 27 14:28:22 CST 2016] Reload success
```



## 验证 SSL

访问 ssllabs.com 输入你的域名，检查 SSL 的配置是否都正常：

[https://ssllabs.com/ssltest/analyze.html?d=example.com](https://ssllabs.com/ssltest/analyze.html)

确保验证结果有 A 以上，否则根据提示调整问题



## 设置提醒

这里以 email 通知为例，添加如下内容到 `/root/.acme.sh/acme.sh.env` 文件中，同时使他生效

```
export MAIL_FROM="xx@example.com"
export MAIL_TO="youremail@xx.com"
```

生效之后，运行如下命令配置

```
acme.sh --set-notify  --notify-hook mail
```

更过通知方式参考[notify · acmesh-official/acme.sh Wiki](https://github.com/acmesh-official/acme.sh/wiki/notify)。



参考文章

---

1. [使用 acme.sh 给 Nginx 安装 Let’ s Encrypt 提供的免费 SSL 证书 · Ruby China](https://ruby-china.org/topics/31983)
1. [HTTPS quick-start — Caddy Documentation](https://caddyserver.com/docs/quick-starts/https)