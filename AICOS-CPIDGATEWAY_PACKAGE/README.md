# bocloud.cpp.gateway 部署说明

算力标识**采集上报网关**（端口 18000，无 DB），部署在**运营商现场**。它拉取企业资源、编码 CPURL、采集服务器使用率（含 CPU 核数 `cpu` / 内存 `memory` GB），上报到上游平台 booter，由 booter 落 ClickHouse。

> 与本机/交付环境 75 上的 `bocloud.gateway-6.6.0.jar`（平台 API 路由网关）**不是同一个东西**，互不相关。

## 数据流

```
拉企业资源(StandardResourceCollector)
  → 编码 CPURL + 采集 usages[{ip,cpuUsage,memUsage,cpu,memory}]
     （cpu/memory 由 ResourceUsageCollector 从 Server.cpuCores / memoryCapacity 填，memoryCapacity 单位即 GB）
  → POST 上报到上游 booter（地址在各现场 cnf/application.yml 的 heartbeat.address）
  → booter 落 ClickHouse cpp.server_usage_detail（含 cpu/memory 列）
```

**关键**：哪个现场网关把数据发给目标 booter，就升级哪台的 jar。现场网关不升级 → 上报不带 cpu/memory → detail.cpu/memory 为 0 → 加权利用率读数为 0。

## 关键配置

- **采集频率** `collector.collect-interval`（单位**分钟**，默认 **10**）：由该配置统一维护，**同时影响两件事**——两个推送调度器（`CollectorScheduler`/`StandardCollectorScheduler`）的采集/上报节奏，以及心跳上报给监测平台的 `frequency` 字段（动态生成"每隔N分钟"）。改一处即全联动。
  - 各现场按实际频率在**该现场的 `application.yml`** 的 `collector` 块设置（如 `collect-interval: 5`）；不配则用默认 10 分钟。生产值部署时注入，不改交付仓库默认值。

## 制品构建

```bash
cd backend/bocloud.cpp
git checkout develop && git pull            # 确保含 cpu/memory 改动
mvn -pl bocloud.cpp.gateway -am -DskipTests clean package
```

产物（`bocloud.cpp.gateway/target/`）：

| 文件 | 用途 |
|---|---|
| `bocloud.cpp.gateway-3.0.1-install.tar.gz` / `.zip` | **安装部署包**（bin/cnf/lib/pid/log 目录树），全新安装用 |
| `bocloud.cpp.gateway-3.0.1.jar` | 裸 fat jar，老现场整 jar 覆盖升级用（见下"现场部署脚本"） |
| `docker/dist/cpp-gateway.zip` | 华为 NE40E OAS 容器包（另行执行 `docker/make-oas-package.sh` 生成，见 `docker/README.md`） |

## 全新安装（安装部署包）

包内目录树（解压后根目录 `gateway/`，与现有现场 `/usr/local/gateway` 布局一致）：

```
gateway/
├── bin/   start.sh（启动，已运行则先停后启=重启）、stop.sh
├── cnf/   application.yml（运维配置，覆盖 jar 内同名默认值）、logback-spring.xml
├── lib/   bocloud.cpp.gateway-3.0.1.jar
├── log/   cpp.log（应用日志）、gateway-console.out（启动引导）、gateway-startup-error.out（启动失败抽取）
└── pid/   gateway.pid
```

**环境要求**：JDK 21（`JAVA_HOME` 指定，未设则用 PATH 的 java）；端口 18000；出方向可达上游采集接口与上级平台；内存预留 1.5GB（JVM 默认 `-Xms256m -Xmx1g`）。

```bash
# 1. 解压（以 /usr/local 为例 → /usr/local/gateway）
tar -xzf bocloud.cpp.gateway-3.0.1-install.tar.gz -C /usr/local
cd /usr/local/gateway

# 2. 按环境改配置：heartbeat.address/clientId/clientSecret（上级平台）、
#    collector.enterprise/address/authority/zones（上游采集）、collect-interval
vi cnf/application.yml

# 3. 启动并验证
./bin/start.sh
curl http://127.0.0.1:18000/status     # 返回 "I am OK!" 即成功
tail -f log/cpp.log

# 停止 / 重启
./bin/stop.sh
./bin/start.sh
```

