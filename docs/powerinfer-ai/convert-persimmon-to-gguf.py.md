# `PowerInfer\convert-persimmon-to-gguf.py`

```
# 导入 torch 库
import torch
# 导入 os 库
import os
# 从 pprint 库中导入 pprint 函数
from pprint import pprint
# 导入 sys 库
import sys
# 从 argparse 库中导入 ArgumentParser 类
import argparse
# 从 pathlib 库中导入 Path 类
from pathlib import Path
# 从 sentencepiece 库中导入 SentencePieceProcessor 类
from sentencepiece import SentencePieceProcessor
# 如果环境变量中没有 NO_LOCAL_GGUF，则将 gguf-py 目录添加到 sys.path 中
if 'NO_LOCAL_GGUF' not in os.environ:
    sys.path.insert(1, str(Path(__file__).parent / 'gguf-py'))
# 导入 gguf 库

import gguf

# 定义函数 _flatten_dict，用于将嵌套字典扁平化为一级字典
def _flatten_dict(dct, tensors, prefix=None):
    # 断言 dct 是字典类型
    assert isinstance(dct, dict)
    # 遍历字典的键
    for key in dct.keys():
        # 如果 prefix 不为 None，则新的前缀为 prefix + '.' + key，否则为 key
        new_prefix = prefix + '.' + key if prefix is not None else key
        # 如果字典的值是 torch.Tensor 类型，则将其添加到 tensors 中
        if isinstance(dct[key], torch.Tensor):
            tensors[new_prefix] = dct[key]
        # 如果字典的值是字典类型，则递归调用 _flatten_dict 函数
        elif isinstance(dct[key], dict):
            _flatten_dict(dct[key], tensors, new_prefix)
        # 如果字典的值不是上述类型，则抛出 ValueError
        else:
            raise ValueError(type(dct[key]))
    # 返回 None
    return None

# 定义函数 _get_sentencepiece_tokenizer_info，用于获取 sentencepiece tokenizer 的信息
def _get_sentencepiece_tokenizer_info(dir_model: Path):
    # 构建 tokenizer_path
    tokenizer_path = dir_model / 'adept_vocab.model'
    # 打印信息
    print('gguf: getting sentencepiece tokenizer from', tokenizer_path)
    # 创建 SentencePieceProcessor 对象
    tokenizer = SentencePieceProcessor(str(tokenizer_path))
    # 打印信息
    print('gguf: adding tokens')
    # 初始化列表 tokens、scores、toktypes
    tokens: list[bytes] = []
    scores: list[float] = []
    toktypes: list[int] = []

    # 遍历 tokenizer 的词汇表
    for i in range(tokenizer.vocab_size()):
        text: bytes
        score: float

        # 获取词汇表中 id 对应的词和分数
        piece = tokenizer.id_to_piece(i)
        text = piece.encode("utf-8")
        score = tokenizer.get_score(i)

        # 初始化 toktype 为 1
        toktype = 1
        # 根据词的类型设置 toktype 的值
        if tokenizer.is_unknown(i):
            toktype = 2
        if tokenizer.is_control(i):
            toktype = 3
        if tokenizer.is_unused(i):
            toktype = 5
        if tokenizer.is_byte(i):
            toktype = 6

        # 将词、分数、类型添加到对应的列表中
        tokens.append(text)
        scores.append(score)
        toktypes.append(toktype)
        pass
    # 返回 tokens、scores、toktypes 列表
    return tokens, scores, toktypes

# 定义主函数 main
def main():
    # 创建 ArgumentParser 对象，设置描述信息
    parser = argparse.ArgumentParser(description="Convert a Persimmon model from Adept (e.g. Persimmon 8b chat) to a GGML compatible file")
    # 添加命令行参数 --outfile，设置类型为 Path，帮助信息为 path to write to; default: based on input
    parser.add_argument("--outfile", type=Path, help="path to write to; default: based on input")
    # 添加命令行参数，指定 persimmon 检查点 .pt 文件的路径
    parser.add_argument("--ckpt-path", type=Path, help="path to persimmon checkpoint .pt file")
    # 添加命令行参数，指定包含模型的目录，例如 8b_chat_model_release
    parser.add_argument("--model-dir", type=Path, help="directory containing model e.g. 8b_chat_model_release")
    # 添加命令行参数，指定 adept-inference 代码目录的路径
    parser.add_argument("--adept-inference-dir", type=str, help="path to adept-inference code directory")
    # 解析命令行参数
    args = parser.parse_args()
    # 将 adept-inference 目录添加到系统路径中
    sys.path.append(str(args.adept_inference_dir))
    # 加载 persimmon 检查点文件
    persimmon_model = torch.load(args.ckpt_path)
    # 获取 persimmon 模型的参数
    hparams = persimmon_model['args']
    # 打印模型参数
    pprint(hparams)
    # 创建空字典 tensors
    tensors = {}
    # 将 persimmon 模型的权重展平到 tensors 字典中
    _flatten_dict(persimmon_model['model'], tensors, None)

    # 设置模型架构为 PERSIMMON
    arch = gguf.MODEL_ARCH.PERSIMMON
    # 创建 GGUFWriter 对象，指定输出文件和模型架构名称
    gguf_writer = gguf.GGUFWriter(args.outfile, gguf.MODEL_ARCH_NAMES[arch])

    # 设置 block_count 为模型参数中的层数
    block_count = hparams.num_layers
    # 设置 head_count 为模型参数中的注意力头数
    head_count = hparams.num_attention_heads
    # 设置 head_count_kv 为注意力头数
    head_count_kv = head_count
    # 设置 ctx_length 为模型参数中的序列长度
    ctx_length = hparams.seq_length
    # 设置 hidden_size 为模型参数中的隐藏层大小
    hidden_size = hparams.hidden_size

    # 添加模型名称到 GGUFWriter
    gguf_writer.add_name('persimmon-8b-chat')
    # 添加上下文长度到 GGUFWriter
    gguf_writer.add_context_length(ctx_length)
    # 添加嵌入长度到 GGUFWriter
    gguf_writer.add_embedding_length(hidden_size)
    # 添加块数量到 GGUFWriter
    gguf_writer.add_block_count(block_count)
    # 添加前馈网络长度到 GGUFWriter
    gguf_writer.add_feed_forward_length(hparams.ffn_hidden_size)
    # 添加绳索维度数量到 GGUFWriter
    gguf_writer.add_rope_dimension_count(hidden_size // head_count)
    # 添加注意力头数到 GGUFWriter
    gguf_writer.add_head_count(head_count)
    # 添加键值注意力头数到 GGUFWriter
    gguf_writer.add_head_count_kv(head_count_kv)
    # 添加绳索频率基数到 GGUFWriter
    gguf_writer.add_rope_freq_base(hparams.rotary_emb_base)
    # 添加层归一化 epsilon 到 GGUFWriter
    gguf_writer.add_layer_norm_eps(hparams.layernorm_epsilon)

    # 获取句子片段分词器信息，包括 tokens, scores, toktypes
    tokens, scores, toktypes = _get_sentencepiece_tokenizer_info(args.model_dir)
    # 添加分词器模型到 GGUFWriter
    gguf_writer.add_tokenizer_model('llama')
    # 添加分词器模型的 token 列表到 GGUFWriter
    gguf_writer.add_token_list(tokens)
    # 添加分词器模型的 token 分数到 GGUFWriter
    gguf_writer.add_token_scores(scores)
    # 添加分词器模型的 token 类型到 GGUFWriter
    gguf_writer.add_token_types(toktypes)
    # 添加起始 token id 到 GGUFWriter
    gguf_writer.add_bos_token_id(71013)
    # 添加结束 token id 到 GGUFWriter
    gguf_writer.add_eos_token_id(71013)

    # 获取模型张量名称映射
    tensor_map = gguf.get_tensor_name_map(arch, block_count)
    # 打印张量名称映射
    print(tensor_map)
    # 遍历张量字典的键值对
    for name in tensors.keys():
        # 获取当前张量的数据
        data = tensors[name]
        # 如果张量名以".self_attention.rotary_emb.inv_freq"结尾，则跳过当前循环
        if name.endswith(".self_attention.rotary_emb.inv_freq"):
            continue
        # 保存当前数据的数据类型
        old_dtype = data.dtype
        # 将数据类型转换为torch.float32，然后压缩并转换为numpy数组
        data = data.to(torch.float32).squeeze().numpy()
        # 根据原始张量名获取新的张量名
        new_name = tensor_map.get_name(name, try_suffixes = (".weight", ".bias"))
        # 如果获取不到新的张量名，则打印错误信息并退出程序
        if new_name is None:
            print("Can not map tensor '" + name + "'")
            sys.exit()
        # 获取数据的维度，并打印新的张量名、维度、原始数据类型和转换后的数据类型
        n_dims = len(data.shape)
        print(new_name + ", n_dims = " + str(n_dims) + ", " + str(old_dtype) + " --> " + str(data.dtype))
        # 将新的张量名和数据添加到gguf_writer中
        gguf_writer.add_tensor(new_name, data)
    # 打印信息，写入gguf文件头部
    print("gguf: write header")
    gguf_writer.write_header_to_file()
    # 打印信息，写入gguf元数据
    print("gguf: write metadata")
    gguf_writer.write_kv_data_to_file()
    # 打印信息，写入gguf张量数据
    print("gguf: write tensors")
    gguf_writer.write_tensors_to_file()

    # 关闭gguf_writer
    gguf_writer.close()

    # 打印成功导出模型的信息
    print(f"gguf: model successfully exported to '{args.outfile}'")
    # 打印空行
    print("")
# 如果当前模块被直接执行，则调用 main() 函数
if __name__ == '__main__':
    main()
```