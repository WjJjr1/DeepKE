# InstructionKGC-指令驱动的自适应知识图谱构建

<p align="left">
    <b> <a href="https://github.com/zjunlp/DeepKE/tree/main/example/llm/InstructKGC/README.md">English</a> | 简体中文 </b>
</p>


- [InstructionKGC-指令驱动的自适应知识图谱构建](#instructionkgc-指令驱动的自适应知识图谱构建)
  - [1.任务目标](#1任务目标)
  - [2.数据](#2数据)
  - [3.准备](#3准备)
    - [环境](#环境)
    - [下载数据](#下载数据)
    - [模型](#模型)
  - [4.LoRA微调](#4lora微调)
    - [LoRA微调LLaMA](#lora微调llama)
    - [LoRA微调Alpaca](#lora微调alpaca)
    - [LoRA微调智析](#lora微调智析)
    - [LoRA微调Vicuna](#lora微调vicuna)
    - [LoRA微调ChatGLM](#lora微调chatglm)
    - [LoRA微调Moss](#lora微调moss)
    - [LoRA微调Baichuan](#lora微调baichuan)
  - [5.P-Tuning微调](#5p-tuning微调)
    - [P-Tuning微调ChatGLM](#p-tuning微调chatglm)
  - [6.预测](#6预测)
      - [LoRA预测](#lora预测)
    - [P-Tuning预测](#p-tuning预测)
  - [7.模型输出转换\&计算F1](#7模型输出转换计算f1)
  - [8.Acknowledgment](#8acknowledgment)
  - [Citation](#citation)


## 1.任务目标

根据用户输入的指令抽取相应类型的实体和关系，构建知识图谱。其中可能包含知识图谱补全任务，即任务需要模型在抽取实体关系三元组的同时对缺失三元组进行补全。

以下是一个**知识图谱构建任务**例子，输入一段文本`input`和`instruction`（包括想要抽取的实体类型和关系类型），以`(ent1,rel,ent2)`的形式输出`input`中包含的所有关系三元组`output`：

```python
instruction="使用自然语言抽取三元组,已知下列句子,请从句子中抽取出可能的实体、关系,抽取实体类型为{'专业','时间','人类','组织','地理地区','事件'},关系类型为{'体育运动','包含行政领土','参加','国家','邦交国','夺得','举办地点','属于','获奖'},你可以先识别出实体再判断实体之间的关系,以(头实体,关系,尾实体)的形式回答"
input="2006年，弗雷泽出战中国天津举行的女子水球世界杯，协助国家队夺得冠军。2008年，弗雷泽代表澳大利亚参加北京奥运会女子水球比赛，赢得铜牌。"
output="(弗雷泽,获奖,铜牌)(女子水球世界杯,举办地点,天津)(弗雷泽,属于,国家队)(弗雷泽,国家,澳大利亚)(弗雷泽,参加,北京奥运会女子水球比赛)(中国,包含行政领土,天津)(中国,邦交国,澳大利亚)(北京奥运会女子水球比赛,举办地点,北京)(女子水球世界杯,体育运动,水球)(国家队,夺得,冠军)"
```


## 2.数据

模型的输入需要包含`instruction`和`input`(可选)字段, 我们在 [kg2instruction/convert.py](./kg2instruction/convert.py)、[kg2instruction/convert_test.py](./kg2instruction/convert_test.py) 提供了一个脚本，用于将数据统一转换为可以直接输入模型的格式。

* 请注意！在执行 [kg2instruction/convert.py](./kg2instruction/convert.py) 之前，请参考 [data](./data) 目录, 其中包含了每种任务的预期数据格式。


```bash
python kg2instruction/convert.py \
  --src_path data/NER/sample.json \
  --tgt_path data/NER/processed.json \
  --schema_path data/NER/schema.json \
  --language zh \       # 不同语言使用的template及转换脚本不同
  --task NER \          # ['RE', 'NER', 'EE']三种任务
  --sample 0 \          # 若为-1, 则从4种指令和4种输出格式中随机采样其中一种, 否则即为指定的指令格式, -1<=sample<=3
  --all                 # 是否将指令中指定的抽取类型列表设置为全部schema
```

[kg2instruction/convert_test.py](./kg2instruction/convert_test.py) 不要求数据具有标签(`entity`、`relation`、`event`)字段, 只需要具有 `input` 字段, 以及提供 `schema_path`, 适合用来处理测试数据。

```bash
python kg2instruction/convert_test.py \
    --src_path data/NER/sample.json \
    --tgt_path data/NER/processed.json \
    --schema_path data/NER/schema.json \
    --language zh \      
    --task NER \          
    --sample 0 
```


* 更详细的关于IE模版、数据格式转换、数据提取请参考[kg2instruction/README_CN.md](./kg2instruction/README_CN.md)

* 您也可以自行构建包含`instruction`和`input`字段的数据

下面是一些现成的处理后的数据：

| 名称                  | 下载                                                                                                                     | 数量     | 描述                                                                                                                                                       |
| ------------------- | ---------------------------------------------------------------------------------------------------------------------- | ------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| KnowLM-IE.json       | [Google drive](https://drive.google.com/file/d/1hY_R6aFgW4Ga7zo41VpOVOShbTgBqBbL/view?usp=sharing) <br/> [HuggingFace](https://huggingface.co/datasets/zjunlp/KnowLM-IE)      | 281860 | [InstructIE](https://arxiv.org/abs/2305.11527) 中提到的数据集                                                                                     |
| train.json, valid.json          | [Google drive](https://drive.google.com/file/d/1vfD4xgToVbCrFP2q-SD7iuRT2KWubIv9/view?usp=sharing)                     | 5000   | [CCKS2023 开放环境下的知识图谱构建与补全评测任务一：指令驱动的自适应知识图谱构建](https://tianchi.aliyun.com/competition/entrance/532080/introduction) 中的初赛训练集及测试集，从 KnowLM-IE.json 中随机选出的 |


`KnowLM-IE.json`：包含 `'id'`(唯一标识符)、`'cate'`(文本主题)、`'instruction'`(抽取指令)、`'input'`(输入文本)、`'output'`(输出文本)字段、`'relation'`(三元组)字段，可以通过`'relation'`自由构建抽取的指令和输出，`'instruction'`有16种格式(4种prompt * 4种输出格式)，`'output'`是按照`'instruction'`中指定的输出格式生成的文本。。


`train.json`：字段含义同`train.json`，`'instruction'`、`'output'`都只有1种格式，也可以通过`'relation'`自由构建抽取的指令和输出。

`valid.json`：字段含义同`train.json`，但是经过众包标注，更加准确。


以下是各字段的说明：

|    字段     |                          说明                          |
| :---------: | :----------------------------------------------------: |
|     id      |                     唯一标识符                     |
|    cate     |     文本input对应的主题(共12种)                          |
|    input    |    模型输入文本（需要抽取其中涉及的所有关系三元组）    |
| instruction |                 模型进行抽取任务的指令                 |
| output      | 模型期望输出 |
| relation    |                  input中涉及的关系三元组(head, relation, tail)               |


此处 [schema](./kg2instruction/schema.py) 提供了12种文本主题, 以及该主题下常见的关系类型。

## 3.准备
### 环境
请参考[DeepKE/example/llm/README_CN.md](../README_CN.md/#环境依赖)创建python虚拟环境, 然后激活该环境 `deepke-llm`:

```bash
conda activate deepke-llm
```

！！！注意，为适配`qlora`技术，我们升高了原deepke-llm中`transformers`、`accelerate`、`bitsandbytes`、`peft`库的版本

1. transformers 0.17.1 -> 4.30.2
2. accelerate 4.28.1 -> 0.20.3
3. bitsandbytes 0.37.2 -> 0.39.1
4. peft 0.2.0 -> 0.4.0dev



### 下载数据
```bash
mkdir results
mkdir lora
mkdir data
```

数据放在目录 `./data` 中。


### 模型
下面是一些模型
* [LLaMA-7b](https://huggingface.co/decapoda-research/llama-7b-hf) | [LLaMA-13b](https://huggingface.co/decapoda-research/llama-13b-hf)
* [Alpaca-7b](https://huggingface.co/circulus/alpaca-7b) | [Alpaca-13b](https://huggingface.co/chavinlo/alpaca-13b)
* [Vicuna-7b-delta-v1.1](https://huggingface.co/lmsys/vicuna-7b-delta-v1.1) | [Vicuna-13b-delta-v1.1](https://huggingface.co/lmsys/vicuna-13b-delta-v1.1) | 
* [zjunlp/knowlm-13b-base-v1.0](https://huggingface.co/zjunlp/knowlm-13b-base-v1.0)(需搭配相应的IE Lora) | [zjunlp/knowlm-13b-zhixi](https://huggingface.co/zjunlp/knowlm-13b-zhixi)(无需Lora即可直接预测) | [zjunlp/knowlm-13b-ie](https://huggingface.co/zjunlp/knowlm-13b-ie)(无需Lora, IE能力更强, 但通用性有所削弱)
* [THUDM/chatglm-6b](https://huggingface.co/THUDM/chatglm-6b)
* [moss-moon-003-sft](https://huggingface.co/fnlp/moss-moon-003-sft)
* [Chinese-LLaMA-7B](https://huggingface.co/Linly-AI/Chinese-LLaMA-7B)
* [baichuan-inc/Baichuan-7B](https://huggingface.co/baichuan-inc/Baichuan-7B) | [baichuan-inc/Baichuan-13B-Base](https://huggingface.co/baichuan-inc/Baichuan-13B-Base) | [baichuan-inc/Baichuan2-7B-Base](https://huggingface.co/baichuan-inc/Baichuan2-7B-Base) | [baichuan-inc/Baichuan2-13B-Base](https://huggingface.co/baichuan-inc/Baichuan2-13B-Base)


## 4.LoRA微调

注意！！以下所有命令都应在`InstrctKGC`目录下执行！！例如运行微调脚本是：`bash scripts/fine_llama.bash`

### LoRA微调LLaMA

你可以通过下面的命令设置自己的参数使用LoRA方法来微调Llama模型:
```bash
output_dir='path to save Llama Lora'
mkdir -p ${output_dir}
CUDA_VISIBLE_DEVICES="0,1" torchrun --nproc_per_node=2 --master_port=1331 src/finetune.py \
    --do_train --do_eval \
    --model_name_or_path 'path or name to Llama' \
    --model_name 'llama' \
    --train_path 'data/train.json' \
    --output_dir=${output_dir}  \
    --per_device_train_batch_size 16 \
    --per_device_eval_batch_size 16 \
    --gradient_accumulation_steps 8 \
    --preprocessing_num_workers 8 \
    --num_train_epochs 10 \
    --learning_rate 1e-4 \
    --optim "adamw_torch" \
    --cutoff_len 512 \
    --val_set_size 1000 \
    --evaluation_strategy "epoch" \
    --save_strategy "epoch" \
    --save_total_limit 10 \
    --lora_r 8 \
    --lora_alpha 16 \
    --lora_dropout 0.05 \
    --max_memory_MB 24000 \
    --fp16 \
    --bits 8 \
    | tee ${output_dir}/train.log \
    2> ${output_dir}/train.err
```

1. Llama模型我们采用[LLaMA-7b](https://huggingface.co/decapoda-research/llama-7b-hf)
2. 你可以使用`--valid_file`提供验证集, 或者什么都不做（在`finetune.py`中, 我们会从train.json中划分`val_set_size`数量的样本做验证集）, 你也可以使用`val_set_size`调整验证集的数量。
3. `prompt_template_name`我们采用默认的`alpaca`模版, 详见 [templates/alpaca.json](./templates/alpaca.json)
4. 更详细的参数信息请参考 [src/utils/args.py](./src/utils/args.py)
5. `max_memory_MB`(默认80000) 指定显存大小, 你需要根据自己的GPU指定
6. 我们在 `RTX3090` 上跑通了llama-lora微调代码

相应的脚本在 [scripts/fine_llama.bash](./scripts/fine_llama.bash)



### LoRA微调Alpaca

微调指令大致遵循上面的[LoRA微调LLaMA](./README_CN.md/#lora微调llama)命令, 仅需对`scripts/fine_llama.bash`做出下列修改

```bash
output_dir='path to save Alpaca Lora'
--model_name_or_path 'path or name to Alpaca' \
--model_name 'alpaca' \
```

1. Alpaca模型我们采用[Alpaca-7b](https://huggingface.co/circulus/alpaca-7b)
2. `prompt_template_name`我们采用默认的`alpaca`模版, 详见 [templates/alpaca.json](./templates/alpaca.json)
3. 我们在 `RTX3090` 上跑通了alpaca-lora微调代码




### LoRA微调智析

首先！！请参考[KnowLM2.2预训练模型权重获取与恢复](https://github.com/zjunlp/KnowLM#2-2)获得完整的智析模型权重。
注意: 由于智析已经在大量的信息抽取指令数据集上经过LoRA训练, 因此可以跳过这一步直接执行预测, 你也可以选择进一步训练。

微调指令大致遵循上面的[LoRA微调LLaMA](./README_CN.md/#lora微调llama)命令, 仅需对`scripts/fine_llama.bash`做出下列修改

```bash
output_dir='path to save Zhixi Lora'
--per_device_train_batch_size 4 \
--per_device_eval_batch_size 4 \
--model_name_or_path 'path or name to Zhixi' \
--model_name 'zhixi' \
```

1. 由于Zhixi目前只有13b的模型, 因此需要相应减小batch size
2. `prompt_template_name`我们采用默认的`alpaca`模版, 详见 [templates/alpaca.json](./templates//alpaca.json)
3. 我们在 `RTX3090` 上跑通了ZhiXi-lora微调代码




### LoRA微调Vicuna

你可以通过下面的命令设置自己的参数使用LoRA方法来微调Vicuna模型:
```bash
output_dir='path to save Vicuna Lora'
mkdir -p ${output_dir}
CUDA_VISIBLE_DEVICES="0,1" torchrun --nproc_per_node=2 --master_port=1331 src/finetune.py \
    --do_train --do_eval \
    --model_name_or_path 'path or name to Vicuna' \
    --model_file 'vicuna' \
    --prompt_template_name 'vicuna' \
    --train_path 'data/train.json' \
    --output_dir=${output_dir}  \
    --per_device_train_batch_size 16 \
    --per_device_eval_batch_size 16 \
    --gradient_accumulation_steps 8 \
    --preprocessing_num_workers 8 \
    --num_train_epochs 10 \
    --learning_rate 1e-4 \
    --optim "adamw_torch" \
    --cutoff_len 512 \
    --val_set_size 1000 \
    --evaluation_strategy "epoch" \
    --save_strategy "epoch" \
    --save_total_limit 10 \
    --lora_r 8 \
    --lora_alpha 16 \
    --lora_dropout 0.05 \
    --max_memory_MB 24000 \
    --fp16 \
    --bits 8 \
    | tee ${output_dir}/train.log \
    2> ${output_dir}/train.err
```

1. Vicuna模型我们采用[Vicuna-7b-delta-v1.1](https://huggingface.co/lmsys/vicuna-7b-delta-v1.1)
2. 由于Vicuna-7b-delta-v1.1所使用的prompt_template_name与`alpaca`模版不同, 因此需要设置 `--prompt_template_name 'vicuna'`, 详见 [templates/vicuna.json](./templates//vicuna.json)
3. `max_memory_MB`(默认80000) 指定显存大小, 你需要根据自己的GPU指定
4. 我们在 `RTX3090` 上跑通了vicuna-lora微调代码

相应的脚本在 [scripts/fine_vicuna.bash](./scripts//fine_vicuna.bash)




### LoRA微调ChatGLM

你可以通过下面的命令设置自己的参数使用LoRA方法来微调ChatGLM模型，注意⚠️目前chatglm更新速度较快，请您确保您的模型与chatglm最新模型保持一致:
```bash
output_dir='path to save ChatGLM Lora'
mkdir -p ${output_dir}
CUDA_VISIBLE_DEVICES="0,1" python --nproc_per_node=2 --master_port=1331 src/finetune.py \
    --do_train --do_eval \
    --model_name_or_path 'path or name to ChatGLM' \
    --model_name 'chatglm' \
    --train_file 'data/train.json' \
    --output_dir=${output_dir}  \
    --per_device_train_batch_size 4 \
    --per_device_eval_batch_size 4 \
    --gradient_accumulation_steps 8 \
    --preprocessing_num_workers 8 \
    --num_train_epochs 10 \
    --learning_rate 1e-5 \
    --weight_decay 5e-4 \
    --adam_beta2 0.95 \
    --optim "adamw_torch" \
    --max_source_length 512 \
    --max_target_length 256 \
    --val_set_size 1000 \
    --evaluation_strategy "epoch" \
    --save_strategy "epoch" \
    --save_total_limit 10 \
    --lora_r 8 \
    --lora_alpha 16 \
    --lora_dropout 0.1 \
    --max_memory_MB 24000 \
    --fp16 \
    | tee ${output_dir}/train.log \
    2> ${output_dir}/train.err
```

1. ChatGLM模型我们采用[THUDM/chatglm-6b](https://huggingface.co/THUDM/chatglm-6b)
2. `prompt_template_name`我们采用默认的`alpaca`模版, 详见 [templates/alpaca.json](./templates/alpaca.json)
3. 由于使用8bits量化后训练得到的模型效果不佳, 因此对于ChatGLM我们没有采用量化策略
4. `max_memory_MB`(默认80000) 指定显存大小, 你需要根据自己的GPU指定
5. 我们在 `RTX3090` 上跑通了chatglm-lora微调代码

相应的脚本在 [scripts/fine_chatglm.bash](./scripts//fine_chatglm.bash)




### LoRA微调Moss

你可以通过下面的命令设置自己的参数使用LoRA方法来微调Moss模型:
```bash
output_dir='path to save Moss Lora'
mkdir -p ${output_dir}
CUDA_VISIBLE_DEVICES="0,1" torchrun --nproc_per_node=2 --master_port=1331 src/finetune.py \
    --do_train --do_eval \
    --model_name_or_path 'path or name to Moss' \
    --model_file 'moss' \
    --prompt_template_name 'moss' \
    --train_path 'data/train.json' \
    --output_dir=${output_dir}  \
    --per_device_train_batch_size 4 \
    --per_device_eval_batch_size 4 \
    --gradient_accumulation_steps 8 \
    --preprocessing_num_workers 8 \
    --num_train_epochs 10 \
    --learning_rate 2e-4 \
    --optim "paged_adamw_32bit" \
    --max_grad_norm 0.3 \
    --lr_scheduler_type 'constant' \
    --max_source_length 512 \
    --max_target_length 256 \
    --val_set_size 1000 \
    --evaluation_strategy "epoch" \
    --save_strategy "epoch" \
    --save_total_limit 10 \
    --lora_r 16 \
    --lora_alpha 64 \
    --lora_dropout 0.05 \
    --max_memory_MB 24000 \
    --fp16 \
    --bits 4 \
    | tee ${output_dir}/train.log \
    2> ${output_dir}/train.err
```

1. Moss模型我们采用[moss-moon-003-sft](https://huggingface.co/fnlp/moss-moon-003-sft)
2. prompt_template_name在alpaca模版的基础上做了一些修改, 详见 [templates/moss.json](./templates/moss.json), 因此需要设置 `--prompt_template_name 'moss'`
3. 由于 `RTX3090` 显存限制, 我们采用`qlora`技术进行4bits量化, 你也可以在`V100`、`A100`上尝试8bits量化和不量化策略
4. `max_memory_MB`(默认80000) 指定显存大小, 你需要根据自己的GPU指定
5. 我们在 `RTX3090` 上跑通了moss-lora微调代码

相应的脚本在 [scripts/fine_moss.bash](./scripts/fine_moss.bash)




### LoRA微调Baichuan

你可以通过下面的命令设置自己的参数使用LoRA方法来微调Baichuan模型:
```bash
CUDA_VISIBLE_DEVICES="0,1" torchrun --nproc_per_node=2 --master_port=1331 src/finetune.py \
    --do_train --do_eval \
    --model_name_or_path 'path or name to Baichuan' \
    --model_name 'baichuan' \
    --train_path 'data/train.json' \
    --output_dir=${output_dir}  \
    --per_device_train_batch_size 6 \
    --per_device_eval_batch_size 6 \
    --gradient_accumulation_steps 8 \
    --preprocessing_num_workers 8 \
    --num_train_epochs 10 \
    --learning_rate 5e-5 \
    --optim "adamw_torch" \
    --cutoff_len 512 \
    --val_set_size 1000 \
    --evaluation_strategy "no" \
    --save_strategy "epoch" \
    --save_total_limit 10 \
    --lora_r 8 \
    --lora_alpha 16 \
    --lora_dropout 0.05 \
    --max_memory_MB 24000 \
    --fp16 \
    --bits 4 \
    | tee ${output_dir}/train.log \
    2> ${output_dir}/train.err
```

1. Baichuan模型我们采用[baichuan-inc/Baichuan2-7B-Base](https://huggingface.co/baichuan-inc/Baichuan2-7B-Base)
2. 目前在evaluation方面存在一些问题, 因此我们使用`evaluation_strategy` "no"
3. `prompt_template_name`我们采用默认的`alpaca`模版, 详见 [templates/alpaca.json](./templates/alpaca.json)
4. 更详细的参数信息请参考 [src/utils/args.py](./src/utils/args.py)
5. `max_memory_MB`(默认80000) 指定显存大小, 你需要根据自己的GPU指定
6. 我们在 `RTX3090` 上跑通了baichuan-lora微调代码

相应的脚本在 [scripts/fine_baichuan.bash](./scripts/fine_baichuan.bash)



## 5.P-Tuning微调


### P-Tuning微调ChatGLM
你可以通过下面的命令使用P-Tuning方法来finetune模型:
```bash
deepspeed --include localhost:0 src/finetuning_pt.py \
  --train_path data/train.json \
  --model_dir /model \
  --num_train_epochs 20 \
  --train_batch_size 2 \
  --gradient_accumulation_steps 1 \
  --output_dir output_dir_pt \
  --log_steps 10 \
  --max_len 768 \
  --max_src_len 450 \
  --pre_seq_len 16 \
  --prefix_projection true
```




## 6.预测


#### LoRA预测
以下是训练好的一些LoRA版本:
* [alpaca-7b-lora-ie](https://huggingface.co/zjunlp/alpaca-7b-lora-ie)
* [llama-7b-lora-ie](https://huggingface.co/zjunlp/llama-7b-lora-ie)
* [alpaca-13b-lora-ie](https://huggingface.co/zjunlp/alpaca-13b-lora-ie)
* [zhixi-13B-LoRA](https://huggingface.co/zjunlp/zhixi-13b-lora/tree/main)

base_model与lora_weights的对应关系:
| base_model   | lora_weights   |
| ------ | ------ |
| llama-7b  | llama-7b-lora  |
| alpaca-7b | alpaca-7b-lora |
| zjunlp/knowlm-13b-base-v1.0 | zhixi-13b-lora |


你可以通过下面的命令设置自己的参数执行来使用训练好的LoRA模型在比赛测试集上预测输出:

```bash
CUDA_VISIBLE_DEVICES="0" python src/inference.py \
    --model_name_or_path 'Path or name to model' \
    --model_name 'model name' \
    --lora_weights 'Path to LoRA weights dictory' \
    --input_file 'data/valid.json' \
    --output_file 'results/results_valid.json' \
    --fp16 \
    --bits 8 
```

1.注意！！`--fp16` 或 `--bf16`、`--bits`、`--prompt_template_name`、`--model_name`一定要与[4.LoRA微调](./README_CN.md/#4lora微调)时设置的一样

也可以使用训练好的模型(无Lora或Lora已合并进模型参数中)在比赛测试集上预测输出:

```bash
CUDA_VISIBLE_DEVICES="0" python src/inference.py \
    --model_name_or_path 'Path or name to model' \
    --model_name 'model name' \
    --input_file 'data/valid.json' \
    --output_file 'results/results_valid.json' \
    --fp16 \
    --bits 8 
```
以下模型支持上述方法：
[zjunlp/knowlm-13b-zhixi](https://huggingface.co/zjunlp/knowlm-13b-zhixi) | [zjunlp/knowlm-13b-ie](https://huggingface.co/zjunlp/knowlm-13b-ie)



### P-Tuning预测

你可以通过下面的命令使用训练好的P-Tuning模型在比赛测试集上预测输出:
```bash
CUDA_VISIBLE_DEVICES=0 python src/inference_pt.py \
  --test_path data/valid.json \
  --device 0 \
  --ori_model_dir /model \
  --model_dir /output_dir_lora/global_step- \
  --max_len 768 \
  --max_src_len 450
```





## 7.模型输出转换&计算F1
我们提供 [evaluate.py](./kg2instruction/evaluate.py) 的脚本，用于将模型的字符串输出转换为列表并计算 F1 分数。

```bash
python kg2instruction/evaluate.py \
  --standard_path data/NER/processed.json \
  --submit_path data/NER/processed.json \
  --task ner \
  --language zh
```


## 8.Acknowledgment

部分代码来自于 [Alpaca-LoRA](https://github.com/tloen/alpaca-lora)、[qlora](https://github.com/artidoro/qlora.git), 感谢！


## Citation

如果您使用了本项目代码或数据，烦请引用下列论文:
```bibtex
@article{DBLP:journals/corr/abs-2305-11527,
  author       = {Honghao Gui and
                  Jintian Zhang and
                  Hongbin Ye and
                  Ningyu Zhang},
  title        = {InstructIE: {A} Chinese Instruction-based Information Extraction Dataset},
  journal      = {CoRR},
  volume       = {abs/2305.11527},
  year         = {2023},
  url          = {https://doi.org/10.48550/arXiv.2305.11527},
  doi          = {10.48550/arXiv.2305.11527},
  eprinttype    = {arXiv},
  eprint       = {2305.11527},
  timestamp    = {Thu, 25 May 2023 15:41:47 +0200},
  biburl       = {https://dblp.org/rec/journals/corr/abs-2305-11527.bib},
  bibsource    = {dblp computer science bibliography, https://dblp.org}
}
```
