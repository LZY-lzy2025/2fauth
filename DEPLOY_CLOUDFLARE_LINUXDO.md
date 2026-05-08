# Cloudflare 网页端手动部署教程（Linux Do OAuth 登录）

本文档用于把本项目通过 **Cloudflare Dashboard 网页端**手动部署为 Cloudflare Worker，并使用 **Linux Do Connect / Linux Do 论坛 OAuth** 作为登录客户端。

> 适用场景：不使用 Wrangler CLI、不写 `wrangler.toml`，直接在 Cloudflare 网页后台复制 `_worker.js` 部署。

## 1. 部署前准备

你需要准备：

1. 一个 Cloudflare 账号。
2. 一个 Linux Do 账号。
3. 当前仓库的 `_worker.js` 文件内容。
4. 一个最终访问域名，可以先使用 Cloudflare 自动分配的 `*.workers.dev`，后续再绑定自定义域名。

Linux Do Connect 当前使用的 OAuth 端点如下：

| 用途 | 地址 |
| --- | --- |
| 授权端点 | `https://connect.linux.do/oauth2/authorize` |
| Token 端点 | `https://connect.linux.do/oauth2/token` |
| 用户信息端点 | `https://connect.linux.do/api/user` |

因此本项目部署时的 `OAUTH_BASE_URL` 应填写：

```text
https://connect.linux.do
```

## 2. 在 Cloudflare 创建 Worker

