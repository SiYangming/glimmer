# Glimmer 3.02 使用示例

Glimmer3 不是一步式流程，而是以「一组命令行程序」发布，标准链路为四步：

```text
long-orfs  →  extract  →  build-icm  →  glimmer3
(选训练ORF)  (提训练序列)   (建ICM模型)   (预测基因)
```

安装方式见 [INSTALL.md](INSTALL.md)，以下命令假定 `bin/` 与 `scripts/` 已在 `PATH` 中。

## 一、基本流程（单基因组）

### 1. 准备输入序列

`long-orfs` 以单条序列为输入。若基因组为多 contig，可先去表头合并成一整条：

```bash
mkdir -p path/to/glimmer_run
cd path/to/glimmer_run

# 单序列基因组可直接使用
cp path/to/genome.fasta genome.fna

# 多 contig 基因组：去掉 ">" 行后连成一整条
sed -e '/>/d' path/to/genome.fasta | tr -d '\n' | awk 'BEGIN{print ">genome"}{print}' > genome.fna
```

### 2. 四步链

```bash
# 第 1 步：挑选长且不重叠的 ORF 作为训练集
#   -n 输出不含表头；-t 1.15 为熵距离截断
long-orfs -n -t 1.15 genome.fna tag.longorfs

# 第 2 步：按坐标提取训练序列（-t 去掉终止密码子；多序列 FASTA 输出到 stdout）
extract -t genome.fna tag.longorfs > tag.train

# 第 3 步：从训练序列构建 ICM 模型（-r 用反向序列建模）
build-icm -r tag.icm < tag.train

# 第 4 步：用 ICM 预测基因，输出前缀为 tag
#   -o50 最大允许重叠长度；-g110 最小基因长度；-t30 判定为基因的分数阈值
glimmer3 -o50 -g110 -t30 genome.fna tag.icm tag
```

### 3. 结果说明

| 文件 | 说明 |
|------|------|
| `tag.predict` | 预测基因坐标，每行格式为 `<orfID> <start> <stop> <frame> <score>` |
| `tag.detail` | 各预测结果的打分细节 |

预测坐标为下游提供基因位置：可据此从基因组提取 CDS 并翻译蛋白。

## 二、批量处理多个基因组

```bash
cd path/to/glimmer_run

for fa in path/to/genomes/*.fasta; do
    name=$(basename "$fa" .fasta)
    sed -e '/>/d' "$fa" | tr -d '\n' | awk 'BEGIN{print ">genome"}{print}' > ${name}.fna
    long-orfs -n -t 1.15 ${name}.fna ${name}.longorfs
    extract -t ${name}.fna ${name}.longorfs > ${name}.train
    build-icm -r ${name}.icm < ${name}.train
    glimmer3 -o50 -g110 -t30 ${name}.fna ${name}.icm ${name}
done
```

每个基因组产出 `<name>.predict` 与 `<name>.detail`。

## 三、使用仓库内的样例运行结果

`sample-run/` 目录保存了一次完整流程的输入与产物，可用于对照检查自己的输出：

| 文件 | 说明 |
|------|------|
| `tpall.fna` | 输入基因组序列 |
| `from-scratch.longorfs` / `from-scratch.train` / `from-scratch.icm` / `from-scratch.predict` / `from-scratch.detail` | 从零开始训练并预测的四步产物 |
| `from-training.upstream` / `from-training.train` / `from-training.icm` / `from-training.motif` / `from-training.predict` / `from-training.detail` | 给定训练集后建模并预测的产物 |
| `iterated.*` | 迭代运行（含 `iterated.run1.*`）的各步产物 |
| `g3-from-scratch.csh` / `g3-from-training.csh` / `g3-iterated.csh` | 与上述产物对应的运行脚本 |

```bash
cd path/to/glimmer_run

# 查看预测结果格式
head path/to/glimmer/sample-run/from-scratch.predict
```

> `scripts/` 下的 `g3-from-scratch.csh`、`g3-from-training.csh`、`g3-iterated.csh` 在使用前需编辑脚本开头，填入本机 `bin` 与 `scripts` 目录的完整路径；其中 `g3-iterated.csh` 还会用到需另行获取的 `elph`。

## 四、参数说明

| 参数 | 所属程序 | 说明 |
|------|---------|------|
| `<sequence-file>` | long-orfs | 输入基因组 FASTA（单序列） |
| `-n` | long-orfs | 输出不含表头等说明 |
| `-t` | long-orfs | 熵距离截断（常用 1.15） |
| `-g` | long-orfs | 只考虑长度 ≥ n 的 ORF |
| `<sequence-file> <coords>` | extract | 基因组 + ORF 坐标文件，训练序列输出到 stdout |
| `-t` | extract | 去掉终止密码子 |
| `output_file` | build-icm | 输出 ICM 模型文件；训练序列由 stdin 读入 |
| `-r` | build-icm | 用反向序列构建模型 |
| `<sequence-file> <icm-file> <tag>` | glimmer3 | 基因组 + ICM 模型 + 输出前缀 |
| `-o` | glimmer3 | 最大允许重叠长度（常用 50） |
| `-g` | glimmer3 | 最小基因长度（常用 110） |
| `-t` | glimmer3 | 判定为基因的分数阈值（常用 30） |
| `-l` | long-orfs / glimmer3 | 假定线性基因组（不做环形 wraparound） |
| `-z` | long-orfs / glimmer3 | 终止密码子所用的 GenBank 翻译表编号（默认 11） |

> 注意：`long-orfs -t`（熵距离截断）与 `glimmer3 -t`（打分阈值）含义不同、取值量级不同。

各程序的完整选项可用 `-h` 查看：

```bash
long-orfs -h
glimmer3 -h
```

## 五、与其他基因预测工具的关系

Glimmer3 面向**原核生物**基因预测。真核/真菌基因组预测应使用同源不同物的 **GlimmerHMM**（基于广义隐马尔可夫模型）：

```bash
# 使用训练好的物种模型目录进行真核基因预测，-g 输出 GFF3
glimmerhmm genome.fasta -d path/to/trained_dir -g > glimmerhmm.gff
```

Glimmer3 亦常与 prodigal、GeneMark 并列为原核基因预测的对照工具。
