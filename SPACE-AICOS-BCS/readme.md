## 项目介绍
算力调度基础设施是资源转化为算力的底层能力平台，维护并设定了资源组织形式、资源向算力转化引擎、算力管理模式。
SPACE-AICOS-BCS 定位于打造底层资源和业务实际应用场景之间的桥梁；致力于复杂应用提供一站式、低门槛的容器编排和服务治理服务。

### 核心能力
- 多云集群管理
  - 完成资源的就近组织，拓扑管理；
  - 设定资源节点的网络配套、存储配套等的作用范围；
  - 为算力的逻辑管理提供物理基础；

- 运行时管理
  - 将资源转化为多种形态的算力，并屏蔽资源的属性差异；
  - 为算力及其配套提供基础定义和组织形态；
  - 为算力提供识别、纳管、运行、调度能力；

- 应用管理
  - 描述应用组织形态，定义应用消费算力的最小管理单元；
  - 管理应用的生命周期，并驱动应用状态机流转；
  - 提供应用标准化交付形式，并为上层复杂消费场景提供基础SLA。

### 技术架构设计
![架构图](./docs/overview/arch.png)

### 高级特性
- [多卡适配]()
- [Pod原地升级InplaceUpdate](./docs/features/bcs-gamestatefulset-operator/inPlaceUpdate.md)
- [容器镜像热更新HotPatchUpdate](./docs/features/bcs-gamestatefulset-operator/hotPatchUpdate.md)
- [基于Hook的应用交互式发布](./docs/features/bcs-hoo-operator/README.md)
- [自动化分步骤灰度发布](./docs/features/bcs-gamedeployment-operator/features/canary/auto-canary-update.md)
- [PreDeleteHook & PreInplaceHook优雅地删除和更新Pod](./docs/features/bcs-gamedeployment-operator/features/preDeleteHook/pre-delete-hook.md)
- [镜像预热]()
- [容器web-console](https://bk.tencent.com/docs/document/6.0/144/6541)

### 应用实践
* [使用AICOS-BCS如何纳管已有k8s集群](https://bk.tencent.com/docs/document/6.0/144/8057#导入已有集群)
* [通过AICOS-BCS模板集部署应用](https://bk.tencent.com/docs/document/6.0/144/8054)
* [通过AICOS-BCS使用helm部署应用](https://bk.tencent.com/docs/document/6.0/144/6542)
* [通过GameStatefulset部署应用](./docs/features/bcs-gamestatefulset-operator/README.md)
* [通过AICOS-BCS完成应用的交互式灰度更新](./docs/features/bcs-gamedeployment-operator/features/canary/auto-canary-update.md)
* [通过AICOS-BCS完成业务的滚动升级](https://bk.tencent.com/docs/document/6.0/144/6517)
* [通过AICOS-BCS完成业务的蓝绿发布](https://bk.tencent.com/docs/document/6.0/144/6518)
* [如何在AICOS-BCS上插件容器监控信息](https://bk.tencent.com/docs/document/6.0/144/6515)

### 快速开始
* [下载与编译](docs/install/source_compile.md)
* [安装部署](docs/install/deploy-guide.md)
* [API使用说明](./docs/apidoc/api.md)

## AICOS社区
中国信通院牵头搭建SPACE AICOS开源社区,旨在汇聚行业各方力量,发挥协同优势与成员单位创新能力，通过关键技术攻关、行业标准制定、生态体系建设等工作，推动AI云操作系统技术创新与产业升级，助力各行业在人工智能驱动下实现数字化转型与高质量发展。

2025年6月，中国信通院联合天翼云、华为、中科院、移动云、摩尔线程、中国铁塔等单位，在全球计算联盟GCC下成立AI云操作系统专委会。专委会设立了首届主任委员、轮值主任委员及委员，并成立了专委会管理委员会及7大专项工作组，包括学术工作组、应用平台工作组、编排调度引擎工作组、推理训练引擎工作组、资源管理引擎工作组、开源生态工作组和标准化工作组,聚焦AI云操作系统的核心技术需求与生态建设。

### 社区核心工作方向
- **技术体系建设**: 围绕AI应用编排、算力调度、资源管理及底层资源适配开展技术研究与落地，打造全栈式AI云操作系统技术体系;
- **专项工作研究**: 联合学术、应用平台、编排调度、推理训练、资源管理、开源生态、标准化等多方位专家,针对性开展技术与生态建设工作;
- **生态协同发展**: 联合行业内科研机构、云服务商、运营商、智算中心、硬件厂商等多方主体，推动技术成果落地与产业应用，覆盖能源、金融等多行业领域;
- **标准与人才建设**: 制定AI云操作系统相关行业标准，推动开源生态建设，同时开展人才培育工作，为行业输送专业技术人才，助力产业创新发展。

### 标准体系建设
SPACE AICOS联合中国信通院从AI云操作系统总体架构、架构各层级及兼容性、安全性、可靠性、关键性能方面规划了完善的标准体系，并逐步启动各项标准的编制工作，旨在为产业发展提供统一的规范和指导。本标准体系已设立19项重要标准，目前《基于AI云操作系统的大模型推理加速能力要求》等三项标准已成功立项。

### 贡献指南
欢迎所有开发者参与SPACE AICOS开源社区建设，提交功能特性、修复问题、完善文档。

详细贡献规范请查看 docs/CONTRIBUTING.md
