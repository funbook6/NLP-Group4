# 基于开源大模型微调中文安全检测与关怀系统
## 项目简介
### 背景
随着大模型的迅速发展，AI迎来了更广大的普通用户，然而，对于潜在的危险用户，大模型针对越狱问题能否给予合规回答并规劝，在遇到心理安全问题时能否能给予初步帮助，是AI普及时代的关键议题。
本项目以总体国家安全观为指导思想，构建覆盖“越狱安全+心理安全”的防护系统———当检测到越狱攻击时予以拒绝规劝，当识别到心理危机时给予安抚
并主动提供求助资源，同时将高危行为实时上报后台，形成“检测-干预-上报”三位一体的安全闭环。

### 问题定义
本项目旨在构建一个基于开源大语言模型微调的中文安全检测与关怀系统；系统面向中文文本输入，
能够正确分类正常请求、越狱/不安全请求以及心理危机表达，并生成对应的安全回复或关怀回复。

## 任务定义
项目基于 **Qwen2.5-1.5B-Instruct**，使用 LoRA 进行参数高效微调，并设计 Base 模型、普通 LoRA 模型和文言增强 LoRA 模型三组实验，用于验证微调效果和文言化表达下的鲁棒性。

三类定义：
* safe：正常请求，可以正常回答。
* jailbreak：越狱、危险、有害或诱导绕过规则的请求，需要安全拒答。
* psych：心理危机、明显心理困扰、自伤/自杀风险表达，需要共情支持和求助建议。

输入：用户中文文本  
输出：   
【风险类型】safe / jailbreak / psych  
【回复】模型回复内容

## 模型说明

| 模型        | 训练方式    | 训练数据           | 作用      |
|-----------|---------|----------------|---------|
| Base      | 不微调     | 无              | 基线对照    |
| 普通 LoRA   | LoRA 微调 | 普通风险训练集        | 验证微调效果  |
| 文言增强 LoRA | LoRA 微调 | 普通训练集 + 文言增强样本 | 验证文言鲁棒性 |

## 代码仓库结构说明
按照项目流程分为三个主要部分：
```text
代码仓库/
├── 数据集处理/
│   ├── data_plus.py
│   ├── generate_psych_reply.py
│   └── new_wenyan.py
├── 无文言文数据微调/
│   ├── configs/
│   ├── data/
│   ├── eval_results/
│   ├── model/
│   └── scripts/
└── 有文言文数据集微调/
    ├── configs/
    ├── data/
    ├── eval_results/
    ├── model/
    └── scripts/
```

| 目录        | 作用                                 |
|-----------|------------------------------------|
| 数据集处理     | 完成多源数据合并、标签映射、psych 回复生成和文言样本构造    |
| 无文言文数据微调  | 保存普通 LoRA 模型的训练配置、预测结果、评测结果和模型文件   |
| 有文言文数据集微调 | 保存文言增强 LoRA 模型的训练配置、预测结果、评测结果和模型文件 |

## 数据清洗与说明
数据来源、标签映射、去重流程、psych 回复生成方式和文言样本构造方法详见：
[数据说明文档](数据说明文档.pdf)

## 环境配置
平台：AutoDL  
GPU：RTX 5090  
框架：LLaMA-Factory  
基座模型：Qwen2.5-1.5B-Instruct  
微调方式：LoRA  
精度：bf16  
具体依赖以 LLaMA-Factory 环境为准，本项目额外需要 pandas、scikit-learn、matplotlib、jieba 用于评测。

## 具体步骤
训练前确认数据集已经注册到 `data/dataset_info.json`，并且训练、验证、测试数据已经放入对应的 `data/` 目录，在两台电脑上分别运行普通LoRA模型和文言增强LoRA模型，
### 训练模型
1.普通LoRA训练：  
```bash
cd /root/autodl-tmp/LLaMA-Factory
mkdir -p logs
screen -S train_normal_lora
HF_ENDPOINT=https://hf-mirror.com llamafactory-cli train configs/train_lora_normal.yaml 2>&1 | tee logs/train_normal.log
```
2.文言增强 LoRA 训练：
```bash
cd /root/autodl-tmp/LLaMA-Factory
mkdir -p logs

screen -S train_wenyan_lora

HF_ENDPOINT=https://hf-mirror.com llamafactory-cli train configs/train_lora_wenyan.yaml 2>&1 | tee logs/train_wenyan.log
```
训练完成后，主要输出包括：

| 文件 / 目录                             | 说明              |
|-------------------------------------|-----------------|
| model/                              | LoRA adapter 权重 |
| training_loss.png                   | 训练损失曲线          |
| training_eval_loss.png              | 验证损失曲线          |
| train_normal.log / train_wenyan.log | 训练日志            |

### 预测模型
所有模型均采用 LLaMA-Factory 的 `do_predict` 模式进行离线预测，预测结果统一保存为 `generated_predictions.jsonl`。
```bash
#Base 普通测试集预测，用于基线对照：  
cd /root/autodl-tmp/LLaMA-Factory
HF_ENDPOINT=https://hf-mirror.com llamafactory-cli train configs/predict_base_testf.yaml 2>&1 | tee logs/predict_base_testf.log
#Base模型文言文测试集预测：
HF_ENDPOINT=https://hf-mirror.com llamafactory-cli train configs/predict_base_wenyan.yaml 2>&1 | tee logs/predict_base_wenyan.log
#普通 LoRA 普通测试集预测：  
HF_ENDPOINT=https://hf-mirror.com llamafactory-cli train configs/predict_lora_testf.yaml
#普通 LoRA 文言测试集预测：  
HF_ENDPOINT=https://hf-mirror.com llamafactory-cli train configs/predict_lora_wenyan.yaml
#文言增强 LoRA 预测：  
HF_ENDPOINT=https://hf-mirror.com llamafactory-cli train configs/predict_wenyan_lora_test.yaml
```
预测完成后，每个输出目录下会生成：  
generated_predictions.jsonl  
predict_results.json  
trainer_log.jsonl

## 评测部分
评测指标体系、三模型结果对比等具体评价体系详见[评测报告](评测报告.pdf)


## 团队分工
| 成员          | 分工                                          |
|-------------|---------------------------------------------|
| 徐凡舒 2312754 | 选题框架确定，部分数据收集，普通样本微调模型的训练以及评测，部分ppt制作，交付物整理 |
| 高曼泽 2311579 | 文言文样本微调模型的训练以及评测，评测报告的撰写，代码仓库整合，部分ppt制作     |
| 谢筱昀 2310726 | 部分数据收集，数据清洗与预处理，数据说明文档撰写，部分ppt制作            |
| 王涵韵 2311145 | 文言样本转化，数据说明文档撰写，部分ppt制作                     |
