# 折纸起始页 · 部署文档

> 纯静态站点：`index.html`（起始页）+ `translate.html`（翻译页）+ `wallpapaer/`（壁纸）+ `lucide.min.js`（图标库）+ `favicon.svg`（站点图标）。
> 无后端、无数据库、无构建，单文件即可运行。
> 部署目标：阿里云 ECS `47.99.147.163`，独立目录 `/opt/zz-start`，经 `https://dreamer.asia/zz/` 访问。

---

## 一、服务器信息

| 项 | 值 |
|------|-----|
| 公网 IP | 47.99.147.163 |
| 系统 | Ubuntu 22.04 LTS (x86_64) |
| 登录 | `ssh root@47.99.147.163`（密码登录） |
| 域名 | `dreamer.asia`（nginx 反代 + Let's Encrypt HTTPS） |
| 部署目录 | `/opt/zz-start/` |
| 访问地址 | `https://dreamer.asia/zz/` |

> ⚠️ **与之前项目隔离**：服务器上另有微信小程序后端 `/opt/uni-preset-server`（Express+Prisma+SQLite）和 H5 `/opt/uni-preset-h5`，以及它们共用的 Nginx 站点 `/etc/nginx/sites-enabled/suiyu`。本项目的部署**只新增 `/opt/zz-start` 目录和 Nginx 的两个 `location`（`/zz/` + `/zz/api/tmt`）**，绝不改动上述既有内容。

---

## 二、部署目录结构

本地项目根目录：

```
D:\code\vibecoding\折纸起始页\
├─ index.html        # 起始页
├─ translate.html    # 翻译页（Google 免费 / AI 大模型 / 腾讯 TMT）
├─ lucide.min.js     # Lucide 图标库（自托管）
├─ favicon.svg       # 站点图标
├─ wallpapaer\       # 壁纸（全量 + 缩略图）
├─ README.md         # 架构说明
├─ DEPLOY.md         # 本文档
└─ _deploy\
    ├─ site\         # 部署暂存目录（deploy_zz.py 整体打包）
    └─ deploy_zz.py  # 一键部署脚本（paramiko）
```

上传到服务器后的 `/opt/zz-start/`：

```
/opt/zz-start/
├─ index.html
├─ translate.html
├─ lucide.min.js     # 自托管图标库，页面本地引用
├─ favicon.svg       # 站点图标（彩纸千纸鹤）
└─ wallpapaer/
   ├─ bg-zp5z2w.jpg        # 花坡
   ├─ tb-zp5z2w.jpg
   ├─ wp-2em38y.jpg        # 夜鹿
   ├─ tb-wp-2em38y.jpg
   ├─ wp-e7kpl8.jpg        # 落日
   ├─ tb-wp-e7kpl8.jpg
   ├─ wp-e89l8k.jpg        # 晴空
   ├─ tb-wp-e89l8k.jpg
   ├─ wp-zm9zlv.jpg        # 波点
   └─ tb-wp-zm9zlv.jpg
```

> 只上传站点引用到的 5 张全量壁纸 + 5 张对应缩略图，原始的 `wallhaven-*.png`（约 6K×3K，体积大）不上传。

---

## 三、Nginx 配置

在既有站点 `suiyu` 内新增了两个 `location`（`/zz/` 静态 + `/zz/api/tmt` 腾讯代理），不干扰其 `root /opt/uni-preset-h5`、`/api/`、`/api/watch/ws`。

```nginx
server {
    server_name dreamer.asia www.dreamer.asia;
    root /opt/uni-preset-h5;      # 既有 H5，保留
    index index.html;

    location / {                  # 既有 H5，保留
        try_files $uri $uri/ /index.html;
    }

    # —— 新增：折纸起始页（独立目录，互不干扰）——
    location /zz/ {
        alias /opt/zz-start/;
        index index.html;
        try_files $uri $uri/ /zz/index.html;
    }

    # —— 新增：腾讯 TMT 反代（绕过浏览器 CORS，透传签名头）——
    location /zz/api/tmt {
        proxy_pass https://tmt.tencentcloudapi.com/;
        proxy_pass_request_headers on;
        proxy_set_header Host tmt.tencentcloudapi.com;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Connection "";
        proxy_http_version 1.1;
        proxy_ssl_server_name on;
        proxy_ssl_name tmt.tencentcloudapi.com;
    }

    location /api/ { ... }        # 既有 API 反代，保留
    location /api/watch/ws { ... }# 既有 WebSocket，保留

    listen 443 ssl;
    ssl_certificate ......;       # 复用 dreamer.asia 的 Let's Encrypt 证书
}
```

* 证书复用 `dreamer.asia` 现有的 Let's Encrypt，无需为新路径单独申证书。
* 该配置在写入前已备份为 `/etc/nginx/sites-enabled/suiyu.bak-zz`（先）与 `suiyu.bak-zz2`（加代理时）；每次 `nginx -t` 校验通过后才 `systemctl reload nginx`。

---

## 四、访问地址

| 页面 | 地址 |
|------|------|
| 起始页 | `https://dreamer.asia/zz/` |
| 翻译页 | `https://dreamer.asia/zz/translate.html` |
| 腾讯代理 | `https://dreamer.asia/zz/api/tmt` |
| 图标库 | `https://dreamer.asia/zz/lucide.min.js` |
| 站点图标 | `https://dreamer.asia/zz/favicon.svg` |

