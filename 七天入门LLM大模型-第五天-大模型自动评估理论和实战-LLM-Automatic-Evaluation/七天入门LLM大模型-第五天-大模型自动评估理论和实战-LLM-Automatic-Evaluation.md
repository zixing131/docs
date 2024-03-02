

# 七天入门 LLM 大模型 | 第五天：大模型自动评估理论和实战--LLM Automatic Evaluation

  

  

#00 

  引 言 

  

背景  

大语言模型（LLM）评测是 LLM 开发和应用中的关键环节。目前评测方法可以分为人工评测和自动评测，其中，自动评测技术相比人工评测来讲，具有效率高、一致性好、可复现、鲁棒性好等特点，逐渐成为业界研究的重点。  

  

模型的自动评测技术可以分为 rule-based 和 model-based 两大类：

-   rule-based 方法：
    

-   benchmark 以客观题为主，例如多选题，被测的 LLM 通过理解 context/question，来指定最佳答案
    
-   解析 LLM 的 response，与标准答案做对比
    
-   计算 metric（accuracy、rouge、bleu 等）
    

-   model-based 方法：
    

-   裁判员模型（e.g. GPT-4、Claude、Expert Models/Reward models）
    
-   LLM Peer-examination
    

  

**内容提要**

-   LLM 自动评估理论
    

-   如何评估一个 LLM
    
-   自动评估的方法
    
-   常用的 benchmark
    
-   LLM 评估面临的问题和挑战
    

-   LLM 自动评估实战
    

-   LLMuses 自动评测框架介绍
    
-   基于客观题 benchmark 自动评估
    
-   基于专家模型的自动评估
    
-   LLM 推理性能评估
    

  

  

#01 

  LLM 评估的方法论 

  

![图片](assets/1709196845-09cf8b49eef7b9aa73b697ba620970b7.png)

  

  

1.1 如何评估一个 LLM

哪些维度？  

-   语义理解（Understanding）  
    
-   知识推理（Reasoning）
    
-   专业能力（e.g. coding、math）
    
-   应用能力（MedicalApps、AgentApps、AI-FOR-SCI ...）
    
-   指令跟随（Instruction Following）
    
-   鲁棒性（Robustness）
    
-   偏见（Bias）
    
-   幻觉（Hallucinations）
    
-   安全性（Safety）
    

![图片](assets/1709196845-754d2498e6de991e9f36ec61edbe7837.png)

> 例：GPT-4 vs LLaMA2-7B 能力维度对比评测

  

1.2 自动评估方法

#### 1.2.1 模型效果评估

-   **基准和指标（Benchmarks & Metrics）**
    

|     |     |     |     |
| --- | --- | --- | --- |
| 数据集 | 描述  | 评价指标 | 样例  |
| MMLU | MassiveMultitaskLanguageUnderstanding<br><br>一个多任务数据集，由各种学科的多项选择题组成。涵盖 STEM、人文、社科等领域。包括 57 个子任务，包括初等数学、美国历史、计算机科学、法律等等。 | Accuracy | Question: In 2016, about how many people in the United States were homeless?<br><br>A. 55,000<br><br>B. 550,000<br><br>C. 5,500,000<br><br>D. 55,000,000<br><br>  <br><br>Answer: B |
| TriviaQA | 阅读理解数据集，包含超过 65 万个问题 - 答案 - 证据三元组。其包括 95K 个问答对，由冷知识爱好者提供 + 独立收集的事实性文档撰写 | EM(ExactMatch)<br><br>  <br><br>F1 (word-level) | ![图片](assets/1709196845-185fcb719495ca01eb5705e05aee0f0e.png)<br><br>（问题 - 答案 - 证据文档） |
| MATH | 12500 道数学题，每道包含 step-by-step solution | Accuracy | ![图片](assets/1709196845-ed4659d192532a8efd2992a8a2b6410e.png) |
| HumanEval | HumanEval (Hand-Written Evaluation Set)<br><br>一个手写的问题解决数据集，要求根据给定的问题和代码模板，生成正确的代码片段。包含 164 个高质量的问题，涵盖五种编程语言：Python, C++, Java, Go, 和 JavaScript。 | pass@k | {<br><br>    "task\_id": "test/0",<br><br>    "prompt": "def return1():\\n",<br><br>    "canonical\_solution": "    return 1",<br><br>    "test": "def check(candidate):\\n    assert candidate() == 1",<br><br>    "entry\_point": "return1"<br><br>} |

  

