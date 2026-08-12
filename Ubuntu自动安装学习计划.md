# Ubuntu 自动安装（autoinstall）学习计划（小白→专家）

> 依据《学习方法论与协作协议.md》规划。版本：2026-08-12
> 场地：Windows 11 + VMware；playground VM（Ubuntu 24.04.4 / 192.168.56.66 / NAT）
> 现有资产：`user-data.yaml`（桌面版）+ `user-data-server.yaml`（服务器版）两份可用的 autoinstall 配置，本计划把它们当"教材基线"
> 前提：已具备 SSH 基础与 Linux 基础（不重讲）；目标对接后续 8 台拓扑 / 腾讯云模拟（CVM 镜像、CLB、TKE=k3s）
> 原则：不背命令、学"会折腾+会定位"；每课可验收；命令全明文；失败 3 次升级；先方案后执行

**版本决策（2026-08-12）**：以 **24.04.x LTS 为主学**（和现有 user-data 文件、playground 一致，社区资料最多）；**26.04 LTS** 已于 2026-04-23 发布（Server 默认自动装 OEM/HWE 内核、桌面安装器改 Flutter），每课只记差异，不两边铺开。若想直接吃新版，说一声整体切 26.04（键名基本兼容）。

---

## 一、先立观念（大白话）

- **自动安装 = 把"装系统的点鼠标过程"变成一份 YAML 说明书**。手动装系统是在图形界面里一步步点（选语言、分磁盘、设用户）；autoinstall 就是把这每一步写成配置，安装器照着跑完，人不用在场。
- **Ubuntu 的"自动安装"特指一套四件套**：**Subiquity 安装器**（文本/图形安装程序）读 **autoinstall.yaml**（安装时的选择，如磁盘怎么分、装哪些软件），把结果交给 **curtin**（真正写盘的执行者）和 **cloud-init**（第一次开机时的系统配置，如建用户、装软件）。三层别混，装完第一台你自然分得清。
- **这套东西的真正价值：批量、可重复、配置即代码**。手装 1 台和装 100 台难度一样——只是复制一份配置、改几个参数。运维后半段的 Ansible、k8s 也都是同一逻辑（声明式配置驱动），这是运维的分水岭，autoinstall 是最典型的入门载体。
- **你手头已是成品**：两份 user-data 文件能直接装出一台中文桌面/服务器、装完自动关机。学这个方向是从"会抄"到"会写配置、会排故障、会批量交付"。
- **装系统的坑不是语法，是"配置没给到"**：yaml 写对不保证装对，得看懂安装器是怎么拿到配置、写到哪个盘、跑在哪一层。这门课 80% 的时间花在"为什么没按我写的来"，这是最重要的判断力。

---

## 二、通用工具链与方法（所有阶段共用）

**一套固定工作流，每课循环：**
```
写/改 autoinstall.yaml → 塞进 ISO 或 HTTP 服务器 → 启动安装器自动装 → 验收（SSH 登录+检查分区/软件/配置）→ 记坑 → git 提交配置
```

| 工具 | 用途 | 备注 |
|---|---|---|
| VMware Workstation | 新建测试机、挂 ISO、快照回滚 | 每课实验前打快照；动磁盘类操作尤其 |
| 官方 ISO（24.04.x LTS） | 安装源 | 从 ubuntu.com 下载 |
| autoinstall-generator / 手动改 ISO | 把配置塞进安装介质 | 阶段1 起用；阶段3 学手动 grub.cfg 改法 |
| dnsmasq / iPXE / grub | 网络引导（PXE） | 阶段4 起 |
| nginx / python http.server | 分发 autoinstall 与镜像文件 | 批量部署的"中央服务器" |
| cloud-init 命令（status/analyze/clean） | 调试开机后配置 | 阶段2 主战场 |
| cloud-init schema | 配置格式校验，喂给安装器前先验 | 阶段1 起，少交一半学费 |
| openssl passwd -6 / mkpasswd | 生成密码 hash（$6$） | 阶段1、5 |
| qemu-img / cloud-localds | 云镜像转 vmdk + NoCloud seed 盘 | 阶段4 |
| jinja2 / Python / sed | 配置模板化、批量生成 | 阶段5 |
| git | 配置仓库，每课提交 | 出问题 diff 回滚 |
| git hook / GitHub Actions / 脚本 | 配置自动校验与产物构建（CI 最小形态 → 正规军） | 阶段5、7 |
| Subiquity 文档 + `subiquity` 日志 | 查键名/语法、排安装故障 | autoinstall-reference 是第一手册 |

