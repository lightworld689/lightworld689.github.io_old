---
layout: default
title: 通过 Cloudflare Worker 免费发送电子邮件
description: 原文地址：https://devcxl.cn/blog/cloudflare-worker-send-email/
---

2022 年 5 月 13 号，Cloudflare 发布了一篇博客，宣布与电子邮件安全公司 Mailchannels 合作。这次合作中，MailChannels 专门为 Cloudflare Workers 创建了一项电子邮件发送服务，降低了 Cloudflare Workers 发送电子邮件的门槛。更值得高兴的是，发送电子邮件是完全免费的。通过该电子邮件发送服务，我们可以发送交易电子邮件，例如交易订单、用户注册确认、密码重置，也能够发送营销邮件。

## 前置条件

- 一个 Cloudflare 帐户
- 一个可以被 Cloudflare 管理的域名
- npm 软件包 create-cloudflare
- git
- openssl（可选）

## 为你的 Cloudflare 帐户启用 MailChannels

### 1. 找到您的帐户 workers.dev 子域

1. 登录 Cloudflare 仪表板并选择你的帐户。
2. 选择 **工作人员和页面 > 概述**。
3. 在概述的右侧，记下您的 workers.dev 子域，例如 myaccount.workers.dev。

### 2. 添加 MailChannels DNS 记录

1. 在 “帐户主页” 中，选择您要为其添加 SPF 记录的网站。
2. 选择 **DNS > 记录 > 添加记录**。
3. 添加以下 TXT DNS 记录，替换 myaccount.workers.dev 为你自己的 workers.dev 子域，替换 youdomain.com 为你自己的自定义域二级域名。

```
Type    Name            Content
TXT     _mailchannels   v=mc1 cfid=myaccount.workers.dev cfid=yourdomain.com
```

yourdomain.com 应设置为 worker 自定义域的二级域名。

### 3. 添加 MailChannel 的 SPF 支持

1. 在 “帐户主页” 中，选择您要为其添加 SPF 记录的网站。
2. 选择 **DNS > 记录 > 添加记录**。
3. 添加以下 TXT DNS 记录：

```
Type    Name    Content
TXT     @       v=spf1 include:_spf.mx.cloudflare.net include:relay.mailchannels.net -all
```

### 4. 添加 MailChannel 的 DKIM 支持

1. 使用 openssl 生成 DKIM 凭据（包含公钥、私钥）

```bash
openssl genrsa 2048 | tee private_key.pem | openssl rsa -outform der | openssl base64 -A > private_key.txt
```

```bash
echo -n "v=DKIM1;p=" > dkim_record.txt && openssl rsa -in private_key.pem -pubout -outform der | openssl base64 -A >> dkim_record.txt
```

2. 在 “帐户主页” 中，选择您要为其添加 DKIM 记录的网站。
3. 选择 **DNS > 记录 > 添加记录**。
4. 添加以下 TXT DNS 记录：

```
Type    Name                            Content
TXT     mailchannels._domainkey         dkim_record.txt中的内容
```

（注：._domainkey 前可以设置任何值，为了方便标识记忆，推荐使用 mailchannels._domainkey）

### 5. 修改 _dmarc DNS 记录

1. 在 “帐户主页” 中，选择您要为其添加 DMARC 记录的网站。
2. 选择 **DNS > 记录 > 添加记录**。
3. 添加以下 TXT DNS 记录：

```
Type    Name    Content
TXT     _dmarc  "v=DMARC1; p=reject; adkim=s; aspf=s; pct=100; fo=1;"
```

（详细参数解释略）

## 编写发送邮件的 Worker 代码

创建 `email.js` 文件，并编写如下代码：

```javascript
export async function send(to, title, content, type = "text/html") {
  const send_request = new Request("https://api.mailchannels.net/tx/v1/send", {
    method: "POST",
    headers: {
      "content-type": "application/json",
    },
    body: JSON.stringify({
      personalizations: [
        {
          to: [{ email: `${to}`, name: `${to}` }],
        },
      ],
      from: { email: "noreplay@example.com", name: "Sender Name" },
      subject: `${title}`,
      content: [
        {
          type: `${type}`,
          value: `${content}`,
        },
      ],
    }),
  });
  return await fetch(send_request);
}
```

注意：将 `noreplay@example.com` 替换为你自己想要设置的发信地址。将 `Sender Name` 设置为你想要设置的发信名。

## 部署 API 服务

1. 克隆该项目

```bash
git clone https://github.com/devcxl/cloudflare-email-sender
```

2. 修改 wrangeler 配置

将 `wrangler.example.toml` 重命名为 `wrangler.toml` 并修改其中的配置项：

- `SENDER_EMAIL`：发送邮件的邮箱
- `SENDER_NAME`：发信人名称
- `DKIM_DOMAIN`：DKIM 发信二级域名
- `DKIM_SELECTOR`：DKIM 选择器（设置 ._domainkey 前对应的值即可）

3. 部署

```bash
npm i
npm run deploy
```

根据提示登录你的 Cloudflare 账号并部署：

```bash
openssl rand -base64 32
npx wrangler secret put ACCESS_TOKEN
npx wrangler secret put DKIM_PRIVATE_KEY
```

## 使用示例

### 发送自定义邮件

```bash
curl -X POST -L https://custom.yourdomain.com/v1/send \
-H 'Content-Type: application/json' \
-H 'Authorization: Bearer {ACCESS_TOKEN}' \
-d '{
    "to": "you@example.com",
    "name": "Jone",
    "title": "Just Test Message",
    "content": "<h1>Hello This is test message</h1>",
    "type": "text/html"
}'
```

### 发送纯文本邮件

```bash
curl -X POST -L https://custom.yourdomain.com/v1/send \
-H 'Content-Type: application/json' \
-H 'Authorization: Bearer {ACCESS_TOKEN}' \
-d '{
    "to": "you@example.com",
    "name": "Jone",
    "title": "Just Test Message",
    "content": "Hello This is test message. ",
    "type": "text/plain"
}'
```

### 发送模板邮件

```bash
curl -X POST -L https://custom.yourdomain.com/v1/send/activation \
-H 'Content-Type: application/json' \
-H 'Authorization: Bearer {ACCESS_TOKEN}' \
-d '{
    "to": "you@example.com",
    "name": "Example",
    "title": "Just Test Message",
    "site_name": "Test Title",
    "url": "https://www.google.com/search?q=devcxl"
}'
```

## 参考资料

- [Send email from workers using MailChannels for free](https://community.cloudflare.com/t/send-email-from-workers-using-mailchannels-for-free/361973/63)
- [Cloudflare Worker send email](https://www.fadhil-blog.dev/blog/cloudflare-worker-send-email/)
- [How to send free transactional emails with worker MailChannels via Cloudflare Workers](https://medium.com/@tristantrommer/how-to-send-free-transactional-emails-with-worker-mailchannels-via-cloudflare-workers-818b787b33f9)
- [Send emails with Resend](https://developers.cloudflare.com/workers/tutorials/send-emails-with-resend/)
- [Email Flare: Send from Worker for Free](https://www.breakp.dev/blog/email-flare-send-from-worker-for-free/)
- [What is DMARC](https://blog.mailchannels.com/what-is-dmarc/)
- [Cloudflare learning DNS DKIM records](https://www.cloudflare.com/en-ca/learning/dns/dns-records/dns-dkim-record/)
- [DKIM setup](https://github.com/cloudsecurityalliance/webfinger.io/blob/main/docs.webfinger.io/DKIM-setup.md)

# 原文链接
- [ErᎥc的数字花园](https://devcxl.cn/blog/cloudflare-worker-send-email/)
