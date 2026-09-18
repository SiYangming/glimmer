# Glimmer 3.02 安装与配置

## 软件简介

Glimmer（**G**ene **L**ocator and **I**nterpolated **M**arkov **M**odelER，别名 `glimmer3`）是原核生物基因预测系统：先在基因组中挑选长 ORF 构建插值上下文模型（ICM），再依据 ICM 预测编码基因，输出每条基因的坐标与打分。

本仓库为 Glimmer 3.02 源码归档，根目录的 `README` 为上游发行说明，完整说明见根目录 `glim302notes.pdf`。本文件说明安装与配置方式，使用示例见 [Example.md](Example.md)。

## 版本与平台

| 项目 | 说明 |
|------|------|
| 版本 | 3.02（上游最终版本，2013 年起停止更新） |
| 平台 | Linux / macOS（源码编译） |
| 许可 | Artistic License，见根目录 `LICENSE` |
| 上游官网 | http://ccb.jhu.edu/software/glimmer/ |

## 依赖

- GNU make
- C++ 编译器（`g++` 或 `clang++`）
- `elph`（可选）：仅 `scripts/g3-iterated.csh` 等脚本需要，需另行获取

## 仓库结构

```text
glimmer/
├── README                                  # 上游发行说明
├── LICENSE                                 # Artistic License
├── glim302notes.pdf                        # 上游完整说明文档
├── Allow-glimmer-to-compile-on-g-4.4.3.patch  # 旧版编译器兼容补丁
├── src/                                    # 上游源码目录（含 Makefile 与编译规则）
│   ├── Makefile  c_make.gen  c_make.glm
│   ├── Common/  Glimmer/  ICM/  Util/
├── SimpleMake/                             # 简化 Makefile + 扁平化源码，产出到 bin/
├── scripts/                                # 运行脚本（g3-from-scratch.csh 等）
├── sample-run/                             # 一次完整运行的输入与产物示例
├── docs/                                   # 上游说明文档源码（notes.tex/notes.pdf）
├── bin/  lib/  obj/                        # 编译产物目录（初始为空）
```

## 安装

### 方式一：使用简化 Makefile（推荐）

仓库根目录的 `SimpleMake/` 提供了扁平的 Makefile，源码集中在该目录，编译产物输出到 `bin/`、`obj/`、`lib/`：

```bash
# 在克隆/解压后的仓库根目录执行
cd SimpleMake
make -j 4
cd ..
```

编译完成后 `bin/` 下会生成 14 个可执行程序：`glimmer3`、`long-orfs`、`build-icm`、`extract`、`multi-extract`、`anomaly`、`test`、`entropy-profile`、`entropy-score`、`start-codon-distrib`、`uncovered`、`window-acgt`、`build-fixed`、`score-fixed`。

### 方式二：使用上游 src/ 结构

按上游 README 的说明进入 `src` 目录编译：

```bash
cd src
make -j 4
cd ..
```

若在现代编译器中报错（例如与 `char16_t` 声明相关的错误，源于编译规则中写死的 `CXXDEFS = -D__cplusplus`），可在命令行覆盖该变量后再编译：

```bash
cd src
make CXXDEFS= -j 4
cd ..
```

仓库根目录另附有旧版编译器的兼容补丁 `Allow-glimmer-to-compile-on-g-4.4.3.patch`，按需使用。

### 方式三：Conda 安装

```bash
mamba create -n glimmer -c conda-forge -c bioconda glimmer=3.02
conda activate glimmer
glimmer3            # 无参数时打印 usage
long-orfs           # 无参数时打印 usage
```

### 方式四：官方容器

```bash
docker pull quay.io/biocontainers/glimmer:3.02--h87f3376_6

# 必须 -u $(id -u):$(id -g) 挂载宿主用户，否则产物归 root
docker run --rm -u $(id -u):$(id -g) -v "$PWD":/data -w /data \
    quay.io/biocontainers/glimmer:3.02--h87f3376_6 \
    long-orfs -n -t 1.15 /data/genome.fna /data/tag.longorfs
```

> `build-icm` 从标准输入读取训练序列，容器内需自行重定向，例如
> `docker run --rm -i -u $(id -u):$(id -g) -v "$PWD":/data -w /data quay.io/biocontainers/glimmer:3.02--h87f3376_6 build-icm -r /data/tag.icm < tag.train`。

## 配置

### 加入 PATH

源码编译完成后，把 `bin/` 与 `scripts/` 加入 `PATH`：

```bash
# 在仓库根目录执行，或替换为实际的绝对路径
echo "export PATH=$PWD/bin:$PWD/scripts:\$PATH" >> ~/.bashrc
source ~/.bashrc
```

### 自检

Glimmer 各程序没有 `--version` 选项，程序在 `PATH` 中且能打印用法即安装成功：

```bash
glimmer3  2>&1 | head -2     # 打印 "USAGE: glimmer3 ..."
long-orfs 2>&1 | head -2     # 打印 "USAGE: long-orfs ..."
```

### 运行脚本配置

`scripts/` 下的脚本（如 `g3-from-scratch.csh`、`g3-from-training.csh`、`g3-iterated.csh`）在使用前需要编辑脚本开头部分，填入本机 `bin` 与 `scripts` 目录的完整路径，详见根目录 `README`。