**通用避坑（所有阶段都会踩）：**
- **语法错误静默**：autoinstall.yaml 写错，安装器常不报错，而是停在"无配置"或装错布局。排查靠 subiquity 安装日志 + 小步验证（每次只改一个键）。
- **配置的三种提供方式别混**：cloud-init user-data / ISO 介质 / PXE 网络，同一份 yaml 三种传法，参数和路径都不一样。
- **password 必须是 hash**（`$6$...`），不是明文；哈希含特殊字符时在 yaml 里用引号包住。
- **动磁盘永远有风险**：storage 一节只改测试机，测试机专用，别用带数据的分区。
- **late-commands 里是安装器环境**：改目标系统必须 `curtin in-target -- ...`，直接写写的是安装器自己的环境。
- **真实装一台要 5-15 分钟**：批量测之前先做配置校验/干跑，别每改一行就全装一遍。
- **配置先校验再装机**：改完配置先 `cloud-init schema --config-file xxx.yaml` 验格式再喂安装器，别靠"装一台试试"。
- **自己起的 DHCP 会和 VMware NAT 的 DHCP 打架**：PXE 阶段用独立 VMnet（关掉该网段 VMware DHCP）或 host-only，同网段只能一个权威 DHCP（详见附录 C）。
- 与内核课程同源的三条：**先方案后执行**、**一次只破坏一个点**、**每阶段写复盘**。

**通用最佳实践：**
- 配置全部进 git，每课一个提交，装炸了 `git checkout` 回滚配置。
- 从 server 版入门（轻、无 GUI），桌面版是它的超集，阶段1 最后再碰。
- 每份配置顶部写清楚"这是给谁装的"（角色/磁盘/内存），贴进文件注释。
- 复现问题时记录"三件套"：完整报错 + 该份配置 + 安装/cloud-init 日志。
- 配置模板化后，**机器数量多到超过 5 台才值得上模板**，1-2 台直接手写。

---

## 三、环境评估与开工准备（2026-08-12 现状）

| 项目 | 现状 | 判定 |
|---|---|---|
| autoinstall 配置 | `user-data.yaml`（桌面）+ `user-data-server.yaml`（服务器）已就绪 | ✅ 现成教材 |
| 场地 VM | playground（24.04.4 / 192.168.56.66 / 1G）可当配置生成、HTTP 分发服务器 | ✅ |
| 网络 | VMware NAT 192.168.56.0/24，宿主机 .1 / 网关 .2 | ✅ apt 可用 |
| 官方 ISO | 未下载（需代理，24.04.x LTS 或 26.04 LTS） | ⚠️ 阶段1 先下载 |
| 目标测试机 | 无专用安装测试 VM | ⚠️ 每课新建 1-2G 测试 VM，装完即弃 |
| 账号 | user / 密码哈希已知（$6$...，两份文件一致） | ✅ |

**开工清单（执行时一次性配齐，先方案后执行）：**
1. 下载官方 Server ISO（阶段1 只用 server 版；desktop ISO 留到阶段1 末尾对比）。版本选择：**以 24.04.x LTS 为主**（和现有文件/playground 一致、资料多）；26.04 LTS 已发布（2026-04-23），差异点在每课标注，不另开战场。
2. 新建 2 台专用"安装测试"VM（各 1-2G 内存 / 20G 磁盘 / CD 挂 ISO），命名如 `ai-test1 / ai-test2`，与 8 台拓扑分开。
3. 打干净基线快照（装系统=磁盘操作，虽比内核开发安全，仍打快照防呆）。
4. 把两份 user-data 文件 init 成 git 仓库，作为教材基线。
5. 记录 playground 关机/开机、SSH 登录现状（开工时确认）。

---

**前提盘点（不重讲，也不假设你会）**：Linux 基础命令与 SSH 你已会（直接复用）；netplan、LVM、DHCP/HTTP 这些概念分别在阶段1-L4/L5、阶段4 顺带讲透，不单独设课；Docker 只需验收能跑（`docker version`），深入用法归运维七阶段（不越界）。

---

## 四、学习路线（7 阶段）

> 每 5-6 课一阶段，阶段末综合实战验收。每课结构见"五、每课模板"。
> 每阶段末尾补"工具与方法 / 避坑经验 / 最佳实践"三块。

### 阶段 1：概念 + 第一次无人值守安装（6 课）⭐入门分水岭

