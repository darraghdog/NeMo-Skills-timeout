# Model evaluation

Here are the commands you can run to reproduce our evaluation numbers.
The commands below are for OpenMath-2-Llama3.1-8b model as an example.
We assume you have `/workspace` defined in your [cluster config](../basics/prerequisites.md#cluster-configs) and are
executing all commands from that folder locally. Change all commands accordingly
if running on slurm or using different paths.

## Download models

Get the model from HF. E.g.

```bash
pip install -U "huggingface_hub[cli]"
huggingface-cli download nvidia/OpenMath2-Llama3.1-8B --local-dir OpenMath2-Llama3.1-8B
```

## Convert to TensorRT-LLM

Convert the model to TensorRT-LLM format. This is optional, but highly recommended for more exact
results and faster inference. If you skip it, replace `--server_type trtllm` with `--server-type vllm`
in the commands below and change model path to `/workspace/OpenMath2-Llama3.1-8B`. You might also need
to set smaller batch size for vllm.

```bash
ns convert \
    --cluster=local \
    --input_model=/workspace/OpenMath2-Llama3.1-8B \
    --output_model=/workspace/openmath2-llama3.1-8b-trtllm \
    --convert_from=hf \
    --convert_to=trtllm \
    --num_gpus=1 \
    --hf_model_name=nvidia/OpenMath2-Llama3.1-8B
```

Change the number of GPUs if you have more than 1 (required for 70B model).

## Prepare evaluation data

```bash
python -m nemo_skills.dataset.prepare gsm8k math amc23 aime24 omni-math
```

## Run greedy decoding

```bash
ns eval \
    --cluster=local \
    --model=/workspace/openmath2-llama3.1-8b-trtllm \
    --server_type=trtllm \
    --output_dir=/workspace/openmath2-llama3.1-8b-eval \
    --benchmarks=aime24:0,amc23:0,math:0,gsm8k:0,omni-math:0 \
    --server_gpus=1 \
    --num_jobs=1 \
    ++prompt_template=llama3-instruct \
    ++batch_size=512 \
    ++inference.tokens_to_generate=4096
```

If running on slurm, you can set `--num_jobs` to a bigger number of -1 to run
each benchmark in a separate node. The number of GPUs need to match what you used
in the conversion command.

After the generation is done, we want to run LLM-as-a-judge evaluation to get more
accurate numbers than symbolic comparison. You need to define `OPENAI_API_KEY` for
the command below to work.

```bash
ns llm_math_judge \
    --cluster=local \
    --model=gpt-4o \
    --server_type=openai \
    --server_address=https://api.openai.com/v1 \
    --input_files="/workspace/openmath2-llama3.1-8b-eval/eval-results/**/output*.jsonl"
```

Finally, to print the metrics run

```bash
ns summarize_results /workspace/openmath2-llama3.1-8b-eval/eval-results --cluster local
```

This should print the metrics including both symbolic and judge evaluation. The judge is typically more accurate.

```
------------------------------------------------- aime24 ------------------------------------------------
evaluation_mode | num_entries | symbolic_correct | judge_correct | both_correct | any_correct | no_answer
greedy          | 30          | 10.00            | 10.00         | 10.00        | 10.00       | 6.67


------------------------------------------------- gsm8k -------------------------------------------------
evaluation_mode | num_entries | symbolic_correct | judge_correct | both_correct | any_correct | no_answer
greedy          | 1319        | 90.75            | 91.70         | 90.75        | 91.70       | 0.00


----------------------------------------------- omni-math -----------------------------------------------
evaluation_mode | num_entries | symbolic_correct | judge_correct | both_correct | any_correct | no_answer
greedy          | 4428        | 18.97            | 22.22         | 18.11        | 23.08       | 2.55


-------------------------------------------------- math -------------------------------------------------
evaluation_mode | num_entries | symbolic_correct | judge_correct | both_correct | any_correct | no_answer
greedy          | 5000        | 67.70            | 68.10         | 67.50        | 68.30       | 1.36


------------------------------------------------- amc23 -------------------------------------------------
evaluation_mode | num_entries | symbolic_correct | judge_correct | both_correct | any_correct | no_answer
greedy          | 40          | 32.50            | 40.00         | 32.50        | 40.00       | 0.00
```

The numbers may vary by 1-2% depending on the server type, number of GPUs and batch size used.

## Run majority voting

```bash
ns eval \
    --cluster=local \
    --model=/workspace/openmath2-llama3.1-8b-trtllm \
    --server_type=trtllm \
    --output_dir=/workspace/openmath2-llama3.1-8b-eval \
    --benchmarks=aime24:256,amc23:256,math:256,gsm8k:256,omni-math:256 \
    --server_gpus=1 \
    --num_jobs=1 \
    --skip_greedy \
    ++prompt_template=llama3-instruct \
    ++batch_size=512 \
    ++inference.tokens_to_generate=4096
```

This will take a very long time unless you run on slurm cluster. After the generation is done, you will be able
to see symbolic scores right away. You can evaluate with the judge by first creating new files with majority
answers. E.g. for "math" benchmark run

```bash
python -m nemo_skills.evaluation.fill_majority_answer \
    ++input_files="./openmath2-llama3.1-8b-eval/eval-results/math/output-rs*.jsonl" \
    ++fill_key=predicted_answer
```

This will replace `predicted_answer` in all files with majority answer.

After that, let's copy just a single of those files into a new folder so that we can run the llm-judge pipeline
on them.

```bash
mkdir -p ./openmath2-llama3.1-8b-eval/eval-results-majority/math
cp ./openmath2-llama3.1-8b-eval/eval-results/math/output-rs0.jsonl ./openmath2-llama3.1-8b-eval/eval-results-majority/math/
```

Repeat the above steps for all benchmarks. Now we are ready to run the judge pipeline and summarize results
after it is finished. You need to define `OPENAI_API_KEY` for the command below to work.

```bash
ns llm_math_judge \
    --cluster=local \
    --model=gpt-4o \
    --server_type=openai \
    --server_address=https://api.openai.com/v1 \
    --input_files="/workspace/openmath2-llama3.1-8b-eval/eval-results-majority/**/output*.jsonl"
```

```bash
ns summarize_results /workspace/openmath2-llama3.1-8b-eval/eval-results-majority --cluster local
```

This will print majority results (they will be labeled as `majority@1` since we fused them into a single file).
You can also ignore the symbolic score as it's not accurate anymore after we filled majority answers.