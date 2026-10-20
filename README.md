# 文字实验室

一个中文文本分析小工具：输入一段话，给出情感倾向评分和全文拼音，
并把每次分析的结果存下来，各人只看得到自己的那一份。

零到全栈课程的贯穿项目。

## 技术栈

- 前端：Next.js（静态导出）＋ React
- 后端：FastAPI ＋ uvicorn
- 分析：snownlp（情感）、pypinyin（注音）
- 存储：SQLite
- 线上：Nginx

## 本地跑起来

需要：Node.js 18+、Python 3.10+

**后端**

```bash
cd backend
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env          # 按下面「配置说明」填好
fastapi dev                   # → http://localhost:8000
```

**前端**（另开一个终端）

```bash
npm install
cp .env.example .env.local    # 按下面「配置说明」填好
npm run dev                   # → http://localhost:3000
```

## 部署到服务器

前提：服务器上已装好 Python 3.10+、Node.js 18+ 和 Nginx。

**1. 拉取代码**

```bash
cd ~/zero-to-tech
git pull
```

**2. 前端：装依赖、写配置、构建**

```bash
npm install
cp .env.example .env.production   # 按下面「配置说明」填好（线上留空）
npm run build                     # 产物进 out/，由 Nginx 提供服务
```

**3. 后端：建环境、装依赖、写配置**

```bash
cd backend
python3 -m venv --prompt=zero-to-tech .venv   # 首次部署才需要
source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env              # 按下面「配置说明」填好
```

**4. 后端：委托给 systemd**（首次部署才需要）

新建 `/etc/systemd/system/zero-backend.service`：

```ini
[Unit]
Description=zero-to-tech backend

[Service]
User=ubuntu
WorkingDirectory=/home/ubuntu/zero-to-tech/backend
ExecStart=/home/ubuntu/zero-to-tech/backend/.venv/bin/fastapi run --host 127.0.0.1
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl start zero-backend     # 启动
sudo systemctl enable zero-backend    # 开机自启
sudo systemctl status zero-backend    # 看到 active (running) 就对了
```

**5. Nginx：静态文件 ＋ `/api/` 反向代理**（首次部署才需要）

编辑 `/etc/nginx/sites-enabled/default`：

```nginx
server {
    listen 80 default_server;
    server_name _;

    root /home/ubuntu/zero-to-tech/out;
    index index.html;

    location / {
        try_files $uri $uri.html $uri/ =404;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8000;     # 末尾不要加斜杠

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

```bash
sudo nginx -t && sudo systemctl reload nginx
```

**6. 域名与 HTTPS**（首次部署才需要）

1. 在域名控制台加两条 A 记录：主机记录分别填 `@` 和 `www`，记录值都填服务器 IP（服务器在中国大陆的，要先完成 ICP 备案）
2. 给网站单独建一份以域名命名的配置，`default` 只留作兜底：

   ```bash
   sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/example.com
   sudo vim /etc/nginx/sites-available/example.com
   sudo vim /etc/nginx/sites-available/default
   sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
   sudo nginx -t && sudo systemctl reload nginx
   ```

   - `example.com`：两行 `listen` 去掉 `default_server`，`server_name _;` 改成 `server_name example.com www.example.com;`，再加一行 `access_log /var/log/nginx/example.com.access.log;`
   - `default`：整个换成兜底块——`listen 80 default_server;`、`listen [::]:80 default_server;`、`server_name _;`、`return 404;`

3. 安装 certbot，签发证书（两个域名都要选上，直接回车即全选）：

   ```bash
   sudo snap install --classic certbot
   sudo ln -s /snap/bin/certbot /usr/bin/certbot
   sudo certbot --nginx
   sudo certbot renew --dry-run      # 彩排一次续期，跑通就不用管了
   ```


**7. 安全组 / 防火墙**

只放行 **22（SSH）**、**80（HTTP）** 和 **443（HTTPS）**。**8000 不要放行**——后端只监听 `127.0.0.1`，由 Nginx 转发进去。


**8. 验证**

浏览器访问 `https://example.com`，地址栏不再标"不安全"；再敲 `http://` 开头的，应该自动跳到 `https://`。做一次分析、看一眼历史记录。


## 日常发布

```bash
cd ~/zero-to-tech
git pull

npm run build                          # 前端改了才需要
sudo systemctl restart zero-backend    # 后端改了才需要
```

## 常用运维命令

```bash
sudo systemctl status zero-backend     # 看状态
sudo systemctl restart zero-backend    # 重启
sudo systemctl stop zero-backend       # 停掉
journalctl -u zero-backend -f          # 实时看日志（Ctrl+C 退出）
journalctl -u zero-backend -n 50       # 看最后 50 行

sudo nginx -t                          # 改完 Nginx 配置先验语法
sudo systemctl reload nginx            # 再让它重读
```


## 配置说明

配置文件不进 Git，请照着 `.env.example` 自己建一份。

**前端**：开发用 `.env.local`，生产构建用 `.env.production`

| 键 | 说明 | 本地 | 线上 |
| --- | --- | --- | --- |
| `NEXT_PUBLIC_API_BASE_URL` | 后端接口地址 | `http://localhost:8000` | **留空**——前后端同源，代码里走相对路径 |


**后端**：`backend/.env`

| 键 | 说明 | 本地 | 线上 |
| --- | --- | --- | --- |
| `ALLOWED_ORIGINS` | 允许跨源访问的前端地址，多个用逗号隔开 | `http://localhost:3000` | `http://服务器IP`（不带端口） |
