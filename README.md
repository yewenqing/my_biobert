# BERT / BioBERT 疾病实体识别

使用 NCBI Disease 的 train / develop / test 固定划分。这里的后训练指基于已有预训练权重的有监督 NER 微调，更新编码器和分类头，不是继续进行 MLM 预训练。

## 安装

### Google Colab

在 [Google Colab](https://colab.research.google.com/) 选择上传 Notebook，打开本项目的 `colab_ner.ipynb`，选择 GPU 运行时，按顺序执行单元即可。Notebook 已内置代码与三份 CoNLL 数据，不需要额外上传项目。

配置单元支持选择 `bert` / `biobert`、训练参数、已训练模型路径及断点恢复。默认挂载 Google Drive 保存模型和 checkpoint；也可关闭 Drive 保存，训练结束下载模型 ZIP。训练后显示 accuracy、precision、recall、F1。

修改本地训练代码或 CoNLL 数据后，运行 `python build_colab_notebook.py` 更新 Notebook 内置快照，再上传 Colab。

### 本地

```bash
python -m pip install -r requirements.txt
```

首次运行需要联网下载 Hugging Face 权重；支持 CPU，完整训练建议使用 GPU。当前 `data/conll/` 已有预处理数据。若需重新生成，额外安装 `spacy`，运行 `python processing.py`。

## 训练及测试

BERT-base（默认 bert-base-cased）：

```bash
python run.py train --model bert --output-dir outputs/bert --epochs 3
```

训练在 develop 集上选择实体 F1 最佳模型，再在 test 集上测试。保存的模型位于 `outputs/bert/best_model/`，测试结果位于 `outputs/bert/test_metrics.json`。

BioBERT 使用同样的数据划分和参数进行微调及测试：

```bash
python run.py train --model biobert --output-dir outputs/biobert --epochs 3
```

分别加载已训练模型进行测试：

```bash
python run.py evaluate --checkpoint outputs/bert/best_model --output-dir outputs/bert_eval
python run.py evaluate --checkpoint outputs/biobert/best_model --output-dir outputs/biobert_eval
```

继续微调已有模型，或完整恢复中断的训练：

```bash
python run.py train --checkpoint outputs/bert/best_model --output-dir outputs/bert_more --epochs 2
python run.py train --model bert --output-dir outputs/bert --resume-from-checkpoint outputs/bert/checkpoint-1000
```

第二个命令中的 checkpoint 路径需替换为真实存在的目录，epochs 表示恢复后的总目标轮数。外部 NER checkpoint 必须包含明确的 `O / B-Disease / I-Disease` 标签映射。

## 未微调基线与指标

原始 BERT-base、BioBERT 权重没有训练好的疾病 NER 分类头。若要观察直接加载基础模型的基线，可显式运行以下命令；分类头会随机初始化，分数不能代表微调后模型效果：

```bash
python run.py evaluate --model bert --allow-untrained-head --output-dir outputs/bert_baseline
python run.py evaluate --model biobert --allow-untrained-head --output-dir outputs/biobert_baseline
```

所有指标为 0–1：`test_accuracy` 是词级标签准确率；`test_precision`、`test_recall`、`test_f1` 是实体级 micro 指标，要求边界和类别完全匹配。严格 BIO 下孤立的 I 标签不视为实体起点。数据里的任务标记不参与计算，仅首个 WordPiece 参与监督与评估，特殊 token / padding 为 -100。超长句按完整词分块，不丢弃尾部；评估先还原句子再计算实体，跨块上下文仍受模型长度限制。

实现参考 [Transformers 官方 NER 示例](https://huggingface.co/docs/transformers/v4.50.0/en/tasks/token_classification)。

轻量验证（无需安装模型依赖）：

```bash
python -m unittest discover -s tests -v
```
# my_biobert