起始页底部导航的「翻译」项使用相对路径 `translate.html`，在 `/zz/` 下会正确解析到上面的翻译页，无需单独配置。

---

## 五、部署/更新（一键脚本：paramiko）

本地已备好 `_deploy/deploy_zz.py`，自动完成「本地打包 → 上传 → 服务器解压到 `/opt/zz-start`」。

### 1) 更新要部署的文件

本地项目根目录：
- 修改 `index.html` / `translate.html`，或替换 `wallpapaer/` 里的壁纸。

### 2) 刷新暂存目录

`_deploy/site/` 里的文件需与最新一致。拷贝下列文件到 `_deploy/site/`：
- `index.html`、`translate.html`、`lucide.min.js`、`favicon.svg`
- `wallpapaer/` 里 5 张全量 + 5 张缩略图 → `_deploy/site/wallpapaer/`

### 3) 一键部署

```powershell
python D:\code\vibecoding\折纸起始页\_deploy\deploy_zz.py <服务器密码>
```

脚本会：
1. 把 `_deploy/site` 打成本地 `tar.gz`。
2. 上传到服务器 `/tmp/zz-start.tar.gz`。
3. 服务器上把现有 `/opt/zz-start` 备份为 `/opt/zz-start.backup`，再解压新的到 `/opt/zz-start`。
4. 输出 `DEPLOY_OK` 表示成功。

> 密码作为命令行参数会短暂出现在进程列表，注意本机安全；也可在脚本运行后立即删除该历史记录。

### 4) 验证

```bash
curl -s -o /dev/null -w 'HTTP %{http_code}\n' https://dreamer.asia/zz/index.html
curl -s -o /dev/null -w 'HTTP %{http_code}\n' https://dreamer.asia/zz/lucide.min.js
curl -s -o /dev/null -w 'HTTP %{http_code}\n' https://dreamer.asia/zz/favicon.svg
# 均期望 HTTP 200
```

---

## 六、回滚（出问题恢复）

### 回滚站点文件

服务器上每次部署前都会备份旧目录为 `/opt/zz-start.backup`：

```bash
ssh root@47.99.147.163
rm -rf /opt/zz-start && cp -r /opt/zz-start.backup /opt/zz-start
```

### 回滚 Nginx 配置

```bash
cp /etc/nginx/sites-enabled/suiyu.bak-zz /etc/nginx/sites-enabled/suiyu
nginx -t && systemctl reload nginx
```

---

## 七、常用服务器命令

```bash
ssh root@47.99.147.163
ls -R /opt/zz-start                # 查看已部署文件
cat /etc/nginx/sites-enabled/suiyu # 查看站点配置（含 /zz/ 与 /zz/api/tmt）
nginx -t                           # 校验配置语法
systemctl reload nginx             # 重载（不中断服务）
rm -rf /opt/zz-start.backup        # 清理旧备份（确认新版本正常后）
```

---

## 八、引擎说明（translate.html）

三种翻译引擎，可随时在翻译页右上角齿轮切换：

| 引擎 | 浏览器 `https://dreamer.asia/zz/` | 说明 |
|------|:---:|------|
| **Google 免费** | ✅ 可用 | 默认，无需配置，自带 CORS |
| **AI 大模型** | ✅ 可用 | 自配接口地址/Key/模型，本地保存在浏览器 localStorage |
| **腾讯 TMT** | ✅ 可用 | 经 `/zz/api/tmt` 反代（Nginx 透传签名头），浏览器不再跨域 |

> 腾讯 TMT 的请求由页面自动判断：在 `dreamer.asia` 域下运行时请求打到 `https://dreamer.asia/zz/api/tmt`（走 Nginx 反代），`file://` 下则回退直连腾讯（仍会被 CORS 拦）。SecretKey 只留在浏览器（只转发签名好的 `Authorization` 头），服务器/代理看不到明文密钥。
>
> 腾讯 TMT 采用 **TC3-HMAC-SHA256 签名**，由页面内纯 JS 实现（`sha256Bytes`/`hmacSha256Bytes` 字节链），已用官方测试向量验证。需在腾讯云开通「机器翻译 TMT」并配置 SecretId/SecretKey/地域（默认 `ap-guangzhou`）。

---

## 九、本地浏览器与 HTTPS 的区别

* 部署后走 HTTPS，`localStorage` 域名是 `dreamer.asia`。你在本地 `file://` 双击打开时存的主题/壁纸/搜索/导航偏好，与服务器版是**两套数据**，互不影响。
* 服务器版首次打开会初始化默认值；如需沿用某套偏好，在对应端重新设置即可。

---

## 十、安全提示

* 部署脚本 `deploy_zz.py` 接收明文密码，属本地运维用；切勿上传或提交到公开仓库。
* 服务器 `/opt/uni-preset-server` 是你微信小程序的生产环境，**严禁**在本项目部署时覆盖其 `.env`、`prisma/dev.db`（已通过独立目录规避）。
* 若担心密码出现在 PowerShell 历史，可在执行前临时设置环境变量或改用交互输入。
