# `ggml\examples\prompts\tokenize_huggingface.py`

```cpp
# 导入操作系统模块
import os
# 从transformers库中导入AutoTokenizer类
from transformers import AutoTokenizer

# 设置环境变量TOKENIZERS_PARALLELISM为"false"，禁用并行处理
os.environ['TOKENIZERS_PARALLELISM'] = "false"

# 定义一个列表，包含了多个Hugging Face模型仓库的名称
list_repo_hf  = ["databricks/dolly-v2-3b",           # dolly-v2 (3b, 7b, 12b models share the same tokenizer)
                 "gpt2",                             # gpt-2 (gpt2-xl, gpt2-large share the same tokenizer)
                 "uer/gpt2-chinese-cluecorpussmall", # gpt-2-chinese
                 "EleutherAI/gpt-j-6b",              # gpt-j
                 "EleutherAI/gpt-neox-20b",          # gpt-neox
                 "EleutherAI/polyglot-ko-1.3b",      # gpt-neox (polyglot-ko 5.8b and 12.8b share the same tokenizer")
                 "rinna/japanese-gpt-neox-3.6b",     # gpt-neox
                 # mpt-7b (uses gpt-neox-20b tokenizer)
                 "replit/replit-code-v1-3b",         # replit
                 "bigcode/starcoder",                # starcoder (huggingface-cli login required)
                 "openai/whisper-tiny"               # whisper (base, large, large-v2 share the same tokenizer)
                 ]

# 定义一个字典，将模型仓库名称映射为对应的GGML模型名称
repo2ggml     = {"databricks/dolly-v2-3b"           : "dolly-v2",
                 "gpt2"                             : "gpt-2",
                 "uer/gpt2-chinese-cluecorpussmall" : "gpt-2-chinese",
                 "EleutherAI/gpt-j-6b"              : "gpt-j",
                 "EleutherAI/gpt-neox-20b"          : "gpt-neox",
                 "EleutherAI/polyglot-ko-1.3b"      : "polyglot-ko",
                 "rinna/japanese-gpt-neox-3.6b"     : "gpt-neox-japanese",
                 "replit/replit-code-v1-3b"         : "replit",
                 "bigcode/starcoder"                : "starcoder",
                 "openai/whisper-tiny"              : "whisper"}
# 仓库名称到语言的映射字典
repo2language = {"databricks/dolly-v2-3b"           : "english",
                 "gpt2"                             : "english",
                 "uer/gpt2-chinese-cluecorpussmall" : "chinese",
                 "EleutherAI/gpt-j-6b"              : "english",
                 "EleutherAI/gpt-neox-20b"          : "english",
                 "EleutherAI/polyglot-ko-1.3b"      : "korean",
                 "rinna/japanese-gpt-neox-3.6b"     : "japanese",
                 "replit/replit-code-v1-3b"         : "english",
                 "bigcode/starcoder"                : "english",
                 "openai/whisper-tiny"              : "english"}

# 分隔符
delimeter = ": "
# 测试句子列表
test_sentences = []
# 读取测试用例文件
with open("test-cases.txt", "r") as f:
    # 去除每行末尾的换行符
    lines = [l.rstrip() for l in f.readlines()]
    # 遍历每行
    for l in lines:
        # 如果存在分隔符
        if delimeter in l:
            # 获取语言和句子
            language = l[:l.index(delimeter)]
            sentence = l[l.index(delimeter) + len(delimeter):]
            # 将语言转换为小写，并添加到测试句子列表中
            test_sentences.append((language.lower(), sentence))

# 遍历仓库列表
for repo in list_repo_hf:

    # 获取目标语言
    target_language = repo2language[repo]

    # 从预训练模型中加载分词器
    tokenizer = AutoTokenizer.from_pretrained(repo, trust_remote_code=True)

    # 分词结果列表
    tokens_hf = []
    # 遍历测试句子
    for language, sentence in test_sentences:
        # 如果语言与目标语言相同
        if language == target_language:
            # 将句子转换为 token id，并添加到分词结果列表中
            tokens = tokenizer.convert_tokens_to_ids(tokenizer.tokenize(sentence))
            tokens_hf.append((sentence, tokens))

    # 保存结果到文本文件
    save_txt = repo2ggml[repo] + ".txt"
    with open(save_txt, "w") as f:
        # 将句子和对应的 token id 写入文件
        f.writelines([sentence + " => " + ",".join(str(t) for t in tokens) + "\n" for sentence, tokens in tokens_hf])
```