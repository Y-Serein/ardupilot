# ZHIER / ArduPilot

面向 ZHIER 飞控的 ArduPilot 定制工程，基于 ArduCopter 4.5.7，维护板级配置、默认参数，以及既有 OpenMV 和自定义飞行模式。

本仓库使用标准 MAVLink 协议，系统 ID 为 8 位。固件构建使用本仓库记录的固定依赖版本。

- 项目仓库：<https://github.com/Y-Serein/ardupilot>
- 上游项目：<https://github.com/ArduPilot/ardupilot>
- 构建与测试：<https://github.com/Y-Serein/ardupilot/actions>
- 主线：`main`；历史节点通过 `archive/*` 标签追溯。

## 维护范围

| 项目 | 内容 |
| --- | --- |
| 主线 | `main`；历史分支以 `archive/*` 标签保留 |
| 整理基线 | `APM_Swarm-1.0.5`，提交 `27cb610d15b0d196af6bdd61a0ae571120e4b637` |
| 固件版本 | 源码声明为 ArduCopter 4.5.7；旧分支名 `1.0.5` 不是上游固件版本 |
| 目标硬件 | ZHIER；定义位于 `libraries/AP_HAL_ChibiOS/hwdef/ZHIER/` |
| 仿真目标 | Copter SITL |
| 构建依赖 | 以主仓库及嵌套子模块记录的提交为准，不跟随依赖分支的最新版本 |

编译成功仅证明对应目标可构建。板级默认参数包含既有机型设置，使用固件前仍需核对实际硬件和参数；本次项目整理没有重新验证真实硬件或飞行行为。

## 获取源码

```bash
git clone --recurse-submodules --branch main https://github.com/Y-Serein/ardupilot.git
cd ardupilot
```

在干净的新克隆中，如子模块尚未下载，可运行：

```bash
git submodule update --init --recursive
```

已有工作区先检查主仓库及子模块的未提交修改，不通过重置或更新依赖覆盖本地工作。

## 原生构建

Linux / WSL 构建需要 Python 3、ArduPilot 对应的 Python 依赖，以及 ZHIER 使用的 GCC Arm Embedded 10 工具链。CI 沿用 ArduPilot 的 `v0.1.3` 构建镜像；本地环境安装参考[上游构建说明](https://ardupilot.org/dev/docs/building-the-code.html)。

下面直接调用项目自带 Waf。`python3 waf` 与原有 Waf 入口相同，也能避免 Windows 检出文件的换行符影响脚本首行。产品编译关闭测试，三类构建使用独立目录。

### ZHIER 主固件

```bash
python3 waf configure --board ZHIER --out=build/firmware --no-submodule-update --disable-tests --default-parameters=libraries/AP_HAL_ChibiOS/hwdef/ZHIER/defaults.parm
python3 waf copter
```

产物：`build/firmware/ZHIER/bin/arducopter.bin`；同目录另有 Waf 正常生成的 `.apj` 等文件。

### ZHIER Bootloader

```bash
python3 waf configure --board ZHIER --bootloader --out=build/bootloader --no-submodule-update --disable-tests
python3 waf bootloader
```

产物：`build/bootloader/ZHIER/bin/AP_Bootloader.bin`。

这条命令只编译，不覆盖 `Tools/bootloaders/ZHIER_bl.bin`。带 Bootloader 的 `.hex` 仍使用该归档文件；本项目日常上传的产物是分开的两个 bin。

### Copter SITL

```bash
python3 waf configure --board sitl --out=build/sitl-product --no-submodule-update --disable-tests
python3 waf copter
```

产物：`build/sitl-product/sitl/bin/arducopter`。编译不会自动启动仿真。

## 独立测试

通用单元测试使用独立目录和 GitHub 手动工作流 `Copter unit tests`，不作为产品编译或固件上传的前置条件：

```bash
python3 waf configure --board sitl --out=build/unit-tests --no-submodule-update --debug
python3 waf tests
```

运行需要的测试可执行文件，例如：

```bash
build/unit-tests/sitl/tests/test_math
```

GitHub 工作流会运行该目录的全部测试可执行文件，分别记录编译及执行结果。通用测试覆盖软件逻辑，硬件与飞行验收单独进行。

## GitHub Actions 与固件下载

- `ZHIER firmware`：`main` 的代码推送、面向 `main` 的 PR 或手动触发；分别编译主固件与 Bootloader。
- `Copter SITL build`：同样的触发方式，只编译 SITL。
- `Copter unit tests`：手动触发，独立编译并运行通用测试。
- 继承的其他平台工作流保留手动入口，关闭推送、PR 和定时自动触发；未将这些旧平台列为本项目已验证目标。

在 [Actions](https://github.com/Y-Serein/ardupilot/actions) 中进入成功的 `ZHIER firmware` 运行，从 **Artifacts** 下载包含提交 SHA 和运行编号的产物包。包内只保存同次构建的 `arducopter.bin`、`AP_Bootloader.bin`，保留 **30 天**；提交、编译器、子模块版本和两个文件的 SHA256 写入该次运行摘要。

每天两至三次构建、每次合计不超过 4 MB，30 天约占 240–360 MB。账号下其他构建产物和 Packages 仍需计入共享额度。没有配置固定每日构建，也不会自动创建 Release、上传设备或刷写固件。

## 版本管理

日常修改围绕 `main`，确有需要时建立短期功能分支，合入后清理。历史节点用标签定位，正式发布时再创建版本标签与 Release；不改写上游历史，不把旧分支名称当作验证等级。

`archive/APM_Swarm-1.0.4` 保留的 OpenMV/降落及参数编号变更没有合入当前基线；`archive/APM_Swarm-0.0.1` 保留早期高度消息和仿真位置记录。需要恢复这些功能时单独审查和验证。

## 许可证与上游归属

本项目基于 ArduPilot，沿用 [GNU GPL v3](COPYING.txt)。原始版权声明、作者信息、源码历史和许可证保持不变。ArduPilot 的上游平台能力及官方测试结果不代表本定制版本已经通过相同验收。