1. 登录 [Cloudflare Dashboard](https://dash.cloudflare.com/)。
2. 进入 **Workers & Pages**。
3. 点击 **Create** / **Create application**。
4. 选择 **Worker**。
5. 设置 Worker 名称，例如：

   ```text
   2fa-secure-manager
   ```

6. 创建完成后进入 Worker 的代码编辑页面。
7. 删除默认示例代码，把仓库中的 `_worker.js` 全部复制进去。
8. 点击 **Save and deploy**。

部署完成后，你会得到一个 Worker 访问地址，通常类似：

```text
https://2fa-secure-manager.<你的子域>.workers.dev
```

后面的 OAuth 回调地址会用到它。

## 3. 创建 KV 存储

本项目使用 Cloudflare KV 保存 2FA 账户数据、WebDAV 配置等信息。代码里固定通过 `env.USER_DATA` 访问 KV，所以绑定名必须是 `USER_DATA`。

操作步骤：

1. 在 Cloudflare Dashboard 进入 **Workers KV**。
2. 点击 **Create instance** / **Create namespace**。
3. 名称建议填写：

   ```text
   USER_DATA
   ```

4. 创建完成。

## 4. 给 Worker 绑定 KV

1. 回到刚才创建的 Worker。
2. 进入 **Settings**。
3. 找到 **Bindings**。
4. 点击 **Add**。
5. 选择 **KV Namespace**。
6. 按下面填写：

   | 字段 | 填写内容 |
   | --- | --- |
   | Variable name / Binding name | `USER_DATA` |
   | KV namespace | 选择刚才创建的 KV namespace |

7. 保存并重新部署。

> 注意：绑定名必须是大写的 `USER_DATA`，不要写成 `user_data`、`KV` 或其他名字。

## 5. 在 Linux Do Connect 申请 OAuth 应用

1. 打开：

   ```text
   https://connect.linux.do
   ```

2. 使用 Linux Do 账号登录。
3. 进入 **我的应用接入**。
4. 点击 **申请新接入**。
5. 填写应用信息。

关键字段如下：

| Linux Do Connect 字段 | 填写示例 |
| --- | --- |
| 应用名称 | `2FA Secure Manager` |
| 应用简介 | 简短描述你的 2FA 管理器用途 |
| 回调地址 | `https://你的-worker域名/api/oauth/callback` |

例如你的 Worker 地址是：

```text
https://2fa-secure-manager.example.workers.dev
```

那么 Linux Do Connect 的回调地址应填写：

```text
https://2fa-secure-manager.example.workers.dev/api/oauth/callback
```

申请成功后，你会获得：

- `Client Id`
- `Client Secret`

它们分别对应本项目的：

- `OAUTH_CLIENT_ID`
- `OAUTH_CLIENT_SECRET`

## 6. 获取自己的 Linux Do 用户 ID

本项目会使用 `OAUTH_ID` 限制只有指定 Linux Do 用户可以登录。这里必须填写 Linux Do 返回的用户唯一 `id`，不是用户名，也不是昵称。

获取方式：

1. 先完成部署和变量配置。
2. 如果不确定自己的 `id`，可以临时把 `OAUTH_ID` 填成一个明显不可能的值，例如：

   ```text
   OAUTH_ID=0
   ```

3. 登录时如果被拒绝，可到 Cloudflare Worker 日志中查看 OAuth 返回的 `userId`。
4. 把日志里的真实 `userId` 填回 `OAUTH_ID`。

如果你已经通过其他方式拿到了 Linux Do Connect 的用户信息接口返回值，则直接使用返回 JSON 里的 `id` 字段。例如：

```json
{
  "id": 12345,
  "username": "yourname",
  "name": "Your Nickname"
}
```

则应填写：

```text
OAUTH_ID=12345
```

## 7. 在 Cloudflare 配置环境变量和 Secrets

进入 Worker：

1. 打开 **Settings**。
2. 进入 **Variables and Secrets**。
3. 点击 **Add**。
4. 按下面表格逐个添加。
5. 添加完成后点击 **Deploy**。

### 7.1 必填变量

| 变量名 | 类型建议 | 示例 | 获取方式 |
| --- | --- | --- | --- |
| `OAUTH_BASE_URL` | Text | `https://connect.linux.do` | 固定填写 Linux Do Connect 基础地址 |
| `OAUTH_REDIRECT_URI` | Text | `https://你的-worker域名/api/oauth/callback` | 你的 Worker 地址加 `/api/oauth/callback` |
| `OAUTH_ID` | Text | `12345` | Linux Do 用户信息里的 `id` 字段 |
| `OAUTH_CLIENT_ID` | Secret | `你的 Client Id` | Linux Do Connect 申请应用后获得 |
| `OAUTH_CLIENT_SECRET` | Secret | `你的 Client Secret` | Linux Do Connect 申请应用后获得 |
| `JWT_SECRET` | Secret | 随机长字符串 | 自己生成 |
| `ENCRYPTION_KEY` | Secret | 随机长字符串 | 自己生成 |

### 7.2 可选变量

| 变量名 | 类型建议 | 示例 | 说明 |
| --- | --- | --- | --- |
| `ALLOWED_ORIGINS` | Text | `https://你的-worker域名` | 限制允许的跨域来源；不填时默认 `*` |

建议生产环境填写 `ALLOWED_ORIGINS`，值为你的实际访问地址。例如：

```text
ALLOWED_ORIGINS=https://2fa-secure-manager.example.workers.dev
```

如果后续绑定了自定义域名，例如：

```text
https://2fa.example.com
```

则改为：

```text
ALLOWED_ORIGINS=https://2fa.example.com
```

多个域名用英文逗号分隔：

```text
ALLOWED_ORIGINS=https://2fa.example.com,https://2fa-secure-manager.example.workers.dev
```

## 8. 生成 JWT_SECRET 和 ENCRYPTION_KEY

`JWT_SECRET` 用于登录会话签名，`ENCRYPTION_KEY` 用于加密保存在 KV 中的 2FA 数据。两者都应该是高强度随机字符串，并且不要使用同一个值。

如果你本地有 OpenSSL，可以分别执行：

```bash
openssl rand -base64 48
openssl rand -base64 48
```

把第一次输出填入：

```text
JWT_SECRET
```

把第二次输出填入：

```text
ENCRYPTION_KEY
```

> 重要：部署后不要随意更换 `ENCRYPTION_KEY`。旧的 2FA 数据是用该密钥加密的，更换后可能无法解密历史数据。

## 9. 推荐的完整变量示例

假设 Worker 地址为：

```text
https://2fa-secure-manager.example.workers.dev
```

则 Cloudflare 变量可以这样配置：

### Text 变量

```text
OAUTH_BASE_URL=https://connect.linux.do
OAUTH_REDIRECT_URI=https://2fa-secure-manager.example.workers.dev/api/oauth/callback
OAUTH_ID=12345
ALLOWED_ORIGINS=https://2fa-secure-manager.example.workers.dev
```

### Secret 变量

```text
OAUTH_CLIENT_ID=从 Linux Do Connect 获取的 Client Id
OAUTH_CLIENT_SECRET=从 Linux Do Connect 获取的 Client Secret
JWT_SECRET=openssl rand -base64 48 生成的随机值
ENCRYPTION_KEY=openssl rand -base64 48 生成的另一个随机值
```

## 10. 测试登录

1. 访问你的 Worker 首页：

   ```text
   https://你的-worker域名/
   ```

2. 点击页面上的 OAuth / 第三方授权登录按钮。
3. 页面应跳转到 Linux Do Connect 授权页面。
4. 同意授权后，浏览器应跳回：

   ```text
   https://你的-worker域名/api/oauth/callback
   ```

5. 如果 `OAUTH_ID` 匹配，系统会登录成功并返回首页。

## 11. 常见问题排查

### 11.1 点击登录后没有跳到 Linux Do

重点检查：

- `OAUTH_BASE_URL` 是否为 `https://connect.linux.do`。
- `OAUTH_CLIENT_ID` 是否填写正确。
- Worker 是否已经重新部署。

### 11.2 回调后提示授权失败

重点检查：

- Linux Do Connect 后台填写的回调地址是否和 `OAUTH_REDIRECT_URI` 完全一致。
- 是否多了或少了末尾 `/`。
- 是否使用了 `http://` 而不是 `https://`。
- `OAUTH_CLIENT_SECRET` 是否填写正确。

### 11.3 提示 Unauthorized user

说明 OAuth 已经成功，但当前 Linux Do 用户的 `id` 和 `OAUTH_ID` 不一致。

处理方式：

1. 查看 Cloudflare Worker 日志中的 `userId`。
2. 把 `OAUTH_ID` 改成该用户 ID。
3. 重新部署。

### 11.4 保存 2FA 数据失败

重点检查：

- Worker 是否绑定了 KV namespace。
- KV binding name 是否为 `USER_DATA`。
- `ENCRYPTION_KEY` 和 `JWT_SECRET` 是否已配置。

### 11.5 更换域名后无法登录

如果你从 `workers.dev` 换成了自定义域名，需要同步修改三个地方：

1. Linux Do Connect 后台的回调地址。
2. Cloudflare Worker 变量 `OAUTH_REDIRECT_URI`。
3. Cloudflare Worker 变量 `ALLOWED_ORIGINS`。

三者都改完后重新部署。

## 12. 安全建议

1. `OAUTH_CLIENT_SECRET`、`JWT_SECRET`、`ENCRYPTION_KEY` 必须使用 Cloudflare 的 Secret 类型保存。
2. 不要把 `Client Secret` 发给别人，也不要写进公开仓库。
3. 不要频繁更换 `ENCRYPTION_KEY`，否则历史加密数据可能无法读取。
4. 生产环境建议绑定自定义域名，并把 `ALLOWED_ORIGINS` 限制为该域名。
5. 定期通过 WebDAV 或导出功能备份你的 2FA 数据。