-   Rule-based 自动评测
    

> 基本流程

![图片](assets/1709196845-ae9dc648141bf8dae6ebb9ae38233a6f.png)

  

-   **根据数据集原始 question 来构建 prompt**
    

> 示例 (few-shot)

![图片](assets/1709196845-99dd114d914cfd912caa3b18731b0e91.png)

> 示例：few-shot with CoT

```plain
# Examples in BBH

Evaluate the result of a random Boolean expression.

Q: not ( ( not not True ) ) is
A: Let's think step by step.
Remember that (i) expressions inside brackets are always evaluated first and that (ii) the order of operations from highest priority to lowest priority is "not", "and", "or", respectively.
We first simplify this expression "Z" as follows: "Z = not ( ( not not True ) ) = not ( ( A ) )" where "A = not not True".
Let's evaluate A: A = not not True = not (not True) = not False = True.
Plugging in A, we get: Z = not ( ( A ) ) = not ( ( True ) ) = not True = False. So the answer is False.

Q: True and False and not True and True is
A: Let's think step by step.
Remember that (i) expressions inside brackets are always evaluated first and that (ii) the order of operations from highest priority to lowest priority is "not", "and", "or", respectively.
We first simplify this expression "Z" as follows: "Z = True and False and not True and True = A and B" where "A = True and False" and "B = not True and True".
Let's evaluate A: A = True and False = False.
Let's evaluate B: B = not True and True = not (True and True) = not (True) = False.
Plugging in A and B, we get: Z = A and B = False and False = False. So the answer is False.
```

  

-   **模型预测**
    

> Generate

```plain
# Demo -- model_genereate 直接生成 response

def model_generate(query: str, infer_cfg: dict) -> str:

    inputs = tokenizer.encode(query)
    input_ids = inputs['input_ids']
    ...

    # Process infer cfg (do_sample, top_k, top_p, temperature, special_tokens ...)
    generation_config = process_cfg(args)
    ...

    # Run inference
    output_ids = model.generate(
        input_ids=input_ids,
        attention_mask=attention_mask,
        generation_config=generation_config,
    )

    response = tokenizer.decode(output_ids, **decode_kwargs)
    return response
```

> Likelihood

```plain
# Demo -- model_call 方式计算 loglikelihood

# context + continuation 拼接，示例：
# Question：法国的首都是哪里？
# Choices：A.北京  B.巴黎  C.汉堡  D.纽约
  # pair-1: (ctx, cont) = (法国的首都是哪里？，A.北京)
  # pair-2: (ctx, cont) = (法国的首都是哪里？，B.巴黎)
  # pair-3: (ctx, cont) = (法国的首都是哪里？，C.汉堡)
  # pair-4: (ctx, cont) = (法国的首都是哪里？，D.纽约)
# Logits -->

def loglikelihood(self, inputs: list, infer_cfg: dict = None) -> list:
    # To predict one doc
    doc_ele_pred = []
    for ctx, continuation in inputs:

        # ctx_enc shape: [context_tok_len]  cont_enc shape: [continuation_tok_len]
        ctx_enc, cont_enc = self._encode_pair(ctx, continuation)

        inputs_tokens = torch.tensor(
            (ctx_enc.tolist() + cont_enc.tolist())[-(self.max_length + 1):][:-1],
            dtype=torch.long,
            device=self.model.device).unsqueeze(0)

        logits = self.model(inputs_tokens)[0]
        logits = torch.nn.functional.log_softmax(logits.float(), dim=-1)

        logits = logits[:, -len(cont_enc):, :]
        cont_enc = cont_enc.unsqueeze(0).unsqueeze(-1)
        logits = torch.gather(logits.cpu(), 2, cont_enc.cpu()).squeeze(-1)

        choice_score = float(logits.sum())
        doc_ele_pred.append(choice_score)

    # e.g. [-2.3, 1.1, -12.9, -9.2], length=len(choices)
    return doc_ele_pred
```

  

