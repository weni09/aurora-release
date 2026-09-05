# AuroraMihomo

> 本仓库是 AuroraMihomo 的**公开发布库**：只存放安装脚本、Docker Compose 与 GitHub Release 二进制，不含源码。
> 安装、升级、面板内自升级都从本仓库拉取。

Mihomo 内核运行时与配置管理平台：订阅聚合、配置合并、冲突处理、内嵌面板，一体化管理。

## 功能

- **订阅管理**：远程订阅与手动粘贴节点，支持流量信息展示（已用/总量/到期）、自定义 User-Agent、独立处理管道
- **组合订阅**：多订阅聚合，12 种处理算子（过滤/改名/加国旗/地区筛选/排序/JS 脚本等），多种输出格式
- **分享分发**：订阅与组合均可生成免登录分享链接，支持 `?target=` 切换客户端格式、`?filter=` 临时筛选
- **配置合并**：Base / Remote / Override 三层模型，自动检测冲突并支持本地优先、远程优先、自动合并、手动指定四种解决策略
- **版本管理**：配置变更自动备份（保留 10 份）与版本回滚
- **内核管理**：mihomo 启停、重载、日志实时推送、配置校验失败自动回滚
- **透明代理**：Linux（TUN / TProxy）与 macOS（TUN），带环境检测、强制确认与自动回滚
- **内嵌面板**：Zashboard 挂载在 `/ui/`
- **自动更新**：mihomo 与 Zashboard 定时更新，内置 8 个 CDN 源依次回退

单个静态二进制，不依赖 CGO 与外部 C 库，Alpine 与 Debian/Ubuntu 通用。

完整使用说明部署运行后可在面板侧边栏的「使用文档」里查看。

## 快速开始

两种部署方式，按场景选：