1. **安装器简史与链路**：debian-installer/preseed → Subiquity/autoinstall；自动安装整体链路（启动→安装器→curtin→cloud-init→应用）
2. **autoinstall.yaml 全键解析**：version/identity/locale/keyboard/updates/shutdown——用现有 `user-data-server.yaml` 逐键对照读
3. **配置三种提供方式**：cloud-init user-data / ISO 介质 / PXE，先走 ISO 方式跑通第一台
4. **storage 磁盘布局**：`name: lvm` vs `direct` vs `zfs` + actions 手动分区（看懂现有文件里的 lvm 到底分了几块）
5. **应用层**：packages/snaps/ssh/source/updates——装完能 SSH、能用 python/docker
6. **late-commands / early-commands** 与阶段综合验收

- **验收**：用现有 `user-data-server.yaml` 从 ISO 全自动装出一台 VM，SSH 免密/密码登录成功，docker+python 栈可用，装完自动关机（`shutdown: poweroff`）
- **工具与方法**：
  - 新建 VM 选"稍后安装"，挂 ISO，启动即自动装（注意 VMware 首次启动选"我稍后安装操作系统"，否则不引导）
  - 安装过程切 tty（Ctrl+Alt+F2~F5）看 subiquity 进度与报错；装完 `ssh user@<IP>` 验收
  - 验证命令固定一套：`id`、`docker version`、`python3 -c "import numpy"`、`cat /etc/default/locale`
  - 每次只改一个键、验证一个键；配置文件每课 `git commit`
- **避坑经验**：
  - **配置没给到**：ISO 方式要正确改 ISO 引导参数（加 `autoinstall`），否则进交互模式"看起来没自动"——这是新手第一卡
  - `shutdown: poweroff` 会让 VMware 显示"未正常关机"，是**正常现象**，不是故障
  - 用 server ISO 装不出桌面；source 选 `ubuntu-desktop` 但介质是 server ISO → 行为不符
  - 中文 locale 只改 yaml 不够，late-commands 里的 `locale-gen` 不能省（现有文件已有，理解为什么）
  - lvm 布局装完：`/dev/ubuntu-vg/ubuntu-lv`，别找 `/dev/sda3` 找不到就慌
- **最佳实践**：
  - 第一阶段只跟 server 版走，桌面版（用户密码/自动登录坑更多）留到阶段1末尾做一次对照
  - 每份实验配置复制成 `lab/xxx.yaml`，不污染作为教材的两份基线
  - 装完立刻打快照 `installed-base`，后面所有实验从干净系统出发

### 阶段 2：cloud-init —— 装完之后的自动化引擎（5 课）

1. **cloud-init 生命周期**：开机运行、datasource 发现、三阶段（init→network→config），autoinstall 只是它的一个特殊用法
2. **四件套与优先级**：user-data / meta-data / vendor-data / network-config，谁覆盖谁
3. **核心模块**：users、packages、write_files、runcmd、ssh_authorized_keys、apt、apt-sources
4. **调试三板斧**：`cloud-init status`、`cloud-init analyze blame`、`cloud-init clean --reboot` 重跑
5. **NoCloud datasource 本地实操**：seed.iso / 目录直接喂给一台空 VM，脱离安装器只测 cloud-init

- **验收**：一台 VM 开机后自动完成——建用户+注入 SSH key、装指定软件、写一个配置文件、跑一段初始化命令，全程无人；能用 `cloud-init analyze` 说出每步耗时
- **工具与方法**：
  - `cloud-init query --all` 看最终生效的数据；`cat /var/log/cloud-init.log` 看执行
  - NoCloud 用目录方式最省事：`datasource_list: [NoCloud]` + seed 目录
  - 想重跑：`sudo cloud-init clean --reboot`，别重装系统
- **避坑经验**：
  - **模块没跑**：默认模块在 `/etc/cloud/cloud.cfg` 配置，被注释/被 vendor 覆盖就不执行——先查生效配置再怀疑语法
  - runcmd 里的命令失败**不中断**后续，要看日志确认真的成功了
  - write_files 写二进制/大文件别硬塞，用 base64 或单独放文件
  - 网络配置四件套里 `network-config` 优先级高于系统 netplan，改错会断网
  - 与 autoinstall 的关系：autoinstall 的 identity/packages 最终就是转成 cloud-init 数据，理解这一点阶段5 的模板化才有意义
- **最佳实践**：
  - 一切可配置项优先用 cloud-init 的声明式模块，`runcmd` 是兜底，别全堆 runcmd
  - 每台机器 `#cloud-config` 头必带，没头就不是 cloud-init 配置
  - 测试固定场景：从空 VM → cloud-init 全自动 → 验收清单逐项勾

### 阶段 3：定制 ISO + 离线安装（5 课）

