# 部署指南（Docker + 云服务器）

## 1. 服务器要求

- Linux（Ubuntu 22.04 / Debian 12 均可），1 核 2G 内存起步
- 已安装 Docker 24+ 与 Docker Compose 插件
- 放行 80 / 443（走反向代理）或直接放行 3000

```bash
curl -fsSL https://get.docker.com | sh
docker compose version
```

## 2. 上传项目并配置

把项目目录传到服务器（`git clone` 或 `scp -r`），然后：

```bash
cd MBoard
cp .env.example .env

# 生成随机密钥，替换 .env 里的 AUTH_SECRET
openssl rand -hex 32
```

编辑 `.env`：

```ini
AUTH_SECRET="刚刚生成的 64 位随机串"
SESSION_DAYS="7"
COOKIE_SECURE="false"   # 配好 HTTPS 后改成 true
```

编辑 `config/accounts.yml`，把演示账号换成自己的账号。推荐存 bcrypt 哈希：

```bash
# 容器启动后任意时刻都可以生成
docker compose exec mboard node scripts/hash-password.mjs "你的新密码"
```

把输出的 `passwordHash` 贴进配置文件，然后 `docker compose restart mboard`。

默认账号为 maxy / tannami / fashi / tomato，密码都是 `123456`。用户登录后可以在「账号设置」里修改显示名和密码，修改结果保存在数据库里并覆盖配置文件的初始值。

## 3. 启动

```bash
docker compose up -d --build
docker compose logs -f mboard
```

首次启动会自动：

1. 创建 `/app/data` 与上传目录
2. `prisma db push` 建表
3. 幂等写入演示看板、画板、文档
4. 启动 Next + WebSocket 服务

浏览器访问 `http://服务器IP:3000`，使用配置里的账号登录。

健康检查：

```bash
curl -I http://127.0.0.1:3000/login
# HTTP/1.1 200 OK
```

## 4. Nginx 反向代理（推荐）

```nginx
server {
    listen 80;
    server_name mboard.example.com;

    client_max_body_size 60m;   # 上传上限略大于 50MB

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;

        # 协作 WebSocket 必须保留 Upgrade
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 3600s;   # 保持长连接
        proxy_send_timeout 3600s;
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

### 配置 HTTPS

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d mboard.example.com
```

证书生效后，把 `.env` 里的 `COOKIE_SECURE` 改成 `true`，然后：

```bash
docker compose up -d
```

## 5. 数据备份

所有业务数据都在 `data/`（SQLite 数据库 + 上传文件）与 `config/`（账号）。

```bash
# 简单备份（短暂停服，保证数据库一致）
docker compose stop mboard
tar -czf mboard-$(date +%F).tar.gz data config
docker compose start mboard
```

如果宿主机安装了 `sqlite3`，也可以在线热备份数据库：

```bash
docker compose exec mboard sh -c 'sqlite3 /app/data/mboard.db ".backup /app/data/backup-$(date +%F).db"' || true
```

恢复：解压覆盖 `data/` 与 `config/`，再 `docker compose up -d`。

## 6. 升级

```bash
git pull                # 或重新上传代码
docker compose up -d --build
```

- 容器启动时自动执行 `prisma db push`，兼容新增字段
- 数据库与上传文件在挂载卷里，重新构建不会丢
- 账号配置改完只需 `docker compose restart mboard`

## 7. 常见问题

| 现象 | 排查 |
| --- | --- |
| 登录一直 401 | `.env` 的 `AUTH_SECRET` 是否被改动；`config/accounts.yml` 格式是否正确（YAML 缩进） |
| 页面能开但协作光标不出现 | 反向代理是否转发 `Upgrade` / `Connection` 头；`proxy_read_timeout` 是否过短 |
| 浏览器控制台 WebSocket 401 | Cookie 未带上：检查是否同域访问、`COOKIE_SECURE=true` 时是否已启用 HTTPS |
| 上传大文件失败 | Nginx `client_max_body_size` 调大；应用侧单文件上限 50MB |
| 时区不对 | `docker-compose.yml` 已设置 `TZ=Asia/Shanghai`，检查宿主机时间 |
| 容器反复重启 | `docker compose logs --tail=200 mboard`，多半是 `data/` 权限或账号配置报错 |

## 8. 安全建议

- 上线前必须修改 `AUTH_SECRET` 与所有默认账号密码，密码使用 bcrypt 哈希
- 只暴露 80/443，3000 端口不直接对公网开放
- 用 HTTPS，并把 `COOKIE_SECURE` 设为 `true`
- 定期备份 `data/`，并验证恢复流程
- 本项目的权限模型只有「登录 / 未登录」，不要把敏感数据放在共享工作台
