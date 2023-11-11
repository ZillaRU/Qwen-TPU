![](./assets/tpumlir.png)

# Qwen-TPU

本工程实现BM1684X部署语言大模型[Qwen-7B-Chat](https://huggingface.co/Qwen/Qwen-7B-Chat)。通过[TPU-MLIR](https://github.com/sophgo/tpu-mlir)编译器将模型转换成bmodel，并采用c++代码将其部署到BM1684X的PCIE环境，或者SoC环境。



## 开发环境准备

### 1. 下载`Qwen-7B-Chat`

``` shell
git lfs install
git git@hf.co:Qwen/Qwen-7B-Chat
```

该工程比较大，会花较长时间。并对该工程做如下修改。

#### 调整`config.json`文件中参数配置

```json
  "bf16": true,
  "max_position_embeddings": 512,
  "seq_length": 512,
```

我们采用bf16导出ONNX模型，原因是该模型是通过bf16训练的。用F32也可以，但是这样ONNX体积太大。

#### 对`modeling_qwen.py`文件代码做调整

1) 第一点修改如下（这是因为TORCH2的算子转ONNX会失败）：

    ``` python
    # SUPPORT_TORCH2 = hasattr(torch, '__version__') and int(torch.__version__.split(".")[0]) >= 2
    SUPPORT_TORCH2 = False
    ```

2) 第二点修改如下（这是因为转ONNX，提示Shape推导失败）：

    ```python
    # attn_weights = attn_weights / torch.full(
    #     [],
    #     size_temp ** 0.5,
    #     dtype=attn_weights.dtype,
    #     device=attn_weights.device,
    # )
    attn_weights = attn_weights / (size_temp ** 0.5)
    ```

3) 第三点修改如下（这段代码全部注释掉，是因为可以直接采用`attention_mask`，避免复杂逻辑，提升性能）：

    ```python
    # if self.use_cache_quantization:
    #     query_length, key_length = query.size(-2), key[0].size(-2)
    # else:
    #     query_length, key_length = query.size(-2), key.size(-2)
    # causal_mask = registered_causal_mask[
    #     :, :, key_length - query_length : key_length, :key_length
    # ]
    # mask_value = torch.finfo(attn_weights.dtype).min
    # mask_value = torch.full([], mask_value, dtype=attn_weights.dtype).to(
    #     attn_weights.device
    # )
    # attn_weights = torch.where(
    #     causal_mask, attn_weights.to(attn_weights.dtype), mask_value
    # )
    ```

4) 第四点修改如下（同上原因）：

    ``` python
    # query_length, key_length = query.size(-2), key.size(-2)
    # causal_mask = registered_causal_mask[
    #     :, :, key_length - query_length : key_length, :key_length
    # ]
    # mask_value = torch.finfo(attn_weights.dtype).min
    # mask_value = torch.tensor(mask_value, dtype=attn_weights.dtype).to(
    #     attn_weights.device
    # )
    # attn_weights = torch.where(causal_mask, attn_weights, mask_value)
    ```

### 2. 下载本项目`Qwen-TPU`

下载本项目，并导出所有的ONNX，如下：

``` shell
git clone git@github.com:sophgo/Qwen-TPU.git
git submodule update --init

cd Qwen-TPU/compile
pip install transformers_stream_generator einops tiktoken
export PYTHONPATH=$PWD/../Qwen-7B-Chat:$PYTHONPATH
python3 export_onnx.py
```

因为我们采用BF16格式导出ONNX，需要您的环境上带有CUDA。默认x86不支持BF16。

### 3. 下载docker，启动容器

``` shell
docker pull sophgo/tpuc_dev:latest

# myname1234 is just an example, you can set your own name
docker run --privileged --name myname1234 -v $PWD:/workspace -it sophgo/tpuc_dev:latest
```
后文假定环境都在docker的`/workspace`目录。

### 4. 下载`TPU-MLIR`代码并编译

(也可以直接下载编译好的release包解压)

``` shell
git clone git@github.com:sophgo/tpu-mlir.git
cd tpu-mlir
source ./envsetup.sh
./build.sh
```

## 编译模型

注意此时在Docker环境workspace目录。

目前TPU-MLIR支持对`Qwen-7B`进行BF16、INT8和INT4量化，且支持多芯分布式推理，默认情况下会进行INT8量化和单芯推理，最终生成`qwen-7b_int8.bmodel`文件

```shell
./compile.sh
```

若想进行2芯推理，则执行以下命令，最终生成`qwen-7b_int8_2dev.bmodel`文件，4芯8芯同理：

```shell
./compile.sh --num_device 2
```

## 编译程序(C++版本)

执行如下编译 (注意如果是SoC版本，需要把demo目录拷贝到SoC环境编译)：

```shell
cd Qwen-TPU/demo
mkdir build
cd build
cmake ..
make
```


编译生成qwen可执行程序，将`qwen`、`qwen-7b_int8.bmodel`和`qwen.tiktoken`拷贝到同一个目录下就可以执行了。
(`qwen.tiktoken`来自[Qwen-7B-Chat](https://huggingface.co/Qwen/Qwen-7B-Chat))。

运行`qwen`:
```shell
./qwen --model qwen-7b_int8.bmodel
```

如果是2芯分布式推理，使用如下命令(比如指定在2号和3号芯片上运行, 用`bm-smi`查询芯片id号)：
```shell
./qwen --model qwen-7b_int8_2dev.bmodel --devid 2,3
```

## 运行效果

以下为单芯片下INT8量化模式的运行效果：

![](./assets/qwen.jpg)

## 常见问题

### demo程序无法正常运行

如果demo程序拷贝到运行环境提示无法运行，比如接口找不到等等错误。
原因是运行环境的库有所不同，将demo中的`lib_pcie`（PCIE）或者 `lib_soc`(SoC)里面的so文件拷贝到运行环境，链接到里面的so即可。

### tiktoken是如何用C++支持的

tiktoken官方没有C++版本，只有python版本。
本工程使用[QwenLM/qwen.cpp](https://github.com/QwenLM/qwen.cpp)中的tiktoken处理代码。

