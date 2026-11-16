# honk-lzcapp

[honk](https://github.com/daeuniverse/honk) 的懒猫微服打包：实验性 eBPF 透明代理引擎（Linux）。数据面遵循 dae 设计，出站分组与 Clash API 风格源自 sing-box。GPL-3.0-only。

- 上游镜像：`ghcr.io/daeuniverse/honk`（GHCR 经 `ghcr.1ms.run` 加速）
- 版本策略：镜像模式（`delivery.mode: mirror` + `require_digest_match`），跟随上游 `v0.1.0-beta.N` SemVer prerelease tag 自动更新，只发布到喵喵商店
- 运行要求：懒猫微服内核 6.12+（`uname -r` 确认）；特权容器 + 宿主网络 + PID 命名空间（上游官方 Docker 部署方式相同）
- 与 dae / daed 互斥：同一台机器只能运行一个

## 与上游 Docker 部署的差异

| 项 | 上游 compose | 本打包 |
|----|-------------|--------|
| `privileged: true` | ✅ | ✅ compose_override |
| `network_mode: host` | ✅ | ✅ manifest `services.honk.network_mode`（平台原生字段，避免与平台 compose 网络互斥冲突） |
| `pid: host` | ✅ | ✅ compose_override |
| `/sys` bind mount | ✅ | ❌ **不能带**：bind 进来的 /sys 是静态视图，容器内新建的 dae0 不在其中，honk 读 dae0 MAC 会 ENOENT 退出；privileged 容器自带的真实 sysfs 可用 |
| `/run/honk-core.lock` 挂载 | ✅ | ❌ 懒猫盒子不跑原生 honk；宿主无该文件时 Docker 会把挂载点建成目录导致启动失败 |
| bpffs | 宿主挂载 | setup_script 内 `mount -t bpf bpf /sys/fs/bpf` |
| 配置 | 宿主 `/etc/honk/config.dae` | 首次启动生成（doona UI + direct fallback）到 `/lzcapp/var/etc/config.dae`，每次启动复制到 `/etc/honk/config.dae` |
| doona 监听 | `127.0.0.1:9527` | `0.0.0.0:9527`（网关经 host.lzcapp 访问；`allowed_hosts`/`allow_origins` 放行应用域名），密码登录（password_auth） |

## 首次使用

1. 打开应用，进入 doona 管理界面，首次访问创建管理员账号（存 `/lzcapp/var/data/state/honk.db`，升级保留）。
2. 默认全直连（`routing.fallback: direct`）。添加节点、把 fallback 改为 `proxy` 后开始代理；在 doona 界面编辑（config_write 已开启）可直接 reload，改文件则重启应用生效。
3. 默认只接管宿主机本机流量（`wan_interface: auto`）。要给局域网设备做网关，在 `global` 段加 `lan_interface: '<网桥名>'`。

## 自动发布

`.github/workflows/lazycat.yml` 使用 [ca-x/lazycat-github-action](https://github.com/ca-x/lazycat-github-action) `v1`：定时（每 3 小时）检查上游镜像新 tag，镜像模式校验 digest 后改写 manifest、构建 LPK、发 GitHub Release（版本化 LPK 附件）并发布到喵喵商店。所需 Secrets（仓库级）：`APPSTORE_URL`、`APPSTORE_TOKEN`。
