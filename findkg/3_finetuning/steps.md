## Get Nemo microservice docker
### Outside docker

`docker run --gpus device=all --shm-size=20g --net=host --ulimit memlock=-1 --rm -it -v ${PWD}:/workspace -w /workspace -v ${PWD}/results:/rresults:/results nvcr.io/ea-bignlp/ga-participants/nemofw-training:23.08.03 bash`

`download llama 7b using /workspace/findkg/3_finetuning/3.1_download_llama7b.py`

## convert the model weight to .nemo format
python /opt/NeMo/scripts/nlp_language_modeling/convert_hf_llama_to_nemo.py --in-file=/workspace/findkg/3_finetuning/llama2-7b-hf --out-file=llama2-7b.nemo

# data processing
/workspace/findkg/3_finetuning/3.2_data_preprocessing.py 

# split data
/workspace/findkg/3_finetuning/3.3_data_split.py

## SFT training 

MODEL="/workspace/findkg/3_finetuning/llama2-7b.nemo"
TRAIN="/workspace/data/processed/us-financial-news-articles/SFT_training/training.jsonl"
VALID="/workspace/data/processed/us-financial-news-articles/SFT_training/validation.jsonl"
TEST="/workspace/data/processed/us-financial-news-articles/SFT_training/test.jsonl"
VALID_NAMES="[/workspace/data/processed/us-financial-news-articles/SFT_training]"
CONCAT_SAMPLING_PROBS="[1.0]"
TP_SIZE=8
PP_SIZE=1

### config file 

`/opt/NeMo/examples/nlp/language_modeling/conf/megatron_gpt_config.yaml
`

