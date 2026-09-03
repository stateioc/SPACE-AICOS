# gateway-cpp

`bocloud.cpp.gateway`（含其编码逻辑 `bocloud.cpp.encoder`）的 **C++** 重写。

职责：**从运营商算力平台拉数据 → 本地编码成 CPURL → 上报算力监测/解析平台**，
并把字典/配置从“每分钟拉取的内存缓存”改为**本地 JSON 文件 + 内存快照**（目标服务器禁用 SQLite）；
同时编码能力以 **C ABI DLL** 对外暴露，可被外部程序调用直接生成算力标识文件。

> 编码主链路（注册码/资源码/规格码/路径码）已**逐方法忠实移植**自 Java，并按对接文档 v5.4 修正。
> **尚未达成“与现网 Java 逐字节一致”**——需黄金文件对拍（见「完成度」与「路线」）。

> ## ⚠️ 移植基准：以 `bocloud.cpp.gateway` 为唯一权威
>
> 代码库里有**两套** Java 编码器，且**行为不同**，对拍/比对时务必认准：
>
> | Java 位置 | 是否权威 | 资源目录(`resourceCatalog`)来源 |
> |---|---|---|
> | `bocloud.cpp.gateway/.../gateway/encoder/*` | ✅ **本项目移植目标** | 由 server 类型**合成** 401/402/403 + size 2/3 特殊次序 + 过滤空 chips |
> | `bocloud.cpp.encoder/.../encoder/*`（共享模块） | ❌ 行为不同，**勿对照** | 取 `summary.resourceCatalog` 经 `formatResourceCatalog` |
>
> 上面第 3 行“含其编码逻辑 `bocloud.cpp.encoder`”指的是 *职责归属*，**不代表**字节语义以该模块为准——
> gateway 自带一份独立的 `encoder` 包，C++ 实现是它的端口。
> 二者差异点至少包括：**资源目录来源**、`handleServer` 是否过滤空 chips、`encodeResourceCatalog` 的映射分支。
> 审查/对拍/改编码语义前，先 `grep` 确认你读的是 `bocloud.cpp.gateway` 那套。
> （历史教训：曾因对照了 `bocloud.cpp.encoder` 模块而误判 C++ 有“分叉”，实际 C++ 本就忠实对齐 gateway。）

---

## 1. 架构 / 模块映射

```
拉取(HTTPS+签名)            上报(HTTPS+gzip)
Authority(运营商) ──► gateway-cpp ──► 监测/解析平台
                         │
   字典(/api/sms) ──► Refresher ──► dict.json(本地文件) ──► 内存 Snapshot(原子换) ──► Encoder
                                                          ▲
   外部程序 ──JSON──► [DLL gw_encode_to_file] ───────────┘ → {ent}-CPURL-DATA-{ts}{seq}.data
```

| Java（gateway/encoder） | C++（本项目） | 状态 |
|---|---|---|
| `domain/*` | `domain/Models` | ✅（按 v5.4 文档修正） |
| `encoder/StandardEncoder` `core/standard/*` | `encoder/StandardEncoder` | ✅ 忠实移植 |
| `encoder/{Res,Spec,Path}CodeEnCoder` | `encoder/SubEncoders` | ✅ 忠实移植 |
| `encoder/StandardEnCoderManager` | `encoder/EncoderManager` | ✅ |
| `detector/*` / `util/IpMac*` | `detector/Detectors` / `encoder/NetAddr` | ✅ |
| `cache/*`+`entity/*`+`CacheFetchTask` | `store/FileStore`(JSON)+`store/Snapshot`+`collector/Refresher` | ✅（本地文件持久化） |
| `auth/*`（HttpDataFactory/各 Maker/Sign） | `auth/{Auth,Sign}` | ◑ 联通/移动/default ✅；电信/雅安/甘肃 ⚠️回退 |
| `core/standard/StandardResourceCollector` | `collector/Collector` | ◑ common+电信解析 ✅ |
| `core/CpurlUploader` / `StandardCollectorScheduler` | `collector/Uploader` / `collector/Scheduler` | ✅（含 gzip>2MB） |
| `daemon/LocalCacheDaemon` | 刷新线程（`main.cpp`） | ✅ |
| — （新增对外能力） | `capi/gwcpp_api`（C ABI DLL）+ `encoder/CityZone` | ✅ |