1. **改官方 ISO 塞配置**：autoinstall-generator / 手动改 `boot/grub/grub.cfg` + user-data（看懂阶段1用的到底是什么法）
2. **Cubic 定制 ISO**：加驱动、预置软件、改 squashfs——做一张"放进去就自动装好"的介质
3. **离线源**：本地 mirror / apt-cacher-ng / debmirror，断外网也能装（服务器机房常见场景）
4. **存储进阶**：LVM 细分卷、swap、LUKS 全盘加密（阶段1 只用了 lvm 默认布局）
5. **安装后置与 OEM**：curtin hooks、snap 预置、OEM 包机制、26.04 的 HWE/OEM 内核默认安装说明

- **验收**：一张自定义 ISO，放进任意空机自动装好，且全程不依赖外网（本地源），装完带加密盘
- **工具与方法**：
  - 改 ISO：解包→改 grub.cfg/塞 user-data→mkisofs 重打包；或先用 autoinstall-generator 自动做再手改
  - LUKS：autoinstall 里 `storage: layout: {name: lvm}` + 加密选项，或 actions 手动指定 `encrypt: true`
  - 离线源先起一台 apt-cacher-ng 缓存机（playground 可当），再让安装器指向它
- **避坑经验**：
  - ISO 重打包后引导失败 → 多半是 `boot/grub` 的路径/`isolinux` 改错，先用官方 ISO 对照
  - 加密盘密码要在配置里写死或安装后手动输入，**忘了写 → 装完起不来**
  - HWE/OEM：26.04 Server 默认自动装匹配硬件的 OEM/HWE 内核，VM 里一般用不上，可 `oem: install: false` 关掉——理解默认变化而不是盲抄
  - 离线源缺包 → 装到一半报"无法下载"，先跑 `apt-cacher-ng` 预热再装机
- **最佳实践**：
  - 自定义 ISO 打版本号，避免"不知道哪张 ISO 装的"；配置和 ISO 一起进 git
  - 阶段3 产出的"介质+源+存储"组合，就是阶段6 企业交付的最小单元

### 阶段 4：PXE / HTTP 网络引导批量部署（6 课）⭐真正的分水岭

1. **引导链路拆解**：DHCP → TFTP/HTTP → 内核+initrd → 目标系统（IPMI/BMC 概念顺带讲清）
2. **iPXE / grub 从网络拉 Subiquity 内核与 initrd**
3. **autoinstall 网络配置**：netplan 语法、dhcp/static/vlan/bonding（装的就是将来要用的网络）
4. **HTTP 服务器分发**：nginx 放 autoinstall 与镜像，一台服务器喂多台机器
5. **Ubuntu 云镜像直装**：cloud image（qcow2）+ NoCloud seed，不经 ISO——现代云厂商建机本质（VMware 里先 `qemu-img convert` 成 vmdk 再起）
6. **阶段综合验收**：一台 HTTP/PXE 服务器，无人值守批量装 3 台不同角色（网关/DB/Web）

- **验收**：关掉光驱、空机通电即自动从网络装好并带角色配置；重复装 3 台无人工
- **工具与方法**：
  - dnsmasq 一次搞定 DHCP+TFTP；HTTP 用 nginx 或 `python3 -m http.server`
  - iPXE 脚本里用变量传 autoinstall 的 URL（`autoinstall=...` 内核参数）
  - 云镜像（VMware 落地法）：下载官方 cloud image → `qemu-img convert -f qcow2 -O vmdk` → 建 VM 从 vmdk 起 → `cloud-localds` 生成 NoCloud seed.iso 挂第二光驱（`ds=nocloud`）→ 开机即自动配好
- **避坑经验**：
  - **PXE 起不来 80% 是 DHCP+TFTP 链路**：按层查（dhcp 租约→tftp 文件→http 可达），别直接怀疑配置
  - **自己起的 dnsmasq 会和 VMware NAT 的 DHCP 冲突**：PXE 实验给测试机用独立 VMnet（自定义网络，关闭该网段 VMware DHCP），dnsmasq 只管这网段（详见附录 C）
  - **qcow2 不能直接给 VMware 引导**：先 `qemu-img convert` 成 vmdk，否则起不来
  - 内核参数 `autoinstall` 拼写错 → 静默进交互模式
  - 批量装先装 1 台验证，再并行，别 3 台一起翻车
  - 网络配置写死在 autoinstall 里 → 装完固化成系统 netplan；**stage 阶段就配好**，别装完再手动改
- **最佳实践**：
  - 网络引导是"装系统"和"云/裸机运维"的桥梁，本阶段吃透，后面 MAAS/腾讯云 CVM 全是同一模型
  - HTTP 服务器和配置放同一台（playground 可承担），路径固定、URL 可复用
  - 每台装完记录 MAC/角色/IP 一张表，批量部署的可审计性从这里开始

