# `PowerInfer\convert-persimmon-to-gguf.py`

```
# 导入 torch 模块
import torch
# 导入 os 模块
import os
# 从 pprint 模块中导入 pprint 函数
from pprint import pprint
# 导入 sys 模块
import sys
# 从 argparse 模块中导入 ArgumentParser 类
import argparse
# 从 pathlib 模块中导入 Path 类
from pathlib import Path
# 从 sentencepiece 模块中导入 SentencePieceProcessor 类
from sentencepiece import SentencePieceProcessor
# 如果环境变量中没有设置 NO_LOCAL_GGUF，则将 gguf-py 目录添加到 sys.path 中
if 'NO_LOCAL_GGUF' not in os.environ:
    sys.path.insert(1, str(Path(__file__).parent / 'gguf-py'))
# 导入 gguf 模块
import gguf

# 定义一个函数 _flatten_dict，用于将嵌套字典中的 torch.Tensor 对象扁平化为一级字典
def _flatten_dict(dct, tensors, prefix=None):
    # 断言 dct 是一个字典
    assert isinstance(dct, dict)
    # 遍历字典的键
    for key in dct.keys():
        # 如果有前缀，则将新前缀设置为前缀加上当前键，否则设置为当前键
        new_prefix = prefix + '.' + key if prefix is not None else key
        # 如果当前值是 torch.Tensor 对象，则将其添加到 tensors 字典中
        if isinstance(dct[key], torch.Tensor):
            tensors[new_prefix] = dct[key]
        # 如果当前值是字典，则递归调用 _flatten_dict 函数
        elif isinstance(dct[key], dict):
            _flatten_dict(dct[key], tensors, new_prefix)
        # 如果当前值不是 torch.Tensor 对象也不是字典，则不做处理
# 如果字典中指定键的值不是预期的类型，则引发 ValueError
def _get_sentencepiece_tokenizer_info(dir_model: Path):
    # 指定 tokenizer_path 为 dir_model 下的 'adept_vocab.model' 文件
    tokenizer_path = dir_model / 'adept_vocab.model'
    # 打印获取 sentencepiece tokenizer 的信息
    print('gguf: getting sentencepiece tokenizer from', tokenizer_path)
    # 使用 tokenizer_path 创建 SentencePieceProcessor 对象
    tokenizer = SentencePieceProcessor(str(tokenizer_path))
    # 打印添加 tokens 的信息
    print('gguf: adding tokens')
    # 初始化 tokens、scores 和 toktypes 列表
    tokens: list[bytes] = []
    scores: list[float] = []
    toktypes: list[int] = []

    # 遍历 tokenizer 的词汇表大小
    for i in range(tokenizer.vocab_size()):
        # 初始化 text 和 score
        text: bytes
        score: float

        # 获取指定 id 对应的 piece，并将其编码为 utf-8 格式的 bytes
        piece = tokenizer.id_to_piece(i)
        text = piece.encode("utf-8")
        # 获取指定 id 对应的 score
        score = tokenizer.get_score(i)
# 初始化 toktype 变量为 1
toktype = 1
# 如果当前字符是未知字符，则将 toktype 设置为 2
if tokenizer.is_unknown(i):
    toktype = 2
# 如果当前字符是控制字符，则将 toktype 设置为 3
if tokenizer.is_control(i):
    toktype = 3
# 如果当前字符是未使用的字符，则将 toktype 设置为 5
if tokenizer.is_unused(i):
    toktype = 5
# 如果当前字符是字节，则将 toktype 设置为 6
if tokenizer.is_byte(i):
    toktype = 6

# 将文本添加到 tokens 列表中
tokens.append(text)
# 将分数添加到 scores 列表中
scores.append(score)
# 将 toktype 添加到 toktypes 列表中
toktypes.append(toktype)
# 返回 tokens, scores, toktypes 列表
return tokens, scores, toktypes

# 主函数
def main():
    # 创建参数解析器对象
    parser = argparse.ArgumentParser(description="Convert a Persimmon model from Adept (e.g. Persimmon 8b chat) to a GGML compatible file")
    # 添加 --outfile 参数，指定输出文件路径，默认为基于输入文件的路径
    parser.add_argument("--outfile", type=Path, help="path to write to; default: based on input")
    # 添加 --ckpt-path 参数，指定 persimmon checkpoint .pt 文件的路径
    parser.add_argument("--ckpt-path", type=Path, help="path to persimmon checkpoint .pt file")
# 添加命令行参数，指定模型目录
parser.add_argument("--model-dir", type=Path, help="directory containing model e.g. 8b_chat_model_release")
# 添加命令行参数，指定adept-inference代码目录
parser.add_argument("--adept-inference-dir", type=str, help="path to adept-inference code directory")
# 解析命令行参数
args = parser.parse_args()
# 将adept-inference代码目录添加到系统路径中
sys.path.append(str(args.adept_inference_dir))
# 加载模型
persimmon_model = torch.load(args.ckpt_path)
# 获取模型参数
hparams = persimmon_model['args']
# 打印模型参数
pprint(hparams)
# 初始化tensors字典
tensors = {}
# 将模型参数展平成字典
_flatten_dict(persimmon_model['model'], tensors, None)

# 设置模型架构为PERSIMMON
arch = gguf.MODEL_ARCH.PERSIMMON
# 创建GGUFWriter对象
gguf_writer = gguf.GGUFWriter(args.outfile, gguf.MODEL_ARCH_NAMES[arch])

# 获取模型块数
block_count = hparams.num_layers
# 获取注意力头数
head_count = hparams.num_attention_heads
# 获取键值注意力头数
head_count_kv = head_count
# 获取上下文长度
ctx_length = hparams.seq_length
# 获取隐藏层大小
hidden_size = hparams.hidden_size

# 添加模型名称到GGUFWriter对象
gguf_writer.add_name('persimmon-8b-chat')
# 设置上下文长度
gguf_writer.add_context_length(ctx_length)
# 设置嵌入长度
gguf_writer.add_embedding_length(hidden_size)
# 设置块数量
gguf_writer.add_block_count(block_count)
# 设置前馈神经网络隐藏层大小
gguf_writer.add_feed_forward_length(hparams.ffn_hidden_size)
# 设置绳索维度数量
gguf_writer.add_rope_dimension_count(hidden_size // head_count)
# 设置头数量
gguf_writer.add_head_count(head_count)
# 设置键值头数量
gguf_writer.add_head_count_kv(head_count_kv)
# 设置绳索频率基数
gguf_writer.add_rope_freq_base(hparams.rotary_emb_base)
# 设置层归一化 epsilon
gguf_writer.add_layer_norm_eps(hparams.layernorm_epsilon)

# 获取句子分词器信息
tokens, scores, toktypes = _get_sentencepiece_tokenizer_info(args.model_dir)
# 添加分词器模型
gguf_writer.add_tokenizer_model('llama')
# 添加分词列表
gguf_writer.add_token_list(tokens)
# 添加分词得分
gguf_writer.add_token_scores(scores)
# 添加分词类型
gguf_writer.add_token_types(toktypes)
# 添加起始标记 ID
gguf_writer.add_bos_token_id(71013)
# 添加结束标记 ID
gguf_writer.add_eos_token_id(71013)

# 获取张量名称映射
tensor_map = gguf.get_tensor_name_map(arch, block_count)
# 打印张量名称映射
print(tensor_map)
# 遍历张量字典的键值对
for name in tensors.keys():
    # 获取张量数据
    data = tensors[name]
    # 如果文件名以".self_attention.rotary_emb.inv_freq"结尾，则跳过本次循环
    if name.endswith(".self_attention.rotary_emb.inv_freq"):
        continue
    # 保存原始数据类型
    old_dtype = data.dtype
    # 将数据类型转换为torch.float32，然后压缩并转换为numpy数组
    data = data.to(torch.float32).squeeze().numpy()
    # 获取新的张量名称
    new_name = tensor_map.get_name(name, try_suffixes = (".weight", ".bias"))
    # 如果新名称为空，则打印错误信息并退出程序
    if new_name is None:
        print("Can not map tensor '" + name + "'")
        sys.exit()
    # 获取数据的维度，并打印原始数据类型和转换后的数据类型
    n_dims = len(data.shape)
    print(new_name + ", n_dims = " + str(n_dims) + ", " + str(old_dtype) + " --> " + str(data.dtype))
    # 将张量数据添加到gguf_writer中
    gguf_writer.add_tensor(new_name, data)
# 打印gguf写入头部信息
print("gguf: write header")
gguf_writer.write_header_to_file()
# 打印gguf写入元数据信息
print("gguf: write metadata")
gguf_writer.write_kv_data_to_file()
# 打印gguf写入张量信息
print("gguf: write tensors")
gguf_writer.write_tensors_to_file()
# 关闭 gguf_writer 对象
gguf_writer.close()

# 打印成功导出模型的消息，包括输出文件名
print(f"gguf: model successfully exported to '{args.outfile}'")
# 打印空行
print("")

# 如果当前脚本被直接执行，则调用 main 函数
if __name__ == '__main__':
    main()
```