| 方式 | 适合 | 前置要求 |
|---|---|---|
| [Docker](#方式一docker) | 大多数场景，升级最省事 | Docker CLI（compose 方式另需 Docker Compose） |
| [二进制](#方式二二进制) | 不想引入 Docker；透明代理最省事 | 无（静态二进制） |

---

### 方式一：Docker

镜像已发布到 GitHub Container Registry（`ghcr.io/weni09/auroramihomo`，多架构 amd64/arm64），**免本地构建**，`docker run` 或 `docker compose` 直接拉取即可。默认拉 `latest` 标签；要锁定版本，设环境变量 `AURORA_VERSION`（如 `AURORA_VERSION=0.11.0`）。

> 镜像名是全小写的：GHCR 包名强制小写，写成大写（如 `ghcr.io/weni09/AuroraMihomo`）会报 `NAME_INVALID` 拉不下来。

**无法直连 GitHub？** 镜像存放在 GHCR，需要先换镜像加速站再拉取：

```bash
# compose：只改前缀（含仓库路径、不含 tag），AURORA_VERSION 锁定版本照常生效
AURORA_REGISTRY=ghcr.nju.edu.cn/weni09/auroramihomo docker compose up -d

# docker run：整条镜像名换成加速站
docker run -d --name auroramihomo ... ghcr.nju.edu.cn/weni09/auroramihomo:latest
```

本仓库实测 `ghcr.nju.edu.cn`（南京大学镜像站）可匿名拉取；其余加速站接口不稳定，用前先自行验证。容器内下载 mihomo 内核 / Zashboard 面板走内置 CDN 源（默认含 ghproxy.com 等），也可在「系统设置 → 下载与更新出网」或环境变量 `AURORA_CDN_PROVIDERS` 调整。

#### 方式 A：docker compose（推荐，含健康检查与安全项）

**1. 获取 compose 文件**

无需 clone 仓库，只下载这一个文件即可：

```bash
mkdir -p auroramihomo && cd auroramihomo
curl -fsSL -o docker-compose.yml \
  https://raw.githubusercontent.com/weni09/aurora-release/main/docker/docker-compose.yml
```

**2. 数据目录**

数据默认落在 compose 文件同级的 `./data`（相对路径按 compose 文件所在目录解析）。换位置时用绝对路径覆盖：

```bash
AURORA_DATA_DIR=/opt/aurora/data docker compose up -d
```

容器默认以内置账户 `aurora`（uid/gid 10001）运行，启动时会自动修正挂载目录属主，**无需手工 chown**。两种可选调整：

- **让数据归当前用户管理**（宿主机不用 sudo 就能读写 `data/`）：把容器运行账户改成你的 uid/gid，并把目录一并 chown 给自己：

  ```bash
  chown -R "$(id -u):$(id -g)" data
  AURORA_PUID=$(id -u) AURORA_PGID=$(id -g) docker compose up -d
  ```

- **不关心属主，直接放开写权限**（容器以任意 uid 都能写，属主保持现状）：

  ```bash
  sudo chmod -R a+rwX data   # 777 的温和版：只加读写位，不额外设执行位
  ```

**3. 设置 JWT 密钥（生产环境建议）**

不设也能跑（首次启动会随机生成并存入数据库），但显式设置便于多实例共用与灾备恢复：

```bash
# 编辑 docker-compose.yml，取消 AURORA_JWT_SECRET 注释并填入随机长串
openssl rand -hex 32   # 生成一个
```

**4. 启动**

```bash
docker compose up -d
```

首次会自动从 GHCR 拉取镜像；随后容器内会自动下载 mihomo 内核与 Zashboard 面板，需要能访问 GitHub，视网速可能要几分钟。

**5. 确认启动成功**

```bash
docker compose ps        # 状态应为 healthy
curl -fsS http://127.0.0.1:8899/healthz && echo OK
```

**6. 取初始密码并登录**

```bash
docker compose logs | grep 初始管理员密码
# 或直接读文件
docker exec auroramihomo cat /data/initial_password.txt
```

浏览器打开 `http://<宿主机IP>:8899`，用该密码登录。

> 首次成功登录或在「系统设置 → 管理员密码」改密后，程序会自动删除该明文文件。请仍尽快修改初始密码。

**升级**

```bash
# 可选：更新 compose 文件本身
curl -fsSL -o docker-compose.yml \
  https://raw.githubusercontent.com/weni09/aurora-release/main/docker/docker-compose.yml
docker compose pull    # 拉取新镜像
docker compose up -d
```

数据都在 `data/` 卷里，升级不丢配置。

#### 方式 B：docker run（免 clone，一条命令）

数据目录无需预先 chown，容器启动时自动修正属主。想让数据归当前用户管理，加 `-e AURORA_PUID=$(id -u) -e AURORA_PGID=$(id -g)` 并把 `data/` chown 给自己（写法见方式 A 第 2 步）：

```bash
mkdir -p data
docker run -d \
  --name auroramihomo \
  --restart unless-stopped \
  --network host \
  --cap-add NET_ADMIN \
  -v "$(pwd)/data:/data" \
  -e AURORA_JWT_SECRET="$(openssl rand -hex 32)" \
  ghcr.io/weni09/auroramihomo:latest
```

初始密码：`docker logs auroramihomo | grep 初始管理员密码`。升级：`docker rm -f auroramihomo` 后重新执行上面的启动命令（`latest` 会拉取最新版）。

#### 内核 / Zashboard 初始化失败与手动放置

首次启动时容器内会自动从 GitHub 下载 mihomo 内核与 Zashboard 面板。若下载失败（无法直连 GitHub、CDN 源全挂等），日志会反复出现：

- `mihomo binary missing, downloading...` / `zashboard assets missing, downloading...` —— 启动时检测到缺失，开始下载
- `查询 ... 最新版本失败` —— 连不上 `api.github.com`（多为网络问题）
- `download failed via ...` / `all download sources failed: ...` —— 版本能查到，但所有下载源都失败（CDN 源被墙/失效）

注意：**面板本身不受影响**，仍可正常访问（`FailOnEnsureError=false`，内核缺失只记日志不退出）；症状是代理内核起不来、`/ui/` 返回 404。先尝试「系统设置 → 下载与更新出网」调整 CDN 源顺序，或设 `AURORA_CDN_PROVIDERS` / `AURORA_GITHUB_API`（见「环境变量」表）；仍不行就按下面手工放置。

**mihomo 内核 → 容器内 `/data/bin/mihomo`，即宿主挂载目录 `data/bin/`**：

```bash
# 从 https://github.com/MetaCubeX/mihomo/releases/latest 选对应平台资产，
# 如 mihomo-linux-amd64-v1.19.29.gz（Linux/macOS 是 .gz 裸二进制，Windows 是 .zip）
mkdir -p data/bin
curl -fsSL -o data/bin/mihomo.gz https://github.com/MetaCubeX/mihomo/releases/download/v1.19.29/mihomo-linux-amd64-v1.19.29.gz
gunzip -f data/bin/mihomo.gz
chmod +x data/bin/mihomo
docker restart auroramihomo   # 内核在启动时检测，放好后需重启容器
```

**Zashboard 面板 → 容器内 `/data/zashboard/`（存在 `index.html` 即生效，无需注册），即宿主 `data/zashboard/`**：

```bash
# 资产名固定为 dist.zip，可直接用 latest 下载：
# https://github.com/Zephyruso/zashboard/releases/latest/download/dist.zip
rm -rf data/zashboard && mkdir -p data/zashboard
curl -fsSL -o /tmp/zashboard.zip https://github.com/Zephyruso/zashboard/releases/latest/download/dist.zip
unzip -o /tmp/zashboard.zip -d /tmp/zash
find /tmp/zash -name index.html          # 找到含 index.html 的目录
cp -r /tmp/zash/<含 index.html 的目录>/* data/zashboard/
```

放好后刷新 `http://<宿主机IP>:8899/ui/` 即可，无需重启。无法直连 GitHub 时，给上面的 URL 套 CDN 前缀（如 `https://ghproxy.com/https://github.com/...`）。手工放置的文件无需担心属主：容器（重启）时入口脚本会自动把 `/data` 修正为运行账户所有。

**GeoSite / GeoIP 规则库（默认不下载）→ 需要时放容器内 `/data/`（宿主同级 `data/`）**

开箱 base **不引用** `geosite:` / `fallback-filter.geoip`（也不写 `geox-url`），因此默认配置校验时 mihomo 不需要规则库、不会联网下载，弱网/离线也能通过「立即拉取」。**只有**配置里出现 `geosite:` / `geoip:` 引用（如订阅自带规则、你在基础配置里加了 geosite 分流）时，mihomo 才会去下载缺失库——默认源是 GitHub，直连超时的典型日志：

```text
can't download GeoSite.dat: ... context deadline exceeded
configuration file /data/config.yaml test failed
validate config failed, rolled back
```

此时任选其一：

1. **配置中心 → 基础配置** 增加 `geox-url`（换成国内可直连的 jsDelivr 镜像）后重新「立即拉取」：

```yaml
geox-url:
  geoip: "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geoip.dat"
  geosite: "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geosite.dat"
  mmdb: "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/country.mmdb"
  asn: "https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/GeoLite2-ASN.mmdb"
```

2. **手工放置**（文件名大小写按 mihomo 约定；`geodata-mode: false` 时至少需要 `GeoSite.dat` 与 `country.mmdb`）：

```bash
# 在 compose 所在目录（即挂载到 /data 的 data/）
curl -fsSL -o data/GeoSite.dat \
  https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geosite.dat
curl -fsSL -o data/country.mmdb \
  https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/country.mmdb
# 可选：geodata-mode: true 时用 GeoIP.dat；ASN 匹配用
# curl -fsSL -o data/GeoIP.dat https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/geoip.dat
# curl -fsSL -o data/ASN.mmdb https://testingcf.jsdelivr.net/gh/MetaCubeX/meta-rules-dat@release/GeoLite2-ASN.mmdb
```

文件到位后无需重启容器，再点「立即拉取」即可。

#### 透明代理的额外改动

默认配置不支持透明代理。需要时编辑 `docker-compose.yml`，取消 `user: "0:0"`、`AURORA_RUN_AS_ROOT` 与 `devices` 的注释，并注释掉 `no-new-privileges`。这会降低容器隔离性，详见面板「使用文档」中的透明代理章节。

另外在**宿主机**写入转发与 `rp_filter`（容器内不会改 sysctl；host 网络下 Docker 也会拒绝相关 `--sysctl`）：

```bash
sudo cp scripts/sysctl-auroramihomo.conf /etc/sysctl.d/99-auroramihomo.conf
sudo sysctl -p /etc/sysctl.d/99-auroramihomo.conf
# 可选：BBR（内核需支持；与转发项分文件）
sudo cp scripts/sysctl-auroramihomo-bbr.conf /etc/sysctl.d/99-auroramihomo-bbr.conf
sudo sysctl -p /etc/sysctl.d/99-auroramihomo-bbr.conf
```

TProxy 需要的 `iptables`/`nftables`/`iproute2` 已预装在镜像里，开箱可用。两种模式的依赖不同：TUN 需要映射 `/dev/net/tun` 但不需要这些命令行工具（mihomo 自己走 netlink）；TProxy 反之，不需要 tun 设备。

---

### 方式二：二进制

产物是静态二进制，不依赖 CGO 与外部 C 库，同一个包在 Alpine（musl）与 Debian/Ubuntu（glibc）上通用。

#### 在线安装

```bash
curl -fsSL https://raw.githubusercontent.com/weni09/aurora-release/main/scripts/install.sh \
  | sudo sh -s -- --repo weni09/aurora-release
```

一条命令装完即可访问，Debian/Ubuntu 与 Alpine 都是。脚本会：

1. 探测系统、架构与服务管理器（systemd 或 OpenRC）
2. 从 Release 拉对应包、校验 sha256、解压到 `/opt/auroramihomo`
3. 补齐透明代理依赖：缺失时装 `iptables`/`nftables`/`iproute2`（Alpine 还有独立的 `ip6tables`），加载 `tun` 与 `nft_tproxy` 并写 `/etc/modules-load.d/` 持久化；**写入 `/etc/sysctl.d/99-auroramihomo.conf`**（`ip_forward`、`ipv6.conf.all.forwarding`、`rp_filter=2`，见 `scripts/sysctl-auroramihomo.conf`）并 `sysctl -p` 加载；内核支持时 **best-effort 启用 BBR**（独立文件 `99-auroramihomo-bbr.conf` / `scripts/sysctl-auroramihomo-bbr.conf`，失败只告警不阻断安装）
4. 装服务单元 —— systemd 装 unit，Alpine 装 OpenRC 脚本（用 `supervise-daemon`，面板的「重启」依赖它）
5. 启用开机自启并启动服务

**开箱默认配置：** 首次启动若数据库中还没有基础配置（base），会自动写入
`backend/internal/service/default_base.yaml`（构建时嵌入二进制）。内容从真实部署
提炼并去掉个人数据：公共 DNS/DoH、`nameserver-policy`、私网直连规则、`MATCH,DIRECT`
兜底、TUN 排除地址等。**不会**预写订阅节点、设备 SRC-IP、内网 DNS、口令或
`tproxy-port`（透明代理仍由面板开关写入）。已有 base 时不会覆盖。

升级时重跑同一条命令：保留现有配置与已有服务单元，自动停服替换再拉起。

想先看清它要做什么，加 `--dry-run` 只打印不执行：

```bash
curl -fsSL https://raw.githubusercontent.com/weni09/aurora-release/main/scripts/install.sh \
  | sudo sh -s -- --repo weni09/aurora-release --dry-run
```

可选参数：

```bash
sudo sh install.sh --version v0.2.0      # 装指定版本
sudo sh install.sh --dir /srv/aurora     # 换安装目录
sudo sh install.sh --no-deps             # 不动系统：不装包、不加载内核模块、不写 sysctl
sudo sh install.sh --no-service          # 不装服务单元（旧名 --no-systemd 仍可用）
sudo sh install.sh --no-start            # 装好但不启用/不启动
```

`--no-deps` 之后透明代理仍需手工补齐依赖，命令见下面「Alpine 补充说明」。容器内运行时脚本会跳过依赖补齐（装的包重建即丢、`modprobe`/`sysctl` 会作用于宿主内核），容器部署请用 Docker 镜像；**宿主机**请另执行：

```bash
sudo cp scripts/sysctl-auroramihomo.conf /etc/sysctl.d/99-auroramihomo.conf
sudo sysctl -p /etc/sysctl.d/99-auroramihomo.conf
# 可选 BBR
sudo cp scripts/sysctl-auroramihomo-bbr.conf /etc/sysctl.d/99-auroramihomo-bbr.conf
sudo sysctl -p /etc/sysctl.d/99-auroramihomo-bbr.conf
```

（`docker/docker-compose.yml` 注释里也有同一套步骤。）

#### 离线安装

内网机器上，先在有网的机器下载对应平台的包：

```
auroramihomo_<版本>_linux_amd64.tar.gz
auroramihomo_<版本>_linux_arm64.tar.gz
auroramihomo_<版本>_darwin_arm64.tar.gz
```

拷到目标机后：

```bash
sudo mkdir -p /opt/auroramihomo
sudo tar -xzf auroramihomo_<版本>_linux_amd64.tar.gz --strip-components=1 -C /opt/auroramihomo
cd /opt/auroramihomo
./auroramihomo -f etc/aurora-api.yaml
```

包内含二进制与默认配置（前端资源已内嵌进二进制），**不含 mihomo 内核**。完全离线时需手工放置内核：

```bash
# 从 https://github.com/MetaCubeX/mihomo/releases 下载对应平台的 .gz
# 注意 Linux/macOS 官方发的是 .gz（gzip 压缩的裸二进制，不是 tar 归档）
sudo mkdir -p /opt/auroramihomo/data/bin
gunzip -c mihomo-linux-amd64-v1.19.29.gz | sudo tee /opt/auroramihomo/data/bin/mihomo >/dev/null
sudo chmod +x /opt/auroramihomo/data/bin/mihomo
```

同理，Zashboard 面板可手工放到 `data/zashboard/`；不放则 `/ui/` 不可用，不影响主面板。

#### 作为服务运行

在线安装脚本已按系统自动装好服务单元（systemd 或 OpenRC）并启动，这一节只在离线安装或想自定义单元时才需要。

**systemd（Debian/Ubuntu 等）**

```bash
sudo tee /etc/systemd/system/auroramihomo.service >/dev/null <<'EOF'
[Unit]
Description=AuroraMihomo
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/opt/auroramihomo
ExecStart=/opt/auroramihomo/auroramihomo -f /opt/auroramihomo/etc/aurora-api.yaml
Restart=always
RestartSec=3
# 主程序自升级会短暂退出再拉起；默认 KillMode=control-group 会把
# cgroup 里仍在跑的 mihomo 一并杀掉，旁路由/TProxy 会全面断网。
KillMode=process

# 透明代理需要的权限。不用透明代理可删掉这两行并加 User=nobody
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_RAW CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_RAW CAP_NET_BIND_SERVICE

# UDP 代理并发高时容易撞上文件描述符上限
LimitNOFILE=1000000

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now auroramihomo
sudo systemctl status auroramihomo
sudo journalctl -u auroramihomo -f      # 看日志
```

进程自身不做 fork 自重启，生产环境依赖 systemd 的 `Restart=always`。

**OpenRC（Alpine）** 在线安装脚本会自动装下面这份脚本，离线安装时手工放。要点是用 `supervise-daemon` 而非 `start-stop-daemon` —— 面板的「重启」是「优雅退出、等进程管理器拉起」，后者不做拉起，会让重启变成单向关机：

```bash
sudo tee /etc/init.d/auroramihomo >/dev/null <<'EOF'
#!/sbin/openrc-run
name="auroramihomo"
directory="/opt/auroramihomo"
command="/opt/auroramihomo/auroramihomo"
command_args="-f etc/aurora-api.yaml"
command_user="root:root"
# 不重定向 stdout/stderr：应用日志由面板写 <data>/logs/aurora.log
# 并受日志清理任务管理，不再往 /var/log 写重复副本。
supervisor="supervise-daemon"
pidfile="/run/auroramihomo.pid"
depend() { need net; after firewall; }
EOF

sudo chmod +x /etc/init.d/auroramihomo
sudo rc-update add auroramihomo default
sudo rc-service auroramihomo start
sudo tail -f /opt/auroramihomo/data/logs/aurora.log
```

#### Alpine 补充说明

在线安装脚本已自动处理下面第一组（透明代理必需项）。离线安装、或用了 `--no-deps` 时需手工执行：

```bash
# ip6tables 是独立包；iproute2 会把 /sbin/ip 从 busybox 换成真 iproute2
sudo apk add --no-cache iptables ip6tables nftables iproute2
# tun 与 nft_tproxy 默认未加载
sudo modprobe tun && sudo modprobe nft_tproxy
printf 'tun\nnft_tproxy\n' | sudo tee /etc/modules-load.d/auroramihomo.conf
```

排查工具脚本不装（不是运行所必需，装不装取决于你的习惯）：

```bash
# 默认只有 busybox 的 wget，不支持 -x 与 --interface
sudo apk add --no-cache curl bind-tools
```

根分区余量值得先确认：官方 cloud 镜像可能只有 100M 出头，而部署约需 55M（二进制 29M + 前端 3M + 内核 15M + 依赖包）。安装脚本会在可用空间不足时告警，但不阻断。

取初始密码：

```bash
sudo cat /opt/auroramihomo/data/initial_password.txt
```
