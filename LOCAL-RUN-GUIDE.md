# TrendRadar 本地部署与使用备忘

> 目标：在自己的机器或 NAS 上运行 TrendRadar，按需推送热点或提供 MCP AI 分析服务。本文总结本地运行要点，并说明是否必须使用 Docker。

## 运行方式速览
- **GitHub Actions 托管**：Fork 项目后依赖 GitHub 定时任务，适合完全“无服务器”场景；本地无需部署。
- **原生 Python 环境**：直接在 Windows/macOS/Linux 上运行 `python main.py`。配置灵活，适合希望手动掌控调度/调试的用户；**不需要 Docker**。
- **Docker / Docker Compose**：将代码打包成容器，适合 NAS、服务器或想快速复制环境的用户。
- **MCP AI 服务（可选）**：通过 `uv run python -m mcp_server.server` 暴露 HTTP/STDIO 接口给 Claude、Cursor 等客户端，需先保证爬虫已产生数据。

## 公共准备
1. **拉取代码**：`git clone https://github.com/sansan0/TrendRadar.git` 或直接下载压缩包。
2. **复制配置**：
   - `config/config.yaml`：主配置（运行模式、推送渠道、推送时间窗口等）。
   - `config/frequency_words.txt`：关键词/词组；空文件代表推送所有热点。
   - 建议先从仓库原始模板复制，再按需求修改。
3. **填写通知渠道**：优先使用环境变量/Secrets；若确认只在本地使用，可直接写入 `config.yaml` 下的 `webhooks`。
4. **确认网络**：默认访问 [newsnow](https://newsnow.busiyi.world/) API，请确保网络可达。

## 方式一：原生 Python 环境（无需 Docker）
1. **安装 Python 3.10+**（Windows 用户记得勾选 "Add to PATH"）。
2. **安装 uv（推荐）**：
   - Windows：运行仓库自带的 `setup-windows-en.bat`（首次执行会自动安装 uv）。
   - macOS/Linux：`curl -LsSf https://astral.sh/uv/install.sh | sh`。
3. **同步依赖**：在项目根目录执行 `uv sync`（或手动 `pip install -r requirements.txt`）。
4. **检查配置**：确保 `config/config.yaml` 与 `config/frequency_words.txt` 存在且语法正确。
5. **手动运行一次**：
   ```powershell
   # Windows PowerShell
   uv run python main.py
   ```
   ```bash
   # macOS / Linux
   uv run python main.py
   ```
   - 成功后会在 `output/` 目录生成 HTML/TXT 报告，并触发已配置的推送渠道。
6. **定时执行（可选）**：
   - Windows：使用“任务计划程序”创建每小时/每天任务，操作指向 `uv run python main.py`。
   - macOS：`launchd`；Linux：`cron`（例如 `0 * * * * cd /path/to/TrendRadar && uv run python main.py`）。
7. **常用调试**：
   - 查看日志：`uv run python main.py` 结束时控制台即是日志。
   - 只看配置来源：运行后日志会打印“配置来源: 环境变量/配置文件”。
   - 若需自定义配置路径，设置环境变量 `CONFIG_PATH` 指向新的 YAML。

### MCP AI 分析（可选）
1. 确保前述爬虫流程已运行过，`output/` 下有数据。
2. 启动 STDIO 服务器（供 Claude Desktop、Cline 等使用）：
   ```powershell
   uv run --directory . python -m mcp_server.server
   ```
3. 启动 HTTP 服务器（供 Cursor、Inspector、Claude CLI 使用）：
   ```powershell
   uv run python -m mcp_server.server --transport http --port 3333
   ```
4. 参照 `README-Cherry-Studio.md`、`README-MCP-FAQ.md` 为各客户端添加 MCP 配置。

## 方式二：Docker / Docker Compose
> 适合一键启动或在 NAS、群晖、家庭服务器部署。

### 准备步骤
1. 在宿主机创建目录（以当前用户目录为例）：
   ```bash
   mkdir -p trendradar/config trendradar/output
   cd trendradar
   ```
2. 下载配置模板：
   ```bash
   curl -o config/config.yaml https://raw.githubusercontent.com/sansan0/TrendRadar/master/config/config.yaml
   curl -o config/frequency_words.txt https://raw.githubusercontent.com/sansan0/TrendRadar/master/config/frequency_words.txt
   ```
3.（可选）下载官方 `docker/.env` 与 `docker-compose.yml`，便于集中管理环境变量。

### 直接使用官方镜像
```bash
docker run -d --name trend-radar \
  -v $(pwd)/config:/app/config:ro \
  -v $(pwd)/output:/app/output \
  -e FEISHU_WEBHOOK_URL="https://..." \
  -e CRON_SCHEDULE="*/30 * * * *" \
  -e RUN_MODE="cron" \
  -e IMMEDIATE_RUN="true" \
  wantcat/trendradar:latest
```
- 必要环境变量：至少配置一个推送渠道或保持本地仅写报告。
- `CRON_SCHEDULE` 控制运行频率，默认 `*/30 * * * *` 为每 30 分钟。
- `RUN_MODE=cron` 使用内置 `supercronic` 调度；设为 `oneshot` 时容器启动后只跑一次。

### 使用 docker-compose（推荐）
1. 编辑 `.env`，填入 Webhook、邮箱、定时表达式等环境变量。
2. `docker-compose.yml` 关键点：挂载 `config/`、`output/`，并引用 `.env`。
3. 启动：
   ```bash
   docker compose pull
   docker compose up -d
   ```
4. 常用命令：
   - 查看日志：`docker compose logs -f`
   - 手动执行一次：`docker exec -it trend-radar python manage.py run`
   - 查看当前设置：`docker exec -it trend-radar python manage.py status`

### 环境变量覆盖（v3.0.5+）
- `ENABLE_CRAWLER=true/false`、`REPORT_MODE=filtered`、`PUSH_WINDOW_*` 等变量可直接覆盖 `config.yaml` 对应字段。
- 日志会标注“来源: 环境变量/配置文件”方便确认。

## 常见问题整理
- **一定要用 Docker 吗？** 不需要。原生 Python 即可运行，仓库脚本已覆盖 Windows/macOS/Linux。
- **依赖安装失败**：检查 Python 版本是否 ≥3.10，或切换国内镜像源后重试 `uv sync`/`pip install`。
- **推送无内容**：若 `frequency_words.txt` 为空，默认推送所有热点；若仍为空，请检查 `config.yaml` 中 `crawler.enable_crawler` 与推送渠道是否启用。
- **Docker 修改配置不生效**：优先生效的可能是环境变量，检查容器的 `docker inspect` 或日志输出。
- **MCP 工具无法连接**：确认服务是否启动、端口是否被占用，必要时用 `start-http.bat`/`start-http.sh`。

## 下一步建议
- 根据需求调整 `config/config.yaml` 的 `report.mode` 与 `weight.*` 权重，打造符合使用场景的热点排序。
- 将所有敏感凭证保存在环境变量/Secrets 中，不要提交到仓库。
- 结合操作系统的定时工具或 Docker 的 `CRON_SCHEDULE` 实现持续抓取与推送。
