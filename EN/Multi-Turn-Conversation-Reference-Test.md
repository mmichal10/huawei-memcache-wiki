# Test Preparation
Prepare the dataset files and testing tools. This guide uses the open-source LongBench dataset (with an average length of several thousand characters, which fits this use case) to build scenario-based tests for long-document question answering and multi-turn conversations. These tests help readers quickly evaluate the benefits of KVCache tiered caching and determine whether it suits their scenarios.

## Step 1: Download the Dataset
LongBench dataset download link: https://huggingface.co/datasets/zai-org/LongBench. Download and extract the JSONL files for use. You may also use your own JSONL dataset and convert it with the preprocessing script below.

## Step 2: Prepare the Testing Tool
This guide uses the open-source AISBench tool for testing. It can be deployed in a Docker container.

AISBench repository: https://github.com/AISBench/benchmark

AISBench image usage guide and download link: https://github.com/AISBench/benchmark/wiki/AISBench-Docker-Images-Guidance

When using the standalone AISBench image, first download the compressed image file from the link to the server.
```
# Load the image
docker load -i aisbench_benchmark_v3.0-20251219-master_aarch64_py_310.tar.gz

# Create a container with network access. Mount the model weights and dataset directories as appropriate (-v specifies a mount).
docker run --name ais_bench_container -itd --net=host \
-w /benchmark \
--ipc=host \
-v /model_path:/model_path \
-v /datasets:/datasets \
--entrypoint /bin/bash \
ghcr.io/aisbench/aisbench_benchmark:v3.0-20251219-master_aarch64_py_310

# Enter the container, which has the AISBench environment installed
docker exec -ti ais_bench_container bash
```

## Step 3: Configure the Test
When using AISBench, configure the model service and dataset information as follows:
```
# Find the configuration file path
ais_bench --models vllm_api_general_chat --custom-dataset-path /datasets/qa.jsonl --mode perf --debug --num-warmups 0 --search

# Locate the model service configuration file
│ --models    │ vllm_api_general_chat │ /benchmark/ais_bench/benchmark/configs/models/vllm_api/vllm_api_general_chat.py │

# Edit the configuration file according to the service information
vim /benchmark/ais_bench/benchmark/configs/models/vllm_api/vllm_api_general_chat.py
```
Update the following parameters according to the service information:
```
models = [
    dict(
        attr="service",
        type=VLLMCustomAPIChat,
        abbr="vllm-api-general-chat",
        path="/data/weights/DeepSeek-R1-Distill-Qwen-32B-w8a8", # Model weights path; the testing tool needs to read files such as the vocabulary
        model="qwen3", # Model name; must match the name used to start the service
        stream=True, # Whether to enable the streaming API
        request_rate=0, # Request rate; 0 means concurrency testing
        retry=2,
        api_key="",
        host_ip="xx.xx.xx.xx", # Set this to the actual IP address
        host_port=8113, # Set this to the configured port
        url="",
        max_out_len=1024, # Maximum output length
        batch_size=32, # Request concurrency
        trust_remote_code=False,
        generation_kwargs=dict(
            temperature=0.01,
            ignore_eos=True, # When True, output reaches the maximum output length. Recommended for performance tests to avoid fluctuations caused by varying output lengths.
        ),
        pred_postprocessor=dict(type=extract_non_reasoning_content),
    )
]
```
## Step 4: Check That the Test Runs Successfully
Create a simple question-and-answer dataset (`qa.jsonl`) to check that the test environment runs correctly. You can replace it with your own dataset or an open-source dataset.
```
{"question": "Q1", "answer": ""}
{"question": "Q2", "answer": ""}
{"question": "Q3", "answer": ""}
{"question": "Q4", "answer": ""}
```
After configuring the test, run it and check that it completes successfully:
```
# Update the configuration and run a request test
ais_bench --models vllm_api_general_chat --custom-dataset-path /datasets/qa.jsonl --mode perf --debug --num-warmups 0
```
# Multi-Turn Conversation Reference Test
The test uses different questions to form a multi-turn conversation that simulates a real-world scenario. Starting with the second turn, questions and answers from earlier turns may be cache hits, demonstrating the performance benefit of cache hits.

Convert the source dataset (for example, `multi_news.jsonl`) to the ShareGPT format supported by AISBench. See the guide at https://github.com/AISBench/benchmark/blob/master/ais_bench/benchmark/configs/datasets/sharegpt/README.md. An example command follows; remove the comments before running it.

## 1. Before Processing: Source Dataset Format
Suppose the source JSONL dataset before preprocessing contains the following, where Q1–Q4 represent different source questions:
```
{"context": "Q1", "xx": "xx"} 
{"context": "Q2", "xx": "xx"} 
{"context": "Q3", "xx": "xx"} 
{"context": "Q4", "xx": "xx"}
```

## 2. Process the Dataset
The source dataset before preprocessing must use the `.jsonl` suffix. Specify the `.json` suffix for the output file containing the processed multi-turn conversations.

```
# Example
python dataset_convert.py \
--input multi_news.jsonl \ # Input file; multiple files are supported
--output multi_news_convert.json \ # Output file
--top-n 32\ # Number of items to select
--min-length 1024 \ # Minimum length filter; shorter items are excluded
--max-length 4096\ # Maximum length filter; longer items are excluded
--repeat 1 \ # Run only one round
--source-fields context \ # Input fields in the source JSONL file; multiple fields can be concatenated
--count-mode valid-only \ # Items excluded by the length filter are not counted
--alternate-order True \ # Reverse the order for each repetition; has no effect when running only one round
--output-format multi-turn\ # Multi-turn conversation output format
--rounds 4 # Number of conversation turns
```

