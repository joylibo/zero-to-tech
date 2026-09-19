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

前提：服务器上已装好 Python 3.10+、Node.js 18+ 和 Nginx，
且 Nginx 的站点根目录已指向本项目的 `out/`、监听 80 端口。

**1. 拉取代码**

```bash
cd ~/zero-to-tech
git pull
```

**2. 前端：装依赖、写配置、构建**

```bash
npm install
cp .env.example .env.production   # 按下面「配置说明」填好
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

**4. 后端：在后台跑起来**

```bash
nohup .venv/bin/fastapi run > backend.log 2>&1 &
```

`fastapi run` 是生产模式，监听 `0.0.0.0:8000`；`nohup ... &` 让它在
SSH 断开后继续运行，日志写进 `backend.log`。

查看日志、停止服务：

```bash
tail -f backend.log           # 看日志
ps aux | grep fastapi         # 找到进程号
kill 进程号                    # 停掉
```

**5. 放行 8000 端口**

去云平台控制台的安全组 / 防火墙，放行 8000 端口（80 端口应该已经放行）。

**6. 验证**

浏览器访问 `http://服务器IP`，打开文字实验室做一次分析，再看历史记录。
换一个浏览器（或无痕窗口）再试一次，两边的历史记录应该是互相看不到的。

## 配置说明

配置文件不进 Git，请照着 `.env.example` 自己建一份。

**前端**：开发用 `.env.local`，生产构建用 `.env.production`

| 键 | 说明 | 本地 | 线上 |
| --- | --- | --- | --- |
| `NEXT_PUBLIC_API_BASE_URL` | 后端接口地址 | `http://localhost:8000` | `http://服务器IP:8000` |

**后端**：`backend/.env`

| 键 | 说明 | 本地 | 线上 |
| --- | --- | --- | --- |
| `ALLOWED_ORIGINS` | 允许跨源访问的前端地址，多个用逗号隔开 | `http://localhost:3000` | `http://服务器IP`（不带端口） |