### 阶段 5：配置工程化（5 课）

1. **模板化生成 autoinstall**：jinja2 / Python 脚本生成，一份模板多角色输出（server/desktop/worker）
2. **secret 管理**：密码 hash、SSH key、token 不入库；hash 生成方法（`mkpasswd`/openssl）
3. **CI 自动化校验与出制品（新增）**：配置仓库 + CI 最小形态（git hook / 本地脚本）——push 前自动 `cloud-init schema` 校验 + 模板渲染检查 + 配置清单核对；进阶用 GitHub Actions 自动构建定制 ISO / 云镜像，固化"改配置 → 自动验证 → 自动出产物"
4. **生产存储策略**：全盘加密 + LUKS、RAID/mdadm、多盘区分数据盘/系统盘
5. **与 Ansible 的分工**：autoinstall 装"系统+基础"，Ansible 配"服务+应用"——边界在哪、为什么
6. **reporting/日志/失败处理**：安装报告、错误上报、失败重装与断点

- **验收**：一份"配置即代码"仓库，改一个参数能装出任一角色，secret 全从变量注入；**CI 校验不过不允许装机**（推 git 自动跑 schema/渲染检查，输出全贴）
- **工具与方法**：
  - jinja2 模板 + `ansible-vault` 或 env 注入管理 secret；hash 用 `openssl passwd -6`
  - 模板渲染后先 `subiquity` 配置校验/干跑，再真装
  - reporting 键把安装结果发到日志服务器/文件，出问题批量回看
  - CI 最小形态：仓库根放 `scripts/validate.sh`（`cloud-init schema` + 模板渲染检查），挂 git pre-commit hook 自动跑；进阶推 GitHub Actions 自动构建 ISO（阶段7-3 深化）
- **避坑经验**：
  - 模板渲染出的 yaml 缩进错 → 安装器静默跳过，**先渲染出成品再喂给安装器**
  - 密码 hash 别直接写模板，否则 git 里泄露；统一从变量来
  - autoinstall 与 Ansible 重叠区（都装软件/建用户）：**划清边界**——系统层归 autoinstall，应用层归 Ansible，别两边都做
  - RAID/加密参数写错 → 装完盘上数据不可恢复，测试机专用+快照
  - CI 里 secret 不进日志：校验失败报错别把渲染后的 hash/token 打全（输出只贴指纹/前几位）
- **最佳实践**：
  - 每个角色 = 一个模板 + 一组变量，产出物进 git 并打 tag
  - 加"渲染校验"进 CI/脚本：`python render.py && check-yaml`，别等装到一半才发现
  - 阶段5 的产物直接升级成阶段6 的交付库

### 阶段 6：企业级平台与云对齐（5 课）→ 半专家

1. **MAAS 大规模裸机自动化**（Canonical Metal as a Service）：DHCP/IPMI/PXE 一体化的企业版模型
2. **iPXE 高级**：引导菜单、多环境（prod/staging）、安装回滚、多镜像切换
3. **安全加固安装**：最小化安装、全盘加密、账号/审计策略、关闭不必要服务（对齐你"安全/权限防兜圈"思路）
4. **与云厂商对齐**：腾讯云 CVM 自定义镜像 / 阿里云镜像 / AWS AMI——cloud-init 同源，镜像制作思路对比（对齐你后续腾讯云模拟）
5. **阶段综合验收**：用 autoinstall 批量装出 8 台拓扑的 node1-5（网关/nginx-LB/Ansible+Prometheus/web/MySQL），各自带角色配置

  - **衔接说明（与 8 台拓扑的关系）**：本阶段默认在**独立一次性测试 VM** 上演练批量安装，不干扰运维学习已有的"克隆建机"流程；若你决定改用 autoinstall 正式重建 node1-5，那是**重大切换**，需另写方案并同步更新《运维学习计划.md》，不在本门课里顺手改。

- **验收**：node1-5 一台台自动装好、IP/主机名/角色全对，`ssh user@nodeN` 全部直达，与 8 台拓扑计划衔接
- **工具与方法**：
  - MAAS 在 playground 同网段起测试环境（或先看官方 demo，避免硬件门槛）
  - iPXE 菜单用脚本 + `choose` 命令；镜像切换用变量
  - 云厂商对照：腾讯云"自定义镜像"= 你做的定制 ISO/云镜像的托管版；AWS AMI = 模板镜像+cloud-init 二段式（本阶段讲清概念即可）