-   评价指标（Metrics）
    

-   WeightedAverageAccuracy  加权平均准确率
    
-   Perplexity 困惑度
    
-   Rouge (Recall-Oriented Understudy for Gisting Evaluation)
    
-   Bleu (Bilingual evaluation understudy)
    
-   ELO Rating System
    
-   PASS@K
    

  

-   Model-based 自动评测
    

![图片](assets/1709196845-1e887f6bb2008482a87e8550101c2608.png)

  

-   中心化评测
    

-   中心化评测模式下，裁判员模型只有一个，可靠性高，但容易收到裁判员模型的 bias 影响
    

-   去中心化评测
    

-   去中心化评测方式，要求模型之间做 peer-examination
    
-   特点是公平性好，但计算量大，且鲁棒性不高
    

> 裁判员模型

-   GPT-4、Claude、Qwen-Max 等 （产品 APIs）
    
-   PandLM、Auto-J (tuned from LLM, like LLaMA)
    
-   Reward models (Ranking learning)
    

  

  

-   Chatbot Arena - 竞技场模式
    

-   Battle count of each combination of models,  from LMSYS
    
    ![图片](assets/1709196845-c1377ef93ba9149c1e68848146be1a2d.png)
    
-   Fraction of Model A wins for all non-tied A vs. B battles,  from LMSYS
    
    ![图片](assets/1709196845-db3bdd63db20bd7556031dbf3fc3b848.png)
    

  

-   LLM 指令攻防
    

-   指令诱导  (诱导模型输出目标答案，from SuperCLUE)
    
    ![图片](assets/1709196845-42f82268f8caa2eac625906fc7f76d01.png)
    
      
    
-   有害指令注入 (将真实有害意图注入到 prompt 中， from SuperCLUE)
    
    ![图片](assets/1709196845-37d80127b73f829661c0d17b431e0b18.svg)
    

  

#### 1.2.2 模型性能评估

> model serving performance evaluation

|     |     |
| --- | --- |
| 指标名称 | 说明  |
| Time | 测试总时间（时间单位均为秒） |
| Expected number of requests | 期望发送的请求数，和 prompt 文件以及期望 number 有关 |
| concurrency | 并发数 |
| completed | 完成的请求数 |
| succeed | 成功请求数 |
| failed | 失败请求数 |
| qps | 平均 qps |
| latency | 平均 latency |
| time to first token | 平均首包延迟 |
| throughput | output tokens / seconds<br><br>平均每秒输出 token 数 |
| time per output token | 平均生成一个 token 需要的时间<br><br>总 output\_tokens/总时间 |
| package per request | 平均每个请求的包数 |
| time per package | 平均每包时间 |
| input tokens per request | 平均每个请求的输入 token 数 |
| output tokens per request | 平均每个请求输出 token 数 |

  

**1.3 问题和挑战**

1.3.1 基准失效&数据泄露

-   静态数据集与快速演进的 LLM 能力形成 GAP，导致基准失效
    
-   公开的 benchmark 被泄露到 LLM 的开发 PT、CPT、SFT 等开发环节
    

> 解决思路： 动态数据集

![图片](assets/1709196845-dbe12ae970df66c00ef9deaef23380b0.png)

  

![图片](assets/1709196845-37d80127b73f829661c0d17b431e0b18.svg)

  

1.3.2 裁判员模型的能力上限

-   裁判员模型的存在明显的能力边界，很难胜任更多场景、更强模型的评测工作
    
