# Interactive Brokers Gateway Docker

<img src="https://github.com/UnusualAlpha/ib-gateway-docker/blob/master/logo.png" height="300" />

在 Docker 中运行 **IB Gateway** + **IBC**，实现无人值守 API 交易网关；策略通过 Socket API 连接宿主机映射端口。

> 裸机 / EC2 见独立仓库 [ibgateway](../ibgateway/README.md)（[单账户](../ibgateway/docs/ubuntu-install.md)、[多账户](../ibgateway/scripts/multi-account/README.md)）。

## Supported Tags

| Channel  | IB Gateway Version | IBC Version | Docker Tags                 |
| -------- | ------------------ | ----------- | --------------------------- |
| `latest` | `10.22.1m`         | `3.16.0`    | `latest` `10.22` `10.22.1m` |
| `stable` | `10.45.1f`         | `3.23.0`    | `stable` `10.45` `10.45.1f` |

See all available tags [here](https://github.com/UnusualAlpha/ib-gateway-docker/pkgs/container/ib-gateway/).

## 镜像内容与 API 端口

| 组件 | 作用 |
|------|------|
| [IB Gateway](https://www.interactivebrokers.com/en/trading/ibgateway-stable.php) | API 网关（stable **10.45.1f**，offline standalone） |
| [IBC](https://github.com/IbcAlpha/IBC) | 自动登录、处理弹窗 |
| [Xvfb](https://www.x.org/releases/X11R7.6/doc/man/man1/Xvfb.1.xhtml) | 虚拟显示 |
| [x11vnc](https://wiki.archlinux.org/title/x11vnc) | 可选 VNC（需设置 `VNC_SERVER_PASSWORD`） |
| [socat](https://linux.die.net/man/1/socat) | 将 API 从容器内 `4000` 映射到宿主机 `4001`/`4002` |

本仓库 `stable` 通道基础镜像：Ubuntu 22.04，`linux/amd64`。

| 模式 | 宿主机端口 |
|------|------------|
| Paper | **4002** |
| Live | **4001** |
| VNC（可选） | **5900** |

## 前置条件

- [Docker](https://docs.docker.com/get-docker/) 与 Docker Compose v2
- **Apple Silicon (M 系列)**：必须 `platform: linux/amd64`（`docker-compose.yml` 已配置）
- IBKR 账户 + **IB Key（2FA）**

## 快速开始

### 克隆仓库

```bash
git clone <your-repo-url> ib-gateway
cd ib-gateway
```

### 配置 `.env`

```bash
cp .env.example .env
vim .env
```

| 变量 | 说明 | 默认 |
|------|------|------|
| `TWS_USERID` | IBKR 用户名 | 必填 |
| `TWS_PASSWORD` | 密码 | 必填 |
| `TRADING_MODE` | `paper` 或 `live` | `paper` |
| `READ_ONLY_API` | `yes` / `no` | 未设置 |
| `VNC_SERVER_PASSWORD` | 设置则启动 VNC；不设则不启动 | 未设置 |

### 构建并启动

```bash
docker compose build
docker compose up -d
docker compose logs -f
```

- 启动后约 **30 秒** API 端口可用
- 手机 **IB Key** 批准 2FA

### 验证

```bash
ss -lntp | grep -E '4001|4002'    # Linux
# macOS: lsof -i :4002

docker compose ps
docker compose logs | grep -iE 'GATEWAY|vnc|error'
```

Paper 模式连接 **`127.0.0.1:4002`**，Live 为 **`127.0.0.1:4001`**。

## `docker-compose.yml` 说明

```yaml
services:
  ib-gateway:
    platform: linux/amd64
    build:
      context: ./stable
    environment:
      TWS_USERID: ${TWS_USERID}
      TWS_PASSWORD: ${TWS_PASSWORD}
      TRADING_MODE: ${TRADING_MODE:-paper}
    ports:
      - "127.0.0.1:4001:4001"
      - "127.0.0.1:4002:4002"
      - "127.0.0.1:5900:5900"
```

| 项 | 说明 |
|----|------|
| `build: ./stable` | 从仓库 `stable/` 构建镜像 |
| `127.0.0.1` 绑定 | API / VNC 仅本机可连 |
| `platform: linux/amd64` | IB 安装包为 x86_64 |

修改 `.env` 后：

```bash
docker compose down
docker compose up -d --force-recreate
```

## VNC（可选）

`.env` 中设置 `VNC_SERVER_PASSWORD` 后，日志应出现 `Starting VNC server`。

```bash
open vnc://127.0.0.1:5900
```

远程服务器：

```bash
ssh -N -L 5900:127.0.0.1:5900 ubuntu@SERVER_IP
```

## 远程访问 API

| 场景 | 做法 |
|------|------|
| 策略与容器同机 | `127.0.0.1:4002`（paper） |
| 策略在本地电脑 | `ssh -N -L 4002:127.0.0.1:4002 user@SERVER` |
| **禁止** | 对 `0.0.0.0/0` 开放 4001/4002 |

## 持久化（生产建议）

```yaml
services:
  ib-gateway:
    volumes:
      - ibgateway-jts:/root/Jts

volumes:
  ibgateway-jts:
```

`AutoRestartTime` 等见 `stable/config/ibc/config.ini.tmpl`（改后需 rebuild）。

## 构建与版本

- Dockerfile：`stable/Dockerfile`
- 更新版本：改 `stable/Dockerfile` 或 `./update.sh stable <version>`（在仓库根目录）

```bash
docker compose build --no-cache
```

## 自定义配置

| 应用 | 容器路径 | 模板 |
|------|----------|------|
| IB Gateway | `/root/Jts/jts.ini` | `stable/config/ibgateway/jts.ini` |
| IBC | `/root/ibc/config.ini.tmpl` | `stable/config/ibc/config.ini.tmpl` |

启动入口：`/root/scripts/run.sh`（镜像内）

## 常用命令

```bash
docker compose up -d
docker compose down
docker compose logs -f
docker compose exec ib-gateway env | grep TWS_
```

## 仓库结构

| 路径 | 说明 |
|------|------|
| `docker-compose.yml` | 本地构建与运行（`build: ./stable`） |
| `.env.example` | 环境变量模板 |
| `stable/` | stable 镜像 Dockerfile 与配置 |
| `latest/` | latest 通道（版本较旧） |
| `image-files/` | 同步到 `stable`/`latest` 的脚本与配置源 |
| `update.sh` | 按版本更新通道目录 |

## 参考

- [IB Gateway Stable](https://www.interactivebrokers.com/en/trading/ibgateway-stable.php)
- [IBC Releases](https://github.com/IbcAlpha/IBC/releases)

## License

See [LICENSE](LICENSE).