---

## 2. 构建

依赖：CMake ≥ 3.20、C++20 编译器；系统库 **curl / openssl / zlib**；
`nlohmann-json`、`yaml-cpp` 由 CMake `FetchContent` 自动拉取（需网络）。
（注：**纯编码 DLL 零运行时第三方依赖**——字典是 JSON 文件、nlohmann 头文件内联；curl/openssl/zlib 仅采集/上报/字典初始化用。）

```bash
# 系统库（macOS）
brew install cmake curl openssl zlib
# 系统库（Debian/Ubuntu）
sudo apt install cmake g++ libcurl4-openssl-dev libssl-dev zlib1g-dev

./scripts/build.sh
# 或：cmake -S . -B build && cmake --build build -j
```

**打包**（产出可部署压缩包 lib/bin/include/cnf/sql/tools/docs）：
```bash
./scripts/package.sh        # 编译+测试+CPack → build/gateway-cpp-<ver>-<os>-<arch>.tar.gz
```

> 完整编译/打包/跨平台(Windows MSVC)/vcpkg/最小 DLL 产物 → **[`docs/BUILD.md`](docs/BUILD.md)**。

产物：`build/gateway`（守护进程）、`libgwcpp.{so,dylib}`/`gwcpp.dll`（对外 DLL）、`build/gwcpp_*_tests`（测试）。

---

## 3. 运行

**① 守护进程（采集→编码→上报 + 字典刷新）**
```bash
./build/gateway cnf/application.yaml
```
- `store.mode: online`：启动即从上游拉字典写 `dict.json`，并每 60s 刷新。
- `store.mode: offline`：只读本地 `dict.json`（需先按 `tools/seed_import.py` 灌库）。
- **主动上报默认关闭**（`push.enabled: false`）：网关被动接受 DLL 调用生成标识文件，不主动采集/上报。
  设 `push.enabled: true` 才启动推送线程（每 `collector.report_cron_minutes` 分钟采集→编码→上报）。
- `Ctrl-C`/SIGTERM 优雅退出。

**② 外部程序调用 DLL 直接生成标识文件（被动）**
```c
gw_ctx* c = gw_open("dict.json");
gw_encode_to_file(c, resource_json, /*is_total*/1, /*is_monitor*/0, "/data/out", &res);
gw_close(c);                              // 详见第 6 节
```

测试：
```bash
ctest --test-dir build --output-on-failure
# golden_encoder（编码） / capi_file（落盘集成） / security（不可信输入回归） / register_validation（注册码校验）
```

> **部署与配置**（DLL 放哪、字典/输出路径怎么配、程序 A 怎么加载）→ **[`docs/DEPLOY.md`](docs/DEPLOY.md)**。

---

## 4. 完成度与三大字节级一致性风险

编码“写得出”已完成；“**和现网 Java 逐字节一致**”是真正的成本，集中在三处（与立项讨论一致）：

1. **签名规范化**（`auth/Sign.cpp`）
   HMAC-SHA256/SHA-256 本身已对齐；风险在待签串拼装：联通是 `sort(key)→"k=v"&...`、
   `v` 用原始字符串（bool→`true/false`、无引号）。需用**签名黄金用例**逐字节核对。

2. **BigDecimal 取整**（`common/Decimal.{h,cpp}`）
   现以 `long double` + 显式 `HALF_UP/UP` 复刻。边界值（`*1024`、`/1000`、GB/TB/PB、
   TFLOPS/PFLOPS 切换）必须对拍；必要时整体换成 `boost::multiprecision::cpp_dec_float`
   （API 已隔离，换实现不动调用方）。

