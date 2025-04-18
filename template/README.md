# LLM Template

## 目录
- [LLM Template](#llm-template)
  - [目录](#目录)
  - [1. 简介](#1-简介)
  - [2. 特性](#2-特性)
  - [3. 目录结构](#3-目录结构)
  - [4. 模型编译](#4-模型编译)
    - [4.1 环境准备](#41-环境准备)
    - [4.2 模型转换与编译](#42-模型转换与编译)
  - [5. 运行](#5-运行)
    - [5.1 环境准备](#51-环境准备)
    - [5.2 模型下载与运行](#52-模型下载与运行)
    - [5.3 多芯模型下载与运行](#53-多芯模型下载与运行)
  - [6. 程序性能测试](#6-程序性能测试)

## 1. 简介
本仓库提供了一个通用的大语言模型（Large Language Model）例程，支持一键导出onnx并通过[TPU-MLIR](https://github.com/sophgo/tpu-mlir)编译器将模型转换成bmodel，支持在SOPHON BM1684X、BM1688/CV186X上进行推理测试。该例程适用于多种语言模型，能够处理文本生成、文本理解等任务。

## 2. 特性
* 支持BM1684X、BM1688(x86/arm PCIe、SE7、SE5)
* 支持INT8、INT4模型编译和推理
* 支持Python例程

## 3. 目录结构
```bash

├── README.md                     # 例程指南
├── compile                       # 模型导出目录
│   ├── model_export.py           # bmodel编译脚本
│   └── onnx_rebuilder.py         # onnx导出脚本
├── demo                          # 单芯demo
│   ├── chat.cpp                  # python依赖的后端cpp文件
│   ├── pipeline.py               # python_demo的执行脚本
│   └── CMakeLists.txt
├── parallel_demo                 # 多芯demo
│   ├── chat.cpp                  # python依赖的后端cpp文件
│   ├── pipeline.py               # python_demo的执行脚本
│   └── CMakeLists.txt
```

## 4. 模型编译

### 4.1 环境准备
* 如果不想编译模型，也可以直接跳转至[5. 运行](#5-运行)测试我们编译好的模型。

模型编译的流程是将llm原始权重转换为onnx，再通过[TPU-MLIR](https://github.com/sophgo/tpu-mlir)编译器将模型转换成bmodel，需要在x86主机上安装TPU-MLIR环境，x86主机已安装Ubuntu16.04/18.04/20.04系统，并且运行内存在12GB以上。
具体环境配置请参考：[MLIR环境安装指南](https://github.com/sophgo/LLM-TPU/blob/main/docs/Mlir_Install_Guide.md)

### 4.2 模型转换与编译

model_export.py是一个通用的llm模型导出工具，能够直接将llm原始权重导出为onnx和bmodel。

``` shell
cd compile
python model_export.py -m $your_model_path -t $your_mlir_path -s $seq_length -q $quantize_type
```
主要参数：
* -m, --model_path     原始权重路径，如 ./Qwen2-7B-Instruct
* -t, --tpu_mlir_path  MLIR环境路径
* -s, --seq_length     LLM sequence长度
* -q, --quantize       Bmodel的量化类型，目前支持：bf16,w8bf16,w4bf16,f16,w8f16,w4f16

比如，导出4k长度的int4量化的Qwen2-7B-Instruct：
``` shell
cd compile
python model_export.py -m ./Qwen2-7B-Instruct -t /workspace/tpu-mlir -s 4096 -q w4bf16
```

导出1k长度的int4量化的QWQ-32B：
``` shell
python model_export.py --quantize w4bf16 --tpu_mlir_path /workspace/tpu-mlir/ --model_path /workspace/models/QWQ-32B/ --seq_length 1024 --out_dir qwq_32b --num_device 2
```

* 注意，model_export.py通常对transformers的版本没有特殊要求，但需要大于在原始权重config.json中标注的版本，比如QWQ-32B的config.json：`"transformers_version": "4.43.1"`

此外还有其他参数：
| **选项**               | **是否需要参数** | **默认值**       | **描述**                                                                 |
|------------------------|------------------|------------------|--------------------------------------------------------------------------|
| `-c`, `--chip`         | 是               | `bm1684x`        | 芯片类型，可选：`bm1688`, `cv186x`                                      |
| `-g`, `--q_group_size` | 是               | 64               | w4a16量化时指定`group_size`大小                                       |
| `--high_precision`     | 否               | 无               | 是否使用高精度量化                                                    |
| `--num_device`         | 是               | 无               | 芯片设备数量，用于导出多芯模型                                          |
| `--not_compile`        | 否               | 无               | 仅导出 ONNX，不编译 BMODEL                                              |
| `--embedding_disk`     | 否               | 无               | 将 embedding 存储为 bin 文件，通过 CPU 推理                             |
| `--compile_mode`       | 是               | `normal`         | 模型转换模式，可选：`fast`, `normal`                                      |
| `--out_dir`            | 是               | `./tmp`          | 输出路径                                                                |
| `--out_bmodel`         | 是               | 模型配置自动生成  | 输出 BMODEL 的名称                                                      |
| `--visual_length`      | 是               | 无               | 视觉长度（Vision Length），用于导出多模态模型                           |
| `--max_workers`        | 是               | `3`              | 编译 BMODEL 时的最大线程数                                              |

## 5. 运行

### 5.1 环境准备
* 芯片运行环境安装请参考[算能官网](https://developer.sophgo.com/site/index/material/all/all.html)

安装LLM-TPU相关依赖库
```bash
git clone https://github.com/sophgo/LLM-TPU.git
pip3 install dfss transformers==4.45.1 pybind11[global] Jinja2
sudo apt install zip
```

编译c++依赖
```bash
cd demo && mkdir build
cd build && cmake .. && make && cp *cpython* .. && cd ..
```

多模态模型（例如Qwen2-VL、Qwen2.5-VL），编译c++依赖
```bash
# 安装OpenCv依赖
sudo apt update
sudo apt install libopencv-dev

cd demo && mkdir build
cd build && cmake -DTYPE=media .. && make && cp *cpython* .. && cd ..
```

多芯模型（例如QWQ-32B），编译c++依赖
```bash
cd parallel_demo && mkdir build
cd build && cmake .. && make && cp *cpython* .. && cd ..
```

### 5.2 模型下载与运行

完成模型编译后，模型相关的推理文件和分词器相关配置，通常都保存在同一目录下，确认其中包含了bmodel文件、config文件，和tokenizer目录，就可以直接运行了：

```bash
cd demo
python3 pipeline.py --devid your_dev_id --dir_path your_model_path
```

* 如果下载的是单独的bmodel文件，可以手动将bmodel文件、原始权重的config文件、和tokenizer或processor(用于多模态模型)放在同一目录下再运行。

#### DeepSeek-R1-Distill-Qwen系列
下载`deepseek-r1-distill-qwen`系列模型，并运行：

```bash
# deepseek-r1-distill-qwen-1.5b
python3 -m dfss --url=open@sophgo.com:/ext_model_information/LLM/LLM-TPU/deepseek-r1-distill-qwen-1-5b.zip
unzip deepseek-r1-distill-qwen-1-5b.zip
python3 pipeline.py --devid 0 --dir_path ./deepseek-r1-distill-qwen-1-5b/

# deepseek-r1-distill-qwen-7b
python3 -m dfss --url=open@sophgo.com:/ext_model_information/LLM/LLM-TPU/deepseek-r1-distill-qwen-7b.zip
unzip deepseek-r1-distill-qwen-7b.zip
python3 pipeline.py --devid 0 --dir_path ./deepseek-r1-distill-qwen-7b/

# deepseek-r1-distill-qwen-14b
python3 -m dfss --url=open@sophgo.com:/ext_model_information/LLM/LLM-TPU/deepseek-r1-distill-qwen-14b-seq512.zip
unzip deepseek-r1-distill-qwen-14b-seq512.zip
python3 pipeline.py --devid 0 --dir_path ./deepseek-r1-distill-qwen-14b/
```

### 5.3 多芯模型下载与运行

```bash
cd parallel_demo
python3 pipeline.py --devid your_dev_id --dir_path your_model_path
```

#### QWQ-32B
下载`qwq-32b`两芯模型，并运行：
```bash
python3 -m dfss --url=open@sophgo.com:/ext_model_information/LLM/LLM-TPU/qwq_32b_w4bf16_seq1024_2dev.tar.gz
tar -zxvf qwq_32b_w4bf16_seq1024_2dev.tar.gz
python3 pipeline.py --devid 0,1 --dir_path ./qwq_32b_w4bf16_seq1024_2dev
```

#### DeepSeek-R1-Distill-Qwen-32B

下载`DeepSeek-R1-Distill-Qwen-32B`四芯模型，并运行：
```bash
# 2048长度
python3 -m dfss --url=open@sophgo.com:/ext_model_information/LLM/LLM-TPU/deepseek-r1-distill-qwen-32b-seq2048-4dev.zip
unzip deepseek-r1-distill-qwen-32b-seq2048-4dev.zip
python3 pipeline.py --devid 0,1,2,3 --dir_path deepseek-r1-distill-qwen-32b-seq2048-4dev


# 4096长度
python3 -m dfss --url=open@sophgo.com:/ext_model_information/LLM/LLM-TPU/deepseek-r1-distill-qwen-32b-seq4096-4dev.zip
unzip deepseek-r1-distill-qwen-32b-seq4096-4dev.zip
python3 pipeline.py --devid 0,1,2,3 --dir_path deepseek-r1-distill-qwen-32b-seq4096-4dev
```

## 6. 程序性能测试
我们测试了不同大小的deepseek-r1蒸馏模型在SE7-32(单芯1684x)上的性能，具体如下：

|   测试平台   |           测试模型              | 量化方式 | 模型长度 | first token latency(s) | token per second(tokens/s) |
| ----------- | ------------------------------ | -------- | -------- | --------------------- | -------------------------- |
| SE7-32      | deepseek-r1-distill-qwen-1.5b  | INT4     | 8192     | 5.431                 | 28.878                     |
| SE7-32      | deepseek-r1-distill-qwen-7b    | INT4     | 2048     | 2.939                 | 10.600                     |
| SE7-32      | deepseek-r1-distill-qwen-14b   | INT4     | 512      | 1.400                 | 5.564                      |

> **测试说明**：  
> 1. 性能测试结果具有一定的波动性，建议多次测试取平均值；
> 2. SE7-32的主控处理器为8核 ARM A53 42320 DMIPS @2.3GHz，PCIe上的性能由于处理器的不同可能存在较大差异；
> 3. 这里使用的SDK版本是BM1684X V24.04.01；

## 7. 常见问题及解答

请参考[LLM-TPU常见问题及解答](../../../docs/FAQ.md)