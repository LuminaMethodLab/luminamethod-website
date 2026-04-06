# Lumina Method Lab — 网站部署指南

> 记录 luminamethod.eu 的完整基础设施搭建过程，供日后参考或重建使用。

---

## 整体架构

| 服务 | 平台 | 费用 |
|------|------|------|
| 域名 | Porkbun | ~€15/年 |
| DNS 管理 | Cloudflare | 免费 |
| 网站托管 | Cloudflare Workers | 免费 |
| 代码仓库 | GitHub (luminaMethodLab 组织) | 免费 |
| 收信转发 | Cloudflare Email Routing | 免费 |
| 发信 SMTP | Resend | 免费（3000封/月） |

---

## 第一步：将域名 DNS 转移到 Cloudflare

### 1.1 创建 Cloudflare 账号
- 前往 [cloudflare.com](https://cloudflare.com) 注册
- 可用 GitHub 账号直接登录

### 1.2 将域名接入 Cloudflare
1. 控制台首页点 **"Add a domain"**
2. 输入 `luminamethod.eu`，选 **Free** 套餐
3. Cloudflare 扫描现有 DNS 记录后，提供两个 Nameserver 地址，格式如：
   - `elle.ns.cloudflare.com`
   - `yisroel.ns.cloudflare.com`

### 1.3 在 Porkbun 更换 Nameserver
1. 登录 [porkbun.com](https://porkbun.com)
2. 找到 `luminamethod.eu` → **Details**
3. 在 **Authoritative Nameservers** 区域删除原有地址
4. 填入 Cloudflare 提供的两个地址
5. 保存

> **等待时间**：DNS 生效通常需要 5 分钟至 2 小时。可在 [dnschecker.org](https://dnschecker.org) 查询 NS 记录确认是否生效。

### 1.4 注意事项
- **DNSSEC**：确保 Porkbun 和 Cloudflare 两侧的 DNSSEC 均为**关闭**状态，否则会导致解析失败
- Porkbun 默认会留下 A 记录和 CNAME 记录（指向 pixie.porkbun.com），需在 Cloudflare DNS 中**全部删除**

---

## 第二步：清理 DNS 残留记录

DNS 转移后，Cloudflare 会扫描并保留 Porkbun 原有记录，需手动删除：

1. Cloudflare → `luminamethod.eu` → **DNS** → **Records**
2. 删除以下记录：
   - A 记录：`44.227.76.166`
   - A 记录：`44.227.65.245`
   - CNAME `*` → `pixie.porkbun.com`
   - CNAME `www` → `pixie.porkbun.com`

---

## 第三步：创建 GitHub 仓库

1. 在 [github.com/luminaMethodLab](https://github.com/luminaMethodLab) 组织下新建仓库
2. 仓库名：`website`，选 **Public**，勾选 **Add a README**
3. 上传网站文件，首页文件必须命名为 `index.html`

### 必要配置文件：`wrangler.json`

在仓库根目录创建此文件，内容如下：

```json
{
  "name": "lumina-website",
  "compatibility_date": "2026-04-05",
  "assets": {
    "directory": "./"
  }
}
```

> 此文件是 Cloudflare Workers 部署所必需的，缺少会导致部署报错。

---

## 第四步：在 Cloudflare 部署网站

1. Cloudflare 控制台 → **Workers & Pages** → **Create application**
2. 选 **Connect to Git** → 授权 GitHub → 选择 `luminaMethodLab/website` 仓库
3. 配置项：
   - **Project name**：`lumina-website`
   - **Build command**：留空
   - **Deploy command**：`npx wrangler deploy --assets .`
4. 点 **Deploy**

部署成功后会生成一个临时域名：`lumina-website.luminamethod-lab.workers.dev`

### 绑定自定义域名

1. 进入 `lumina-website` 项目 → **Domains & Routes**
2. 点 **Add** → 输入 `luminamethod.eu`
3. 确认添加

> **注意**：如提示域名已被占用，检查 DNS Records 中是否有冲突记录，删除后重新添加即可。

---

## 第五步：配置收信邮箱（Email Routing）

让 `contact@luminamethod.eu` 转发到 Gmail。

1. Cloudflare → `luminamethod.eu` → **Email** → **Email Routing**
2. 点 **Get started**
3. 填写：
   - Custom address：`contact`
   - Action：**Send to an email**
   - Destination：`luminamethod.lab@gmail.com`
4. 点击 Cloudflare 发送到 Gmail 的确认邮件中的链接激活

Cloudflare 会自动添加所需的 MX 和 TXT 记录：

| 类型 | 值 |
|------|-----|
| MX | route1/2/3.mx.cloudflare.net |
| TXT | `v=spf1 include:_spf.mx.cloudflare.net ~all` |

---

## 第六步：配置发信（Resend SMTP）

让 Gmail 能以 `contact@luminamethod.eu` 身份发信。

### 6.1 注册 Resend
- 前往 [resend.com](https://resend.com) 注册（仅需邮箱，无需手机号）
- 获取 API Key

### 6.2 在 Resend 添加域名
1. Resend 控制台 → **Domains** → **Add Domain** → 输入 `luminamethod.eu`
2. 手动添加以下 DNS 记录到 Cloudflare：

| 类型 | Name | 内容 |
|------|------|------|
| TXT | `resend._domainkey` | `p=MIGfMA0...`（Resend 提供） |
| TXT | `send` | `v=spf1 include:amazonses.com ~all` |
| MX | `send` | `feedback-smtp.eu-west-1.amazonses.com`（优先级 10） |
| TXT | `_dmarc` | `v=DMARC1; p=none;` |

3. 回到 Resend 点 **Verify**，等待各记录状态变为 Verified

### 6.3 在 Gmail 配置发件地址
1. Gmail → 设置 → **Accounts and Import** → **Send mail as** → **Add another email address**
2. 填写：
   - Name：`Lumina Method Lab`
   - Email：`contact@luminamethod.eu`
   - 勾选 **Treat as an alias**
3. SMTP 服务器配置：
   - **SMTP Server**：`smtp.resend.com`
   - **Port**：`465`
   - **Username**：`resend`
   - **Password**：Resend API Key
   - 选 **Secured connection using SSL**
4. 点击 Gmail 发送的验证邮件中的确认链接

---

## 日常更新网站

只需修改 GitHub 仓库中的文件，push 后 Cloudflare 自动重新部署，无需任何手动操作。

```
修改本地文件 → git push → Cloudflare 自动部署 → luminamethod.eu 更新
```

---

## 后续扩展计划

- **动态图表功能**：Streamlit Cloud 部署 Python 应用，挂载到子域名 `app.luminamethod.eu`
- **付费用户系统**：Supabase（免费层）做数据库 + 用户认证
- **www 子域名**：在 Cloudflare DNS 添加 CNAME `www` → `luminamethod.eu`
- **更多邮箱地址**：在 Cloudflare Email Routing 中添加新的转发规则（如 jessica@luminamethod.eu）

---

*最后更新：2026年4月*