```yaml

name: megatron_gpt_sft

trainer:
  devices: 1
  accelerator: gpu
  num_nodes: 1
  precision: 16
  logger: False # logger provided by exp_manager
  enable_checkpointing: False
  use_distributed_sampler: False
  max_epochs: 9999
  max_steps: 20000 # consumed_samples = global_step * micro_batch_size * data_parallel_size * accumulate_grad_batches
  log_every_n_steps: 10 # frequency with which training steps are logged 
  val_check_interval: 200 # If is an int n > 1, will run val every n training steps, if a float 0.0 - 1.0 will run val every epoch fraction, e.g. 0.25 will run val every quarter epoch
  gradient_clip_val: 1.0

exp_manager:
  explicit_log_dir: null
  exp_dir: null
  name: ${name}
  create_wandb_logger: False
  wandb_logger_kwargs:
    project: null
    name: null
  resume_if_exists: True
  resume_ignore_no_checkpoint: True
  create_checkpoint_callback: True
  checkpoint_callback_params:
    monitor: validation_${model.data.validation_ds.metric.name}
    save_top_k: 2
    mode: max
    save_nemo_on_train_end: False 
    filename: 'megatron_gpt_sft--{${exp_manager.checkpoint_callback_params.monitor}:.3f}-{step}-{consumed_samples}'
    model_parallel_size: ${model.tensor_model_parallel_size}
    save_best_model: True

model:
  seed: 1234
  tensor_model_parallel_size: 1 # intra-layer model parallelism
  pipeline_model_parallel_size: 1 # inter-layer model parallelism
  global_batch_size: 128
  micro_batch_size: 4
  restore_from_path: ??? # Path to an existing p-tuned/prompt tuned .nemo model you wish to add new tasks to or run inference with
  resume_from_checkpoint: null # The path to a checkpoint file to continue the training, restores the whole state including the epoch, step, LR schedulers, apex, etc.
  save_nemo_on_validation_end: True # Saves an inference ready .nemo file every time a checkpoint is saved during training. 
  sync_batch_comm: False
  megatron_amp_O2: False

  ## Sequence Parallelism
  # Makes tensor parallelism more memory efficient for LLMs (20B+) by parallelizing layer norms and dropout sequentially
  # See Reducing Activation Recomputation in Large Transformer Models: https://arxiv.org/abs/2205.05198 for more details.
  sequence_parallel: False

  ## Activation Checkpoint 
  activations_checkpoint_granularity: null # 'selective' or 'full' 
  activations_checkpoint_method: null # 'uniform', 'block', not used with 'selective'
  # 'uniform' divides the total number of transformer layers and checkpoints the input activation
  # of each chunk at the specified granularity
  # 'block' checkpoints the specified number of layers per pipeline stage at the specified granularity
  activations_checkpoint_num_layers: null # not used with 'selective'
  activations_checkpoint_layers_per_pipeline: null
  # This feature is valid only when used with pipeline-model-parallelism. More details in megatron_gpt_config.yaml.
  answer_only_loss: False # not used right now
  gradient_as_bucket_view: False
  seq_len_interpolation_factor: null # if not None, seq_len_interpolation_factor will match the base model's value
  use_flash_attention: null # if not None, will match the base model's value

  hidden_dropout: 0.0
  attention_dropout: 0.0
  ffn_dropout: 0.0

  data:
    chat: False # whether use chatbot data or not
    train_ds:
      # Example of how to specify paths to multiple datasets
      # file_names: 
      #   - /path/to/squad.jsonl
      #   - /path/to/mnli.jsonl
      #   - /path/to/boolq.jsonl
      # Example of how each dataset is formatted
      # {'input': 'John von Neumann\nVon Neumann made fundamental contributions .... Q: What did the math of artificial viscosity do?', 'output': 'smoothed the shock transition without sacrificing basic physics'}
      file_names: ??? # Path to a list of JSONL files corresponding to the source data.
      global_batch_size: ${model.global_batch_size}
      micro_batch_size: ${model.micro_batch_size}
      shuffle: True
      num_workers: 4
      memmap_workers: null
      pin_memory: True
      max_seq_length: 2048
      min_seq_length: 1
      drop_last: True
      # Example of how to specify concat_sampling_probabilities
      # concat_sampling_probabilities:
      #   - 0.5
      #   - 0.25
      #   - 0.25
      concat_sampling_probabilities: null # When providing a list of datasets, this arg defines the sampling probabilities from each dataset when strategy='random'
      label_key: 'output'
      add_eos: True
      add_sep: False
      add_bos: False
      truncation_field: "input" # # Can be multiple keys separated with ',' Options: keys in prompt_template
      index_mapping_dir: null # Path to a directory to write index mapping files.
      prompt_template: "{input} {output}" # fstring to use for assistant prompt. Example: "Q: {input}\nA: {output}"
      hf_dataset: False # Whether to load the json file with the HuggingFace dataset. otherwise, will load the jsonl file with the JSONLMemMapDataset.
      truncation_method: 'right' # Truncation from which position, Options: ['left', 'right'] 

    validation_ds:
      file_names: ??? # Path to a list of JSONL files corresponding to the source data. Data format is identical to train_ds.
      names: null # Names of the corresponding datasets used to log metrics.
      global_batch_size: ${model.global_batch_size}
      micro_batch_size: ${model.micro_batch_size}
      shuffle: False
      num_workers: 4
      memmap_workers: ${model.data.train_ds.memmap_workers}
      pin_memory: True
      max_seq_length: ${model.data.train_ds.max_seq_length}
      min_seq_length: 1
      drop_last: False
      label_key: ${model.data.train_ds.label_key}
      add_eos: ${model.data.train_ds.add_eos}
      add_sep: ${model.data.train_ds.add_sep}
      add_bos: ${model.data.train_ds.add_bos}
      write_predictions_to_file: False
      output_file_path_prefix: null # Prefix of the file to write predictions to.
      truncation_field: ${model.data.train_ds.truncation_field} # Options: keys in prompt_template
      index_mapping_dir: null # Path to a directory to write index mapping files.
      prompt_template: ${model.data.train_ds.prompt_template} # fstring to use for assistant prompt. Example: "Q: {input}\nA: {output}"
      tokens_to_generate: 32 # decide how many tokens we want to generate to evaluate performance with string metrics
      hf_dataset: False # Whether to load the json file with the HuggingFace dataset. otherwise, will load the jsonl file with the JSONLMemMapDataset.
      truncation_method: 'right' # Truncation from which position, Options: ['left', 'right'] 

      metric:
        name: "loss" # Name of the evaluation metric to use. Options: ['exact_string_match', 'loss', 'rouge', 'token_f1']
        average: null # Average the metric over the dataset. Options: ['macro', 'micro']. Works only for 'F1', 'accuracy' etc. Refer to torchmetrics for metrics where this is supported.
        num_classes: null

    test_ds:
      file_names: ??? # Path to a list of JSONL files corresponding to the source data. Data format is identical to train_ds.
      names: null # Names of the corresponding datasets used to log metrics.
      global_batch_size: ${model.global_batch_size}
      micro_batch_size: ${model.micro_batch_size}
      shuffle: False
      num_workers: 4
      memmap_workers: ${model.data.train_ds.memmap_workers}
      pin_memory: True
      max_seq_length: ${model.data.train_ds.max_seq_length}
      min_seq_length: 1
      drop_last: False
      label_key: ${model.data.train_ds.label_key}
      add_eos: ${model.data.train_ds.add_eos}
      add_sep: ${model.data.train_ds.add_sep}
      add_bos: ${model.data.train_ds.add_bos}
      write_predictions_to_file: False
      output_file_path_prefix: null # Prefix of the file to write predictions to.
      truncation_field: ${model.data.train_ds.truncation_field} # Options: keys in prompt_template
      index_mapping_dir: null # Path to a directory to write index mapping files.
      prompt_template: ${model.data.train_ds.prompt_template} # fstring to use for assistant prompt. Example: "Q: {input}\nA: {output}"
      tokens_to_generate: 32 # decide how many tokens we want to generate to evaluate performance with string metrics
      hf_dataset: False # Whether to load the json file with the HuggingFace dataset. otherwise, will load the jsonl file with the JSONLMemMapDataset.
      truncation_method: 'right' # Truncation from which position, Options: Options: ['left', 'right']

      metric:
        name: "loss" # Name of the evaluation metric to use. Options: ['exact_string_match', 'loss']
        average: null # Average the metric over the dataset. Options: ['macro', 'micro']. Works only for 'F1', 'accuracy' etc. Refer to torchmetrics for metrics where this is supported.
        num_classes: null

  optim:
    name: fused_adam # Supports distributed optimizer for memory savings. To enable, set to 'distributed_fused_adam'. Needs Apex to be built with specific args to work.
    lr: 3e-5
    weight_decay: 0.01 
    betas: 
    - 0.9
    - 0.98

inference:
  greedy: True # Whether or not to use sampling ; use greedy decoding otherwise
  top_k: 0  # The number of highest probability vocabulary tokens to keep for top-k-filtering.
  top_p: 0.9 # If set to float < 1, only the most probable tokens with probabilities that add up to top_p or higher are kept for generation.
  temperature: 1.0 # sampling temperature
  all_probs: False  # whether return the log prob for all the tokens in vocab
  repetition_penalty: 1.2  # The parameter for repetition penalty. 1.0 means no penalty.
  min_tokens_to_generate: 0  # The minimum length of the sequence to be generated.
  compute_logprob: False  # a flag used to compute logprob of all the input text, a very special case of running inference, default False
  compute_attention_mask: True

```

