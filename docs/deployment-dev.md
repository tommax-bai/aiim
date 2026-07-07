# AIIM 部署 · dev 环境

> 首次上线 2026-07-07。dev 复用 aidcp 的 ECS `121.89.85.150`，但**独立目录 / 端口 / systemd 单元**，
> 与同机 `isales`、`aidcp-cloud`、`nginx` 完全隔离——**绝不碰它们**。不记录任何敏感值。

## 目标与隔离边界

| 项 | aiim | 同机他人（勿碰） |
| --- | --- | --- |
| 机器 | `121.89.85.150`（key `~/codes/isales-4.pem`，user `root`） | 同 |
| 目录 | `/opt/aiim`（bundle + `.env`） | aidcp `/opt/aidcp`；isales 独立 |
| 端口 | **8990** | aidcp-cloud 8787、aidcp console/nginx 8088/80、isales 8000/80 |
| systemd | `aiim.service` | `aidcp-cloud.service`、nginx、isales |

## 形态：单文件 bundle + 纯 node（无 tsx / 无 node_modules）

box 的 npm registry 网络不通、且 tsx 依赖的 esbuild 是平台原生二进制（本地 macOS-arm64 ≠ box Linux-x64）。
故用 esbuild 把服务打成**自包含单文件**，box 上纯 `node` 直跑：

```bash
# 本地打包（aiim-service）
npm run build            # → dist/aiim-server.mjs（~40KB，跨平台 ESM）
```

服务入口 `apps/server`：`GET /health`、`POST /webhook`（协议回调入站）、`POST /intake`（外部加微指令）。

## 当前部署状态（2026-07-07）

- `aiim.service` **active + enabled**（开机自启）。
- `provider=fake`（骨架模式）：服务能起、能收 webhook、能健康检查，但**不做真实微信收发**。
- 切真实收发需：`.env` 设 `AIIM_PROVIDER=wework` + `WEWORK_BASE_URL` + `WEWORK_GUID`
  （guid 需先经协议服务商扫码开通实例——见 change `friend-add-closed-loop` task 0.1）。

## 重部署（redeploy）

```bash
KEY=~/codes/isales-4.pem
# 1) 本地打包
cd ../aiim-service && npm run build
# 2) 传单文件（只覆盖 bundle，不动 .env / 不动 aidcp/isales）
rsync -az -e "ssh -i $KEY" dist/aiim-server.mjs root@121.89.85.150:/opt/aiim/aiim-server.mjs
# 3) 重启 + 健康检查
ssh -i $KEY root@121.89.85.150 'systemctl restart aiim.service && sleep 2 && systemctl is-active aiim.service && curl -s localhost:8990/health'
```

## 配置（`/opt/aiim/.env`，不进 git、不记敏感值到仓库）

`AIIM_PROVIDER`（fake|wework）、`AIIM_PORT=8990`、`AIIM_ACCOUNTS`、`AIIM_QUOTA_LEVEL`；
wework 模式另需 `WEWORK_BASE_URL`、`WEWORK_GUID`。

## 健康检查 / 运维

```bash
ssh -i ~/codes/isales-4.pem root@121.89.85.150 \
  'systemctl status aiim.service --no-pager | head; curl -s localhost:8990/health; journalctl -u aiim.service -n 30 --no-pager'
```

红线：任何 ECS 操作只作用于 `/opt/aiim` + `aiim.service` + 端口 8990；`isales`/`aidcp-*` 绝不触碰。