3. **多芯片型号拼接顺序**（`encoder/StandardEncoder.cpp` 的 chipCapacity）
   Java 用 `HashMap` 迭代序（依赖 `String.hashCode`）；C++ 用有序 `map`，
   单资源含多种芯片型号时 `ResCode` 拼接顺序可能不同 → 该条 CPURL 串不同。
   对拍按【集合】比较；若上游对顺序敏感，需复刻 Java HashMap 桶序或与平台确认。

> 另注：`SpecCodeEnCoder` 中 `capacity % 1014`（疑似 Java 笔误）已**刻意保留**以求一致，
> 见 `src/encoder/SubEncoders.cpp` 注释。

> **刻意偏离 Java（按对接文档 v5.4 修正，已确认）**：`networkCapacityIp`（Java 误用
> `Tcp` → N00 恒 0）与 `serverType` 在 ZoneServer 层（Java 放 Server 层 → FSU 失效）两处，
> C++ 取**标准正确**行为。故含 IP 带宽或 FSU 超算的资源，CPURL 会与现网 Java 有意分叉
> （N00 段 / 401↔402 / FIN↔FSU），对拍时按标准为准、不是缺陷。详见 `tools/upstream_json_schema.md`。

### 验证方法（务必照做）
- **P0 黄金对拍**：导出现网字典 + 采集输入 + Java 产出 CPURL，断言 C++ 排序后逐条相等
  （脚手架见 `test/golden_encoder_test.cpp`；字典用 `tools/seed_import.py` 生成）。
- **P5 影子运行**：C++ 与 Java 双跑同源数据，diff 上报内容，连续 N 天零差异再切流。

---

## 5. 路线

- [x] 本地 JSON 字典文件 + 原子快照 + 字典刷新（Refresher）
- [x] HTTP 采集/上报/gzip（default/联通/移动）+ 调度循环
- [x] 编码主链路忠实移植（Reg/Res/Spec/Path + 校验 + 容量）
- [x] C ABI DLL（`gw_encode_json` / `gw_encode_to_file`）+ 落盘文件格式
- [x] 安全加固（不可信输入 DoS / 文件名注入 / TLS，详见 `SECURITY.md`）
- [ ] **黄金文件对拍**（最高优先：可行性闸门）← 需现网字典 + 输入 + Java 产出 CPURL
- [ ] P3 补齐电信(20087)/雅安(20402)/甘肃(20036) 鉴权与响应解析
- [ ] 打包脚本 / 多平台 CI

---

## 6. 作为动态库对外发布（DLL / .so / .dylib）

> **外部系统对接完整说明见 [`docs/INTEGRATION.md`](docs/INTEGRATION.md)**
> （前置条件、接口、输入/输出格式、C#/Python/Java/C 调用示例、错误码、运维要点）。

编码能力以 **C ABI** 对外暴露（`include/gwcpp/capi/gwcpp_api.h`），可被
C / C++ / C#(P/Invoke) / Python(ctypes) / Java(JNA) / Go(cgo) 等直接调用。
该库**零运行时第三方依赖**（字典是 JSON 文件、nlohmann 头文件内联，编码不碰网络），轻量易分发。

