# ZeroSpeech Challenge 2021 Baseline System

This repository contains the baseline system of the ZeroSpeech Challenge 2021.

The baseline system consists of 3 different componants: Unsupervised Acoustic Modeling (CPC), Clustering (k-means) and Masked Language Modeling (BERT) as described in \cite{paper}.

## Dependencies
In order to run the evaluation and/or train the baseline model. The two following packages are required:
- [CPC_audio (zerospeech branch)](https://github.com/facebookresearch/CPC_audio/tree/zerospeech)
- [fairseq](https://github.com/pytorch/fairseq)

## Checkpoints
The checkpoints can be downloaded from ??? and put in the directory `checkpoints` as follows:
```
checkpoints  
└───CPC-big-kmeans50-BERT
│   └───CPC_big_6kh
│   │   └─checkpoint_32.pt
│   │   └─checkpoint_args.json
│   └───clustering_kmeans50
│   │   └─clustering_CPC_big_kmeans50.pt
│   │   └─clustering_CPC_big_kmeans50_args.json
│   └───BERT
│       └─BERT_CPC_big_kmeans50.pt
│       └─dict.txt
└───...
```

## Evaluations
### Audio quantization
First of all, we need to quantize the audio files with the CPC+clustering modules.

To quantize a set of audio files, run the `clustering_quantization.py` script.

Example command:
```bash
python clustering_quantization.py checkpoints/CPC-big-kmeans50-BERT/clustering_kmeans50/clustering_CPC_big_kmeans50.pt ../data/LibriSpeech/dev-clean/ ../quantized/LibriSpeech/dev-clean/ --file_extension .flac
```

The quantized units will be written in the `quantized_outputs.txt` file in the output directory.

### Build clustering and one-hot features for ABX
We can use the script `build_CPC_features.py` to extract the CPC features from a set of audio files.

Example command:
```bash
python build_CPC_features.py checkpoints/CPC-big-kmeans50-BERT/CPC_big_6kh/checkpoint_32.pt ../data/LibriSpeech/dev-clean/ ../features/CPC/layer2/LibriSpeech/dev-clean/ --file_extension .flac --gru_level 2
```

In order to export one-hot features from a `quantized_outputs.txt` file, use the `build_1hot_features.py` script:

Example command:
```bash
python build_1hot_features.py ../quantized/LibriSpeech/dev-clean/quantized_outputs.txt ../features/onehot/LibriSpeech/dev-clean/
```

The features of each file will be stored in a corresponding file `file_name.npy`.

### Compute pseudo log-probabilities for sWUGGY, sBLIMP
To compute the pseudo log-probabilities of pseudo-units sequences using BERT model, use the `compute_proba_BERT.py` script.

Example command:
```bash
python compute_proba_BERT.py ../quantized/sWUGGY/dev/quantized_outputs.txt ../scores/sWUGGY/dev/pseudo_log_proba.txt checkpoints/CPC-big-kmeans50-BERT/BERT/BERT_CPC_big_kmeans50.pt
```

### Build BERT features for sSIMI
We can export hidden BERT features of pseudo-units sequences with the `build_BERT_features.py` script.

Example command:
```bash
python build_BERT_features.py ../quantized/sSIMI/dev/quantized_outputs.txt ../features/BERT/layer4/sSIMI/dev/ checkpoints/CPC-big-kmeans50-BERT/BERT/BERT_CPC_big_kmeans50.pt --hidden_level 4
```

The features of each file will be stored in a corresponding file `file_name.npy`.

## Training the baseline system
If you want to reproduce the baseline system, please follow the instructions below.

### CPC
To train the CPC model, follow the instructions at https://github.com/facebookresearch/CPC_audio.

Example command:
```bash
python path/to/CPC_audio/cpc/train.py \
    --pathDB path/to/LibriSpeech/train-clean-100 \
    --pathCheckpoint checkpoints/CPC_small_960h \
    --pathTrain path/to/LibriSpeech/LibriSpeech100_labels_split/train_split.txt \
    --pathVal path/to/LibriSpeech/LibriSpeech100_labels_split/test_split.txt \
    --file_extension .flac
```

where examples of train_split.txt and test_split.txt can be found [here](https://drive.google.com/drive/folders/1BhJ2umKH3whguxMwifaKtSra0TgAbtfb).

### K-means
To train the k-means clustering, run the script `clustering_script.py` in the following [repository](https://github.com/facebookresearch/CPC_audio/tree/zerospeech/cpc/criterion/clustering) `CPC_audio/cpc/criterion/clustering/`.

Example command:
```bash
python clustering_script.py \
    --pathDB path/to/LibriSpeech/train-clean-100/ --recursionLevel 1 \
    --nClusters 50 --MAX_ITER 150 --level_gru 2 \
    --save --load --batchSizeGPU 500 \
    checkpoints/CPC_big_6kh/checkpoint_32.pt \
    checkpoints/clustering_CPC_big_kmeans50/clustering_CPC_big_kmeans50.pt
```

**NOTE:** This command was done on *one 16GB GPU*, and the batchSizeGPU should be modified according to nClusters, so as to fit the memory. Here are the recommended numbers:

nClusters | 20 | 50 | 100 | 200 | 500 | 2000
---|---|---|---|---|---|---
batchSizeGPU | 500 | 500 | 300 | 200 | 100 | 50

### BERT
As long as we have the clustering module, we can quantize the LibriSpeech datasets and train the BERT model on the pseudo units.

We train the fairseq's RoBERTa model as similar to this [example](https://github.com/pytorch/fairseq/blob/master/examples/roberta/README.pretraining.md), with the exception that we used spans of masks insead of single masks (with additional --mask-multiple-length and --mask-stdev options).

Example preprocess command:
```bash
fairseq-preprocess --only-source \
  --trainpref ../quantized/LibriSpeech/fairseq_train.txt \
  --validpref ../quantized/LibriSpeech/fairseq_valid_clean.txt \
  --testpref ../quantized/LibriSpeech/fairseq_test_clean.txt \
  --destdir ../fairseq-bin-data/LibriSpeech \
  --workers 80
```
**NOTE:** The data files contain only the quantized units seperated by space, without the name of the audio files. We need to convert to the right form from the outputs of the `clustering_quantization.py` script, e.g. `5895-34629-0032	31,13,12,12,12,...,13` → `31 13 12 12 12 ... 13`.

Example train command:
```bash
fairseq-train --fp16 ../fairseq-bin-data/LibriSpeech \
    --save-dir checkpoints/BERT_CPC_big_kmeans50 \
    --keep-last-epochs 1 \
    --train-subset train \
    --num-workers 4 \
    --task masked_lm --criterion masked_lm \
    --arch roberta_base \
    --sample-break-mode eos --tokens-per-sample 3072 --max-positions 6144 \
    --optimizer adam --adam-betas '(0.9, 0.98)' --adam-eps 1e-06 --clip-norm 0.0 \
    --lr-scheduler polynomial_decay --lr 0.0005 --total-num-update 250000 --warmup-updates 10000 \
    --dropout 0.1 --attention-dropout 0.1 --weight-decay 0.01 \
    --mask-multiple-length 10 --mask-prob 0.5 --mask-stdev 10 \
    --max-tokens 4096 --update-freq 4 --max-update 250000 \
    --seed 5 --log-format simple --log-interval 10 --skip-invalid-size-inputs-valid-test
```

**NOTE:** This command was done on *32 16GB GPUs*. If train on less GPUs (n GPUs), the --update-freq should be set equal to 128/n. Also, if train on a 32GB GPU, should multiply --max-tokens by 2 accordingly.