可选环境变量（写入 `bin/setenv.sh`，start.sh 自动 source）：`JAVA_HOME`、`GATEWAY_JAVA_OPTS`（默认 `-Xms256m -Xmx1g`）、`CPP_GATEWAY_LOG_DIR`（默认安装目录 log/）、`GATEWAY_STARTUP_GRACE`（判活秒数，默认 5）。

**升级**：`./bin/stop.sh` → 备份并替换 `lib/` 下 jar（`cnf/` 不动）→ `./bin/start.sh` → curl /status。回滚即还原备份 jar 重启。

**启动失败排查**：看 `log/gateway-startup-error.out`；常见为端口占用（`ss -lntp | grep 18000`）、JDK 版本不是 21。

## 现场部署脚本（均为 scp jar → /usr/local/gateway/lib/ 后 ssh 启动；整 jar 覆盖 + 重启，不动 cnf）

| 脚本 | 现场 | host | 启动脚本 |
|---|---|---|---|
| `telecom.sh` | 电信 | 203.25.213.248 | start.sh |
| `unicom.sh` | 联通 | 116.169.59.219 | startup.sh |
| `mobile.sh` | 移动 | 36.170.93.150 | startup.sh |
| `scstxx.sh` | 四川专线 | 175.155.64.77 | start.sh |

```bash
cd backend/bocloud.cpp/bocloud.cpp.gateway
bash telecom.sh        # 或 unicom.sh / mobile.sh / scstxx.sh，按需要升级的现场
```

---

## 部署 Checklist

### 前置
- [ ] 目标 booter（接收上报的那台）**已升级到加权版并已执行 ClickHouse 迁移**（detail 有 `cpu/memory` 列）。未迁移就升级网关，booter 写入会因缺列报错。
- [ ] 已在 `develop` 构建出 `target/bocloud.cpp.gateway-3.0.1.jar`。
- [ ] 确认该现场 `cnf/application.yml` 的上报目标（heartbeat.address / source targets）指向上面这台已升级的 booter。
- [ ] 部署是运营商**生产机**操作，已走发布审批；有当前 jar 的回滚备份。

### 部署
- [ ] 跑对应现场脚本（如 `bash telecom.sh`）。
- [ ] 确认 scp 成功、启动脚本无报错；网关进程（端口 18000）重新起来。

### 验证（部署后一个采集周期内）
- [ ] 现场网关日志：一轮上报的 `usages[]` 每条带了 `cpu` 和 `memory`（非空、非 0）。
- [ ] 接收端 booter 的 ClickHouse 出现真实非 0 核数/内存：
  ```sql
  SELECT toString(ip), cpu_usage, mem_usage, cpu, memory
  FROM cpp.server_usage_detail
  WHERE cpu > 0
  ORDER BY report_time DESC
  LIMIT 10;
  ```
- [ ] 加权利用率读数正常（不再是 0）：
  ```sql
  SELECT enterprise,
         round(sum(cpu_usage*cpu)/nullIf(sum(cpu),0),2)        AS cpu_weighted,
         round(sum(mem_usage*memory)/nullIf(sum(memory),0),2)  AS mem_weighted
  FROM cpp.server_usage_detail
  WHERE report_time >= now() - INTERVAL 1 HOUR
  GROUP BY enterprise;
  ```
- [ ] 前端「负载运行分析」看板利用率非 0、数值合理。

### 已知/注意
- 上游数据源若某些企业不返回核数/内存 → 落 0 → 该机加权权重为 0（不计入），不报错（按设计）。
- 网关无 DB、无需迁移；CH 迁移是 booter 侧（脚本 `backend/bocloud.cpp/docs/sql/clickhouse_server_usage_v2_weighted.sql`）。
- 加权口径与字段细节见 `docs/superpowers/specs/2026-06-23-server-usage-weighted-design.md`。
- ClickHouse 在 75 仅本机可连（`ssh root@10.20.12.75` 后 `clickhouse-client --user default --password <CH密码，见 booter application.yml 的 clickhouse.password>`），8123 对外防火墙不可达。
