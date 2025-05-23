## Overview
<img width="984" alt="Clipboard_Screenshot_1747721350" src="https://github.com/user-attachments/assets/b5459487-b696-45da-9cd3-d64409c235d4" />

## Requirements
We implement the training and RAG pipeline based on [LLaMA-Factory](https://github.com/hiyouga/LLaMA-Factory) and [FlashRAG](https://github.com/RUC-NLPIR/FlashRAG), respectively. 

Please install both libraries and their dependencies as specified in their respective repositories.


## Reproduce

### 0. Datasets


Download preprocessed datasets (augmented with retrieved documents and rationales) from the following [Link](https://drive.google.com/file/d/1MVkdc4g9_D4REtaBFKeJ9gMun4qzdQtO/view?usp=share_link).

### 1. Run Vanilla RAG

Edit configs/eval.yaml and set the generator_model field to your desired model.

Then you can run the following command to generate and evaluate the outputs:
```bash
python eval/inference_vllm.py \
        --data_path $data_path \
        --output_path $output_path \
        --config_file $config_file \
        --type "cot" \
        --dataset_name $dataset_name
```


### 2. Train the CLM

#### 2.1 Synthetic Data

Ensure generator_model is correctly set in configs/eval.yaml.

To generate synthetic rationales:
```bash
types="rationale"
for type in $types; do
    for dataset_name in $dataset_names; do
        python data/data_gen.py \
            --data_path $data_path \
            --output_path $output_path \
            --config_file $config_file \
            --type $type \
            --dataset_name $dataset_name
    done
done
```

To generate synthetic critic rationales:
```bash
types="critic_rationale"
python data/data_gen.py \
    --data_path $data_path \
    --output_path $output_path \
    --config_file $config_file \
    --type $type \
    --dataset_name $dataset_name
```

Then generate the training data:
```bash
python train/data_critic.py \
        --input_path $input_path \
        --output_file $output_path \
        --type "critic_rationale" \
        --dataset_name $dataset_names \
        --filter_type $filter_type
```

add the generated data path to the train/data/dataset_info.json:

```json
{
    "critic_rationale": "data/data_gen.json"
}
```

#### 2.3 Train and Merge CLM

Train the CLM and merge the LoRA weights using LLaMA-Factory:

```bash
FORCE_TORCHRUN=1 CUDA_VISIBLE_DEVICES=0,1 llamafactory-cli main/train_llama/train_llama.yaml
llamafactory-cli export main/train_llama/merge_llama.yaml
```

### 3. Evaluate
To evaluate using the refined CLM, update generator_model in configs/eval.yaml to rbft_llama or rbft_qwen.
Then run:
```bash
python -u eval/inference_vllm.py \
    --dataset_name $DATASET \
    --rag_mode SFT \
    --n_docs $n_docs \
    --output_dir ${MODEL} \
    --critique_model_path $CRITIQUE_MODEL_PATH \
    --model_name_or_path $MODEL_PATH \
    --load_local_model \
    --type "refine" \
    --config_file $config_file \
    --critique_model $CRITIQUE_MODEL \
    --iters $iter \
    --phase $phase \
```

## Results
<img width="1036" alt="Clipboard_Screenshot_1747721379" src="https://github.com/user-attachments/assets/e672b6ba-74f1-49e2-b89f-ae091d000b4f" />