### job launch code
```bash 
 torchrun --nproc_per_node=8 /opt/NeMo/examples/nlp/language_modeling/tuning/megatron_gpt_sft.py    trainer.precision=bf16    trainer.devices=8    trainer.num_nodes=1    trainer.val_check_interval=0.1    trainer.max_steps=50    model.micro_batch_size=1    model.global_batch_size=128    model.tensor_model_parallel_size=${TP_SIZE}    model.pipeline_model_parallel_size=${PP_SIZE}    model.megatron_amp_O2=True    model.sequence_parallel=True    model.activations_checkpoint_granularity=selective    model.activations_checkpoint_method=uniform    model.optim.name=distributed_fused_adam    model.optim.lr=5e-6      model.data.train_ds.file_names=${TRAIN_DS}    model.data.validation_ds.file_names=${VALID_DS}    model.data.test_ds.file_names=${TEST_DS}    model.data.train_ds.concat_sampling_probabilities=${CONCAT_SAMPLING_PROBS}    model.data.train_ds.max_seq_length=2048    model.data.validation_ds.max_seq_length=2048    model.data.train_ds.micro_batch_size=1    model.data.train_ds.global_batch_size=128    model.data.validation_ds.micro_batch_size=1    model.data.validation_ds.global_batch_size=128    model.data.test_ds.micro_batch_size=1    model.data.test_ds.global_batch_size=256    model.data.train_ds.num_workers=0    model.data.validation_ds.num_workers=0    model.data.test_ds.num_workers=0    model.data.validation_ds.metric.name=loss    model.data.test_ds.metric.name=loss   exp_manager.create_wandb_logger=False exp_manager.explicit_log_dir=/results    exp_manager.resume_if_exists=True    exp_manager.resume_ignore_no_checkpoint=True    exp_manager.create_checkpoint_callback=True    exp_manager.checkpoint_callback_params.monitor=validation_loss    exp_manager.checkpoint_callback_params.save_best_model=False    exp_manager.checkpoint_callback_params.save_nemo_on_train_end=True  model.restore_from_path=/workspace/findkg/3_finetuning/llama2-7b.nemo
```