- **避坑经验**：
  - MAAS 与 dnsmasq **别抢 DHCP**，同网段只能有一个权威 DHCP
  - 加固项（禁用密码登录/关 sshd 之外的服务）会锁死自己 → 每步配好逃生 SSH key，先方案后执行
  - 批量装到一半失败 → 先看中央服务器的安装日志，别逐台登
  - 云镜像和 ISO 装的系统服务有细微差异（有无 cloud-init 预置、驱动），别假设两者完全等价
- **最佳实践**：
  - 本阶段的"角色矩阵+批量安装"，就是腾讯云模拟里 CVM 批量建机的本地复刻——一边学一边把你 AWS/腾讯云方案里的概念对上
  - 交付清单固定：主机名/IP/角色/软件/安全基线 一张表，装机即生成

### 阶段 7：专家方向（全部收录，学完前 6 阶段后任选 1-2 深耕）

> **够用即停**：对运维学习者，阶段6 结束已是"半专家 + 可交付"；阶段7 全是可选，按需挑 1 条深耕即可，不必为凑"专家"名头把 7.1-7.5 全学。

| # | 方向 | 主线内容 | 入门抓手 |
|---|---|---|---|
| 7.1 | **Subiquity 源码级** | 安装器源码、curtin internals、自定义安装步骤/安装器插件 | subiquity GitHub + autoinstall-reference |
| 7.2 | **大规模裸机** | MAAS 深入、IPMI/BMC、DHCP/TFTP/PXE 全链路故障排查、机架级交付 | MAAS 官方文档 + 真实排障复盘 |
| 7.3 | **镜像工厂** | CI/CD 流水线构建定制 ISO/云镜像、OEM 定制、多版本矩阵 | GitHub Actions + Cubic + 官方镜像工具 |
| 7.4 | **自动化交付** | autoinstall + Ansible + k8s 全自动交付流水线（装系统→配服务→入集群） | 对接你后续 k3s/TKE 模拟 |
| 7.5 | **商业与合规** | Ubuntu Pro/Auto、Landscape 大规模管理、合规审计 | 官方订阅/管理文档 |

- 7.1-7.4 纯 VMware 内可练；7.5 偏商业，需要时再看
- **进阶里程碑**：你的 autoinstall 配置库被复用给"另一套全新拓扑"只改变量就跑通

---

## 五、每课模板（照方法论）

```
## 第N课：主题
- 目标（一句话，学完能干嘛）
- 大白话概念（类比）
- 真实环境演示（命令全明文 + 输出全贴）
- 关键点/坑（必踩的）
- 练习（1-N 个，有检查标准）
- 验收（可测的通过标准）
- 与其他方案对比（固定尾部：Debian preseed / RHEL Kickstart / 云厂商镜像，代码块逐条列，每厂商一行完整显示）
```

**一课节奏（照带教规则）**：开场用"教学型 vibe 指令"大白话讲清概念 + 出一个练习 → 中间执行 + 调试循环（出错先贴报错原文 → 先解释 → 再修复，不偷偷修）→ 收尾 3 句话总结 + 出 3 道复述题（答不上明天重学）。

---

## 六、协作协议适配（自动安装特有）

- **高危边界**：动磁盘/格式化/加密=较高危，先给方案（含回滚/测试机专用）再执行；下载/改配置/装包=常规，直接干不啰嗦
- **命令全明文**：VM 新建/挂 ISO/开关机/快照也照旧先写后跑、输出全贴——你已有的铁律不因"自动安装"而豁免
- **3 次升级机制**：安装失败 1 次自看 subiquity/cloud-init 日志；2 次停猜，贴"报错原文 + 该份配置 + 安装日志"三件套；3 次降级半自动（给每步命令手动跑）
- **一次只破坏一个点**：一次只改一个键、只动一台测试机，坏了知道回哪个
- **复盘习惯**：每阶段写"问题→原因→解决"；装系统坑高度重复，复盘最值钱

**每日节奏（1 小时，照《运维学习计划》"每日实操节奏"）**：
- 前 10 分钟：教学型 vibe 指令开场，大白话讲今天概念 + 出一个练习
- 中间 40 分钟：执行练习；出错进调试循环（贴报错原文 → 先解释 → 再修复）
- 后 10 分钟：3 句话总结 + 3 道复述题，答不上明天重学
- 本方向一台装机 5-15 分钟，练习拆成"改配置 → 装 → 验证"小循环，不必一次装满

**弯路预警（本方向专属，照《运维学习计划》"常见弯路预警"）**：
- 沉迷调 yaml 语法、不练排障——排障（附录 B）才是这门课的价值
- 一口气改十个键再装——一次只改一个键，装炸了才知道回哪个
- 只装不验——装完必须 SSH + 查软件/分区，验收过了才进下一课
- 不敢删测试 VM——快照在手，装完即弃，不留垃圾
- 总想一步到位上 MAAS/PXE——没有前 4 阶段地基只会更懵