```c
gw_ctx* ctx = gw_open("dict.json");                 // 加载字典快照

// 用法一：直接拿到 CPURL（JSON 返回）
char* out = NULL;
gw_encode_json(ctx, resource_json, 1, &out);      // 1=全量(K1) 0=可用(K2)
// out = {"success":true,"message":"...","cpurls":[...]}
gw_free(out);

// 用法二：编码并落盘（外部程序 A 的目标场景）
char* res = NULL;
gw_encode_to_file(ctx, resource_json,
                  1 /*is_total: K1*/, 0 /*is_monitor: 0=调度 1=监测*/,
                  "/data/cpurl", &res);
// 一次调用 → 一个文件：
//   {out_dir}/{enterprise}-CPURL-DATA-{yyyyMMddHHmmss}{NNNN}.data
//   （NNNN：同目录同秒 4 位序号 0001 起，避免重名/覆盖）
// 文件内容 = JSON 数组，每个 (city,zone) 组一个元素：
//   [ {enterprise,city,zone,isMonitor,cpurls:[...],usage:[{ip,cpuUsage,memUsage},...]}, ... ]
//   （usage 按原始 ip 去重聚合该组所有机器负载；不参与编码）
// res = {"success":bool,"message":"ok=N failed=M","path":str,"record_count":N,
//        "groups":[{enterprise,city,zone,count,usage_count}|{...,error}]}
gw_free(res);

gw_close(ctx);
```

**落盘行为约定**：
- 每个 `(city, zone)` 一个文件；某组若 city 不在字典会单独失败并记入 `files[].error`，不影响其它组。
- 文件名含 `city/zone` 以避免同企业同时间戳重名（已含你要的 `{enterprise}_..._CPURL_DATA_{ts}.dat` 形态）。
- ⚠️ 全量(K1) 与可用(K2) 是**两次独立调用**、文件名不含 K1/K2 标记；若两次落到**同一目录同一秒**会同名覆盖。建议二者写不同 `out_dir`，或按需我在文件名里加 `_K1/_K2` 段。

- **线程安全**：`gw_encode_json` 可并发；`gw_reload` 原子热换字典，不阻塞编码。
- **内存约定**：返回的 `char*` 由库 `malloc`，调用方用 `gw_free` 释放。
- **ABI 干净**：仅导出 7 个 `gw_*` 符号，内部 C++/STL 符号 `visibility=hidden`。

构建：
```bash
cmake --build build --target gwcpp_shared   # → libgwcpp.{so,dylib} / gwcpp.dll
cmake --build build --target c_client       # 纯 C 调用示例（examples/c_client.c）
```

**Windows `.dll` 要点**：
- 已用 `__declspec(dllexport/dllimport)` 宏（见 capi 头），CMake 自动定义 `GWCPP_BUILDING_DLL`；
- `NetAddr.cpp` 已按平台切换 `ws2tcpip.h`，CMake 在 `WIN32` 下自动链 `ws2_32`；
- 依赖（nlohmann，编码 DLL 仅此一个且头文件内联）用 vcpkg：`-DCMAKE_TOOLCHAIN_FILE=...\vcpkg.cmake`；
- 静态链接进调用方时，在包含头文件前定义 `GWCPP_STATIC` 以关闭 dllimport。

> 当前 DLL 暴露的是**编码器**（无状态、给字典+资源→CPURL）。若需把整条
> “采集→编码→上报”守护进程嵌入宿主程序，可另加 `gw_start/gw_stop`（依赖 curl/openssl/zlib），属 P4。

---

## 7. 目录

```
gateway-cpp/
├── CMakeLists.txt / vcpkg.json / scripts/{build,package}.sh
├── cnf/application.yaml         # 运行配置
├── include/gwcpp/**            # 头文件（按模块；capi/ 为对外 C ABI）
├── src/**                      # 实现（common/domain/store(FileStore)/detector/encoder/net/auth/config/collector/capi + main）
├── examples/{c_client.c,gw_gateway.hpp,cpp_app.cpp}  # C / C++(RAII封装) 调用示例
├── test/{golden_encoder,capi_file,security}_test.cpp
├── docs/{INTEGRATION,cpp_client_guide,BUILD,DEPLOY}.md  # 对接 / C++手册 / 编译打包 / 部署
└── tools/{seed_import.py,upstream_json_schema.md}  # 拉字典生成 dict.json + 输入契约
```