## 3. After Processing: Conversation Data Format
The processed JSON dataset has the format shown below. This simplified example contains two conversation turns:
```
[
  {
    "id": "id1",
    "conversations": [
      {
        "from": "human",
        "value": "Q1"
      },
      {
        "from": "gpt",
        "value": " "
      },
      {
        "from": "human",
        "value": "Q2"
      },
      {
        "from": "gpt",
        "value": " "
      }
    ]
  },
  {
    "id": "id3",
    "conversations": [
      {
        "from": "human",
        "value": "Q3"
      },
      {
        "from": "gpt",
        "value": " "
      },
      {
        "from": "human",
        "value": "Q4"
      },
      {
        "from": "gpt",
        "value": " "
      }
    ]
  }
]
```

## 4. Run the Test
```
# Find the configuration file path
ais_bench --models vllm_api_general_chat --datasets sharegpt_gen --mode perf --debug --num-warmups 0 --search

# Locate the test configuration file
│ --models    │ vllm_api_general_chat │ /benchmark/ais_bench/benchmark/configs/models/vllm_api/vllm_api_general_chat.py │

# Edit the configuration file
vim /benchmark/ais_bench/benchmark/configs/models/vllm_api/vllm_api_general_chat.py

# Locate the dataset configuration file and edit it
│ --datasets  │ sharegpt_gen          │ /benchmark/ais_bench/benchmark/configs/datasets/sharegpt/sharegpt_gen.py        │

# Set path to the actual path on the test machine, or place the converted file at the specified location.
sharegpt_datasets = [
dict(
abbr='sharegpt',
type=ShareGPTDataset,
disable_shuffle=True,
path='ais_bench/datasets/sharegpt/ShareGPT_V3_unfiltered_cleaned_split.json', # Dataset path; relative paths are resolved from the source root, and absolute paths are supported
reader_cfg=sharegpt_reader_cfg,
infer_cfg=sharegpt_infer_cfg,
eval_cfg=sharegpt_eval_cfg
)

# Run the multi-turn conversation test
ais_bench --models vllm_api_general_chat --datasets sharegpt_gen --mode perf --debug --num-warmups 0
# Warmup caches some of the dataset, so setting --num-warmups 0 is recommended to avoid affecting cache hits.
# To avoid the performance impact of running without warmup, you can warm up with another dataset before testing.
# After the test, restart the service before the next test to clear the existing cache and avoid affecting results.
```

## 4.5 Performance Results and Log Analysis
For explanations of performance metrics, see https://ais-bench-benchmark-rf.readthedocs.io/zh-cn/latest/base_tutorials/results_intro/performance_metric.html.

The following is an example set of long-document test results:

Table 1. Example long-text test results

|  | Prefix-Cache| MemCache |Repeat rate |Overall hit rate |Average TTFT (ms) |
|--|--|--|--|--|--|
| Test 2| Disabled | Disabled | 50% | 0% | 3057.8 |
| Test 2|Enabled  | Disabled | 50% | 17% | 2807 |
| Test 2| Enabled |Enabled  | 50% | 48.8% | 2070.8 |

Enabling more features increases KVCache capacity and gradually raises the cache hit rate. The main benefit is lower TTFT (Time To First Token). When the hit rate rises from zero to about 50%, TTFT can drop by more than 30%, and it continues to decrease as the hit rate increases.

The vLLM console logs show the current cache hit rate in real time. For details about vLLM's built-in hit-rate calculation, see the official documentation. By default, it reports the hit rate for the most recent 1,000 requests: https://docs.vllm.ai/en/v0.13.0/design/metrics/#prefix-cache-metrics

To estimate the MemCache memory required by a dataset, start a vLLM service once, find the log output showing memory usage, and make an approximate conversion. For example:

```
# Output formats may differ by version. Suppose a four-card run produces the following output:
Available KV cache memory: 9.21 GiB
GPU KV cache size: 150,800 tokens
# Conversion: 4*9.21/150≈0.25GB/k-token, or about 0.25 GB for every 1K tokens

# A two-card service may produce the following logs, which can also be used for the conversion:
Available memory: 36408063180,
GPU KV cache size: 277,760 tokens
# Conversion: (36*1000/1024)*2/277≈0.25GB/k-token, or about 0.25 GB for every 1K tokens
# Suppose the target dataset has 100 items, each with an expected 4K input tokens and 1K output tokens, for 5K tokens total. (Token count does not convert to text length at a 1:1 ratio.) Caching all items requires 500K tokens, so configure at least 500K*0.25GB/k-token=125GB.
```
To use all available memory, update the eviction thresholds in `mmc-meta.conf` as follows. If you do not use all available memory, configure a larger capacity so that the usable space below the thresholds meets the estimate above:
```
# Memory usage threshold that triggers eviction
ock.mmc.evict_threshold_high = 100
# Target free-space threshold after eviction
ock.mmc.evict_threshold_low = 90
```

After enabling MemCache, check whether the cache has enough space for a dataset by searching the MemCache logs for `Evicted` and checking whether any eviction occurred.
```
# Locate the MemCache log
/usr/local/memcache_hybrid/latest/aarch64-linux/logs/mmc-meta.log

# Search for the keyword Evicted; matching log entries indicate that eviction occurred
grep 'Evicted' mmc-meta.log | tail -n 50
```