---

## 七、资源清单（2026-08 调研，保持最新）

**官方/一手（第一手册）**：
- [Autoinstall configuration reference manual（Subiquity 官方文档）](https://canonical-subiquity.readthedocs-hosted.com/en/latest/reference/autoinstall-reference.html)——所有键名的权威来源，全程随身
- [Providing autoinstall configuration（如何传配置给安装器）](https://canonical-subiquity.readthedocs-hosted.com/en/latest/tutorial/providing-autoinstall.html)
- [cloud-init 官方文档](https://cloudinit.readthedocs.io)——阶段2 主参考；`cloud-config` 模块参考是常查页
- [canonical/subiquity（GitHub）](https://github.com/canonical/subiquity)——安装器源码/Release 说明（26.04 默认 HWE/OEM 内核等变更在此看）
- [MAAS 官方文档](https://maas.io/docs)——阶段6
- [Ubuntu 26.04 LTS 中文指南（安装部分）](https://ubuntu.fan/docs/guide/installation/install-walkthrough)

**社区/工具**：
- [Cyclenerd/ubuntu-autoinstall（GitHub）](https://github.com/Cyclenerd/ubuntu-autoinstall)——生成塞配置的 ISO，阶段1/3 帮手
- [tai271828/autoinstall-desktop](https://github.com/tai271828/autoinstall-desktop)——桌面版无人值守镜像实战（含 26.04 桌面坑：自动登录/root 锁定/CIDATA 卷）
- [Cubic（自定义 ISO 工具）](https://github.com/PJ-Singh-001/Cubic)——阶段3
- [OSChina 批量部署实战（中文）](https://my.oschina.net/emacs_8004433/blog/19479997)——Autoinstall 批量服务器部署

**版本现状**：24.04 LTS（主力学习，2029 止安全更新）；26.04 LTS 已发布（2026-04-23，支持至 2031-04，Server 默认自动装 OEM/HWE 内核，桌面安装器改 Flutter）。**建议以 24.04 学、26.04 只记差异**，不两边同时铺开。

---

## 八、时间预估（每天 1-2 小时）

- 阶段 1-2（入门+cloud-init 引擎）：约 3 周
- 阶段 3-4（定制 ISO + PXE 分水岭）：约 4-5 周
- 阶段 5-6（工程化 + CI + 企业级/云对齐）：约 4-5 周
- **合计约 3 个月 → 能独立做"配置即代码"批量交付**（半专家）
- 阶段 7 专家方向：每条 1-3 个月起，长线

---

## 九、验收追踪表

| 阶段 | 主题 | 验收标准 | 状态 |
|---|---|---|---|
| 1 | 概念+首次无人值守 | 现有 server yaml 自动装出一台，SSH+docker+python，自动关机 | ⬜ |
| 2 | cloud-init 引擎 | 空 VM 开机全自动建用户/装软件/写文件，analyze 能分析 | ⬜ |
| 3 | 定制 ISO+离线 | 自定义 ISO 离线全自动装好+加密盘 | ⬜ |
| 4 | PXE/HTTP 批量 | 一台服务器 PXE 批量装 3 台不同角色 | ⬜ |
| 5 | 配置工程化 + CI | 一份模板改参数装出任一角色，secret 全变量，CI 校验自动过 | ⬜ |
| 6 | 企业级+云对齐 | 批量装出 node1-5 带角色，衔接 8 台拓扑 | ⬜ |
| 7 | 专家方向 | 任选 1-2 深耕 + 配置库换拓扑只改变量跑通 | ⬜ |

---

## 十、来源与沉淀

- 学习范式来源：《学习方法论与协作协议.md》
- 场地与资产：playground VM（2026-08-03 建）+ `user-data.yaml` / `user-data-server.yaml`
- 课程拆分原则：由易到难、不重复、每 5-8 课一阶段、阶段末综合实战、融入当前最新实践（26.04 差异已标注）
- 后续联动：本计划阶段4-6 直接服务"8 台拓扑"与"腾讯云模拟"（CVM 镜像/CLB/TKE=k3s），与 [[ops-learning-lab-plan]] 的运维七阶段衔接
- **学习/带教规则落地对照**：《学习方法论与协作协议.md》五条总原则 → 本文档各节（不重复=前提盘点；先方案后执行=六；每课验收=五+九）；教学范式 → 五、每课模板；协作协议 → 六；《运维学习计划.md》每日实操节奏 + 弯路预警 → 六 两个子节

---

## 附录 A：关键命令速查（全文明）

> 每课实际执行时，命令先写后跑、输出全贴，不缩略。

```bash
# 1) 下载官方 ISO（示例 24.04.3 server 版；下载慢/失败时自行加代理）
curl -L -o ubuntu-24.04.3-server-amd64.iso \
  https://releases.ubuntu.com/24.04.3/ubuntu-24.04.3-server-amd64.iso

# 2) 生成密码 hash（$6$...，SHA-512；把 YourPassHere 换成你的密码）
openssl passwd -6 'YourPassHere'

# 3) 校验 cloud-config/autoinstall 配置格式（喂给安装器前先验，省一次全装）
#    报错先确认文件首行是 #cloud-config
cloud-init schema --config-file user-data-server.yaml

# 4) 云镜像转 VMware vmdk（阶段4；qemu-utils 装在 playground 或宿主）
qemu-img convert -f qcow2 -O vmdk ubuntu-24.04-server-cloudimg-amd64.img \
  ubuntu-24.04-server-cloudimg-amd64.vmdk

# 5) 生成 NoCloud seed 盘（cloud-init 直装用，阶段2/4；user-data、meta-data 放同目录）
cloud-localds seed.img user-data meta-data

# 6) dnsmasq 一键 DHCP+TFTP（阶段4，仅限专用 VMnet）
sudo apt install -y dnsmasq
# /etc/dnsmasq.conf 最小配置：
#   dhcp-range=192.168.200.50,192.168.200.150,12h
#   dhcp-boot=grub/grubx64.efi   （EFI 引导）或 pxelinux.0（BIOS）
#   enable-tftp
#   tftp-root=/srv/tftp

# 7) HTTP 分发 autoinstall（阶段4；playground 可当服务器）
python3 -m http.server 8080 --directory /srv/autoinstall
```

---

## 附录 B：安装失败通用排障表（分层定位，遇问题按层查，别跳）

| 现象 | 定位层 | 第一动作 |
|---|---|---|
| 开机停在 GRUB / 不引导 | 介质/引导层 | 查 ISO 是否改坏（boot/grub、isolinux）；VMware 引导顺序 CD 优先 |
| 进了交互式安装器而不是自动装 | 配置传递层 | 查是否加 `autoinstall` 内核参数；user-data 是否在安装器能读到的地方 |
| 装了但分区/布局不对 | 存储层 | 回看 storage 段；subiquity 日志里的 storage 决策；每次只改这一个键 |
| 卡在"下载软件包" | 网络/源层 | 能否 apt 通；代理；离线源路径（阶段3）；换国内源 |
| 装完重启但 SSH 连不上 | cloud-init/网络层 | `cloud-init status`；netplan 是否固化；`allow-pw`/authorized-keys |
| 装完有系统但软件没装 | cloud-init/包层 | packages 是否被 vendor 覆盖；`cloud-init analyze blame`；日志里包安装段 |
| 桌面版装完没自动登录 / root 锁定 | 桌面专项层 | 阶段3 桌面课：dconf/gdm3 custom.conf、shadow 解锁、CIDATA 卷 |

**3 次升级**：失败 1 次自看对应层日志；2 次停猜，贴"报错原文 + 该份配置 + 安装/cloud-init 日志"三件套；3 次降级半自动（给每步命令手动跑）。

---

## 附录 C：VMware 环境适配

- **ISO 自动引导**：新建测试 VM 选"稍后安装操作系统"→ 挂 ISO → 首次开机 CD-ROM 优先；必要时 vmx 里加 `bios.bootOrder = "cdrom,hdd"`（BIOS 固件），EFI 用开机 F2/一次性启动菜单选光驱
- **PXE 专用网络（避免 DHCP 打架）**：VMware 编辑虚拟网络 → 新建/复用自定义 VMnet（如 VMnet3），**取消勾选"使用本地 DHCP 服务"**，测试机网卡选该 VMnet；dnsmasq 起在 playground（192.168.56.66）只管这个网段
- **云镜像（qcow2）**：VMware 不能直接引导 qcow2，一律 `qemu-img convert` 成 vmdk；cloud-init 数据用 `cloud-localds` 的 seed.iso 当第二光驱喂
- **快照纪律**：装系统虽比内核安全，仍是磁盘操作——每课实验前打快照、装炸了回滚；测试 VM 装完即弃，不留垃圾
- **磁盘预算**：单台安装测试 VM 分配 20G 虚拟盘、实际占用约 5-8G；阶段4 批量 3 台、阶段6 批量 5 台，注意 D: 盘余量（当前 146.9G，充足）