-   泛化性问题
    
-   LLM 幻觉的诊断问题
    

  

  

#02 

  LLM 评估实战 

  

LLMuses 框架--轻量化、端到端的大模型自动评估框架

**GitHub:** 

https://github.com/modelscope/llmuses

  

框架特性

-   预置常用的测试基准，包括：MMLU、C-Eval、GSM8K、ARC、HellaSwag、TruthfulQA、MATH、HumanEval、BBH、GeneralQA 等
    
-   常用评估指标（metrics）的实现
    
-   统一 model 接入，兼容多个系列模型的 generate、chat 接口
    
-   客观题自动评估
    
-   使用专家模型实现复杂任务的自动评估
    
-   竞技场模式 (Arena）
    
-   评估报告生成与可视化
    
-   LLM 性能评测（Performance Evaluation）
    

  

环境安装

```plain
# 1. 代码下载
git clone git@github.com:modelscope/llmuses.git

# 2. 安装依赖
cd llmuses/
pip install -r requirements/requirements.txt
pip install -e .
```

  

2.1 简单评测

```plain
# 指定模型和数据集
python llmuses/run.py --model ZhipuAI/chatglm3-6b --datasets ceval --outputs ./outputs/test --limit 10
```

  

-   \--model: ModelScope 模型 id
    
     (https://modelscope.cn/models/ZhipuAI/chatglm3-6b/summary) ，也可以是模型的本地路径
    
-   \--datasets: 数据集的 id
    
-   \--limit: （每个 sub-task）最大评测样本数
    

，时长 00:38

  

2.2 带参数评测

```plain
python llmuses/run.py --model ZhipuAI/chatglm3-6b --outputs ./outputs/test2 --model-args revision=v1.0.2,precision=torch.float16,device_map=auto --datasets arc --limit 10
```

-   \--model-args: 模型参数，以逗号分隔，key=value 形式
    
-   \--datasets: 数据集名称，参考下文\`数据集列表\`章节
    
-   \--mem-cache: 是否使用内存缓存，若开启，则已经跑过的数据会自动缓存，并持久化到本地磁盘
    
-   \--limit: 每个 subset 最大评估数据量
    

，时长 00:15

  

2.3 竞技场模式--Single mode

Single mode，使用专家模型（GPT-4）对待测 LLM 进行打分

```plain
# Example
python llmuses/run_arena.py --c registry/config/cfg_single.yaml --dry-run
```

，时长 00:33

  

2.4 竞技场模式--Baseline mode

Baseline mode，选定 baseline 模型，其它待测 LLM 与该模型进行对比

```plain
# Example
python llmuses/run_arena.py --dry-run --c registry/config/cfg_pairwise_baseline.yaml
```

，时长 00:43

  

2.5 竞技场模式--Pairwise mode

Pairwise mode，待测 LLM 两两组合进行对弈

```plain
python llmuses/run_arena.py -c registry/config/cfg_arena.yaml --dry-run
```

，时长 00:59

![图片](assets/1709196845-26454a2c5b87245ada5e924ef2a77742.png)

  

  

2.6 效果评测报告

按照预定格式存放数据，使用 streamlit 启动可视化服务

```plain
# Usage:
streamlit run viz.py -- --review-file llmuses/registry/data/qa_browser/battle.jsonl --category-file llmuses/registry/data/qa_browser/category_mapping.yaml
```

  

-   报告可视化
    

![图片](assets/1709196845-93e2e4814764d1aa9558356f695d0aae.png)

  

-   Leaderboard:  
    
    https://modelscope.cn/leaderboard/58/ranking?type=free
    

![图片](assets/1709196845-f6b720ee3cde248192e0bc57bf944361.png)

  

2.7 模型性能评测（Perf Eval）

> 性能评测报告示例

![图片](assets/1709196845-f93bb65d3fa2c1a74dc34e8ab6e4294e.png)

![图片](assets/1709196845-8a764508be5555cca5e0bdb348510f11.png)
