# `PowerInfer\convert-hf-to-powerinfer-gguf.py`

```
#!/usr/bin/env python3

from __future__ import annotations  # 导入未来版本的注解特性
from abc import ABC, abstractmethod  # 导入抽象基类和抽象方法

import argparse  # 导入命令行参数解析模块
import contextlib  # 导入上下文管理模块
import json  # 导入 JSON 模块
import os  # 导入操作系统模块
import re  # 导入正则表达式模块
import struct  # 导入 struct 模块
import sys  # 导入系统相关的功能
from enum import IntEnum  # 导入枚举类型 IntEnum
from pathlib import Path  # 导入路径操作模块
from typing import TYPE_CHECKING, Any, ContextManager, Iterator, Optional, cast  # 导入类型提示相关模块

import numpy as np  # 导入 NumPy 数学计算库
import torch  # 导入 PyTorch 深度学习库
import torch.nn as tnn  # 导入 PyTorch 神经网络模块

from dataclasses import dataclass  # 导入数据类装饰器

if TYPE_CHECKING:  # 如果是类型检查
    from torch import Tensor  # 从 torch 模块中导入 Tensor 类型

if "NO_LOCAL_GGUF" not in os.environ:  # 如果环境变量中没有 NO_LOCAL_GGUF
    sys.path.insert(1, str(Path(__file__).parent / "gguf-py"))  # 将 gguf-py 目录添加到系统路径中
import gguf  # 导入 gguf 模块


###### MODEL DEFINITIONS ######


class SentencePieceTokenTypes(IntEnum):  # 定义 SentencePieceTokenTypes 枚举类
    NORMAL = 1  # 普通类型
    UNKNOWN = 2  # 未知类型
    CONTROL = 3  # 控制类型
    USER_DEFINED = 4  # 用户定义类型
    UNUSED = 5  # 未使用类型
    BYTE = 6  # 字节类型


class ReluMLP(tnn.Module):  # 定义 ReluMLP 类，继承自 tnn.Module
    def __init__(self, input_dim: int, hidden_dim: int, output_dim: int):  # 初始化方法
        super(ReluMLP, self).__init__()  # 调用父类的初始化方法
        self.fc1 = tnn.Linear(input_dim, hidden_dim, bias=False)  # 创建全连接层 fc1
        self.relu = tnn.ReLU()  # 创建 ReLU 激活函数
        self.fc2 = tnn.Linear(hidden_dim, output_dim, bias=False)  # 创建全连接层 fc2

    def forward(self, x):  # 前向传播方法
        x = self.fc1(x)  # 全连接层 fc1 前向传播
        x = self.relu(x)  # ReLU 激活函数
        x = self.fc2(x)  # 全连接层 fc2 前向传播
        return x  # 返回结果

    @staticmethod
    def from_file(model_file: Path):  # 从文件中加载模型的静态方法
        model = torch.load(model_file, map_location="cpu")  # 加载模型文件
        hidden_size, input_size = model.get("fc1.weight").shape  # 获取隐藏层大小和输入层大小
        output_size, _ = model.get("fc2.weight").shape  # 获取输出层大小
        mlp = ReluMLP(input_size, hidden_size, output_size)  # 创建 ReluMLP 对象
        mlp.load_state_dict(model)  # 加载模型参数
        return mlp  # 返回模型对象


class Model(ABC):  # 定义 Model 抽象基类
    """Base class for model conversion"""  # 模型转换的基类

    def __init__(  # 初始化方法
        self,
        dir_model: Path,  # 模型目录路径
        dir_mlp_pred: Path,  # MLP 预测目录路径
        ftype: int,  # 文件类型
        fname_out: Path,  # 输出文件名路径
        is_big_endian: bool,  # 是否大端序
    # 初始化函数，设置对象的属性值
    def __init__(
        self, dir_model, dir_mlp_pred, ftype, fname_out, is_big_endian
    ):
        # 设置模型目录属性
        self.dir_model = dir_model
        # 设置 MLP 预测目录属性
        self.dir_mlp_pred = dir_mlp_pred
        # 设置文件类型属性
        self.ftype = ftype
        # 设置输出文件名属性
        self.fname_out = fname_out
        # 设置是否大端属性
        self.is_big_endian = is_big_endian
        # 根据是否大端设置字节顺序属性
        self.endianess = (
            gguf.GGUFEndian.BIG if is_big_endian else gguf.GGUFEndian.LITTLE
        )
        # 判断模型是否使用安全张量
        self.is_safetensors = self._is_model_safetensors()
        # 统计模型部分数量
        self.num_parts = Model.count_model_parts(
            self.dir_model, ".safetensors" if self.is_safetensors else ".bin"
        )
        # 获取模型部分名称列表
        self.part_names = self._get_part_names()
        # 加载模型超参数
        self.hparams = Model.load_hparams(self.dir_model)
        # 获取模型架构
        self.model_arch = self._get_model_architecture()
        # 创建 GGUFWriter 对象
        self.gguf_writer = gguf.GGUFWriter(
            fname_out, gguf.MODEL_ARCH_NAMES[self.model_arch], endianess=self.endianess, use_temp_file = False
        )

    # 设置词汇表
    def set_vocab(self):
        self._set_vocab_gpt2()
    # 返回一个迭代器，迭代器的元素是元组，包含模型层名称和张量数据
    def get_tensors(self) -> Iterator[tuple[str, Tensor]]:
        # 遍历获取 MLP 部分层的名称和部分名称
        for model_layer, part_name in self._get_mlp_part_layer_names():
            # 打印加载 MLP 部分的信息
            print(f"gguf: loading mlp part '{part_name}'")
            # 从文件中加载 ReluMLP 模型
            mlp_model = ReluMLP.from_file(self.dir_mlp_pred / part_name)
            # 遍历 MLP 模型的状态字典，返回模型层名称和张量数据
            for name, data in mlp_model.state_dict().items():
                yield f"blk.{model_layer}.{name}", data

        # 遍历部分名称列表
        for part_name in self.part_names:
            # 打印加载模型部分的信息
            print(f"gguf: loading model part '{part_name}'")
            # 定义上下文管理器
            ctx: ContextManager[Any]
            # 如果使用安全张量
            if self.is_safetensors:
                # 导入安全张量模块，使用安全打开上下文管理器
                from safetensors import safe_open
                ctx = cast(
                    ContextManager[Any],
                    safe_open(self.dir_model / part_name, framework="pt", device="cpu"),
                )
            else:
                # 否则使用普通的上下文管理器，加载模型部分
                ctx = contextlib.nullcontext(
                    torch.load(self.dir_model / part_name, map_location="cpu")
                )

            # 进入上下文管理器，获取模型部分的张量数据
            with ctx as model_part:
                # 遍历模型部分的键
                for name in model_part.keys():
                    # 如果使用安全张量，则获取张量数据，否则直接获取数据
                    data = (
                        model_part.get_tensor(name)
                        if self.is_safetensors
                        else model_part[name]
                    )
                    # 返回张量名称和数据
                    yield name, data

    # 抽象方法
    @abstractmethod
    def set_gguf_parameters(self):
        pass
        # 设置 GGUF 参数

        # 将模型名称添加到 GGUF 写入器
        # self.gguf_writer.add_name(self.dir_model.name)

        # 将块数量添加到 GGUF 写入器
        # self.gguf_writer.add_block_count(
        #     self.hparams.get(
        #         "n_layers",
        #         self.hparams.get("num_hidden_layers", self.hparams.get("n_layer")),
        #     )
        # )

        # 如果存在最大位置嵌入参数，则将其添加到 GGUF 写入器
        # if (n_ctx := self.hparams.get("max_position_embeddings")) is not None:
        #     self.gguf_writer.add_context_length(n_ctx)

        # 如果存在隐藏层大小参数，则将其添加到 GGUF 写入器
        # if (n_embd := self.hparams.get("hidden_size")) is not None:
        #     self.gguf_writer.add_embedding_length(n_embd)

        # 如果存在中间层大小参数，则将其添加到 GGUF 写入器
        # if (n_ff := self.hparams.get("intermediate_size")) is not None:
        #     self.gguf_writer.add_feed_forward_length(n_ff)

        # 如果存在注意力头数量参数，则将其添加到 GGUF 写入器
        # if (n_head := self.hparams.get("num_attention_head")) is not None:
        #     self.gguf_writer.add_head_count(n_head)

        # 将是否使用并行残差连接的参数添加到 GGUF 写入器
        # self.gguf_writer.add_parallel_residual(
        #     self.hparams.get("use_parallel_residual", True)
        # )

    @abstractmethod
    def write_tensors(self):
        pass
        # 抽象方法，用于写入张量数据

    def write(self):
        self.write_tensors()
        self.gguf_writer.write_header_to_file()
        self.gguf_writer.write_kv_data_to_file()
        self.gguf_writer.write_tensors_to_file()
        self.gguf_writer.close()
        # 写入张量数据，并将头部、键值数据和张量数据写入文件，然后关闭 GGUF 写入器

    def write_vocab(self):
        self.gguf_writer.write_header_to_file()
        self.gguf_writer.write_kv_data_to_file()
        self.gguf_writer.close()
        # 将头部和键值数据写入文件，然后关闭 GGUF 写入器

    @staticmethod
    def count_model_parts(dir_model: Path, prefix: str) -> int:
        num_parts = 0
        for filename in os.listdir(dir_model):
            if filename.endswith(prefix):
                num_parts += 1
        return num_parts
        # 统计模型部件的数量

    @staticmethod
    def load_hparams(dir_model):
        with open(dir_model / "config.json", "r", encoding="utf-8") as f:
            return json.load(f)
        # 加载模型的超参数
    # 根据模型架构返回相应的模型类
    def from_model_architecture(model_architecture):
        if model_architecture in ("FalconForCausalLM", "RWForCausalLM"):
            return FalconModel
        if model_architecture == "LlamaForCausalLM":
            return LlamaModel

        # 如果模型架构不在支持的列表中，则抛出未实现的错误
        raise NotImplementedError(f'Architecture "{model_architecture}" not supported!')

    # 检查模型是否包含安全张量
    def _is_model_safetensors(self) -> bool:
        return Model.count_model_parts(self.dir_model, ".safetensors") > 0

    # 获取MLP部分层的名称
    def _get_mlp_part_layer_names(self):
        """Returns a generator of (index, name) for MLP predictors of each model layer"""
        n_mlp_parts = Model.count_model_parts(self.dir_mlp_pred, ".pt")
        return ((n, f"model_{n}.pt") for n in range(n_mlp_parts))

    # 获取模型部分的名称
    def _get_part_names(self):
        if self.is_safetensors:
            if self.num_parts == 1:  # there's only one .safetensors file
                return ("model.safetensors",)
            return (
                f"model-{n:05}-of-{self.num_parts:05}.safetensors"
                for n in range(1, self.num_parts + 1)
            )
        else:
            if self.num_parts == 1:  # there's only one .bin file
                return ("pytorch_model.bin",)
            return (
                f"pytorch_model-{n:05}-of-{self.num_parts:05}.bin"
                for n in range(1, self.num_parts + 1)
            )

    # 获取模型架构
    def _get_model_architecture(self) -> gguf.MODEL_ARCH:
        arch = self.hparams["architectures"][0]
        if arch == "FalconForCausalLM":
            return gguf.MODEL_ARCH.FALCON
        if arch == "RWForCausalLM" or arch == "LlamaForCausalLM":
            return gguf.MODEL_ARCH.LLAMA

        # 如果模型架构不在支持的列表中，则抛出未实现的错误
        raise NotImplementedError(f'Architecture "{arch}" not supported!')

    # 转换张量键
    def _translate_tensor_key(
        self, key: str, try_suffixes=(".weight", ".bias")
    # 定义一个方法，接受一个字符串参数并返回一个可选的字符串
    ) -> Optional[str]:
        # 从超参数中获取块的数量，如果没有则使用默认值
        block_count = self.hparams.get(
            "n_layers",
            self.hparams.get("num_hidden_layers", self.hparams.get("n_layer")),
        )
        # 使用模型架构和块的数量获取张量名称映射
        tensor_map = gguf.get_tensor_name_map(self.model_arch, block_count)
        # 获取张量映射中与给定键相关的张量键
        arch_tensor_key = tensor_map.get_name(key, try_suffixes=try_suffixes)
        # 如果存在与给定键相关的张量键，则返回该张量键
        if arch_tensor_key is not None:
            return arch_tensor_key
        # 检查并处理 ReluMLP 层
        mlp_match = re.match(r"^blk\.\d+\.fc\d\.weight$", key)
        # 如果匹配成功，则返回匹配的字符串
        if mlp_match:
            return mlp_match.group(0)
        # 如果以上条件都不满足，则返回 None
        return None
    # 设置 GPT2 的词汇表
    def _set_vocab_gpt2(self):
        # 获取模型目录和超参数
        dir_model = self.dir_model
        hparams = self.hparams
        # 初始化 tokens 和 toktypes 列表
        tokens: list[bytearray] = []
        toktypes: list[int] = []

        # 导入 AutoTokenizer 类
        from transformers import AutoTokenizer  # type: ignore[attr-defined]

        # 从预训练模型目录创建 tokenizer 对象
        tokenizer = AutoTokenizer.from_pretrained(dir_model)
        # 获取词汇表大小，如果未指定则使用 tokenizer 的词汇表大小
        vocab_size = hparams.get("vocab_size", len(tokenizer.vocab))
        # 断言 tokenizer 的词汇表中的最大值小于指定的词汇表大小
        assert max(tokenizer.vocab.values()) < vocab_size

        # 创建反转的词汇表，将编码的 token 映射到 id
        reverse_vocab = {
            id_: encoded_tok for encoded_tok, id_ in tokenizer.vocab.items()
        }
        # 获取 tokenizer 的添加词汇表
        added_vocab = tokenizer.get_added_vocab()

        # 遍历词汇表大小
        for i in range(vocab_size):
            # 如果 i 不在反转的词汇表中
            if i not in reverse_vocab:
                # 创建并添加用户定义的 pad token 到 tokens 列表
                pad_token = f"[PAD{i}]".encode("utf-8")
                tokens.append(bytearray(pad_token))
                # 添加用户定义类型到 toktypes 列表
                toktypes.append(gguf.TokenType.USER_DEFINED)
            # 如果反转的词汇表中的 id 在添加的词汇表中
            elif reverse_vocab[i] in added_vocab:
                # 添加 token 到 tokens 列表
                tokens.append(reverse_vocab[i])
                # 如果添加的 token 是特殊的，则添加控制类型到 toktypes 列表，否则添加用户定义类型
                if tokenizer.added_tokens_decoder[i].special:
                    toktypes.append(gguf.TokenType.CONTROL)
                else:
                    toktypes.append(gguf.TokenType.USER_DEFINED)
            # 如果反转的词汇表中的 id 不在添加的词汇表中
            else:
                # 添加 token 到 tokens 列表
                tokens.append(reverse_vocab[i])
                # 添加正常类型到 toktypes 列表
                toktypes.append(gguf.TokenType.NORMAL)

        # 将 "gpt2" 添加到 gguf_writer 的 tokenizer 模型中
        self.gguf_writer.add_tokenizer_model("gpt2")
        # 将 tokens 列表添加到 gguf_writer 的 token 列表中
        self.gguf_writer.add_token_list(tokens)
        # 将 toktypes 列表添加到 gguf_writer 的 token 类型中
        self.gguf_writer.add_token_types(toktypes)

        # 创建特殊词汇表对象，加载合并
        special_vocab = gguf.SpecialVocab(dir_model, load_merges=True)
        # 将特殊词汇表添加到 gguf_writer 中
        special_vocab.add_to_gguf(self.gguf_writer)
# 定义 LlamaModel 类，继承自 Model 类
class LlamaModel(Model):
    # 设置词汇表
    def set_vocab(self):
        self._set_vocab_sentencepiece()

    # 设置 GGUF 参数
    def set_gguf_parameters(self, params: PredictorParams):
        # 添加名称为 "Llama" 的 GGUF 参数
        self.gguf_writer.add_name("Llama")
        # 添加上下文长度为 2048，不在 config.json 中
        self.gguf_writer.add_context_length(2048)
        # 添加嵌入长度为 self.hparams["hidden_size"]
        self.gguf_writer.add_embedding_length(self.hparams["hidden_size"])
        # 添加块数量为 self.hparams["num_hidden_layers"]
        self.gguf_writer.add_block_count(self.hparams["num_hidden_layers"])
        # 添加前馈长度为 self.hparams["intermediate_size"]
        self.gguf_writer.add_feed_forward_length(self.hparams["intermediate_size"])
        # 添加绳子维度数量为 self.hparams["hidden_size"] // self.hparams["num_attention_heads"]
        self.gguf_writer.add_rope_dimension_count(
            self.hparams["hidden_size"] // self.hparams["num_attention_heads"]
        )
        # 添加注意力头数量为 self.hparams["num_attention_heads"]
        self.gguf_writer.add_head_count(self.hparams["num_attention_heads"])
        # 添加键值头数量为 self.hparams["num_key_value_heads"]
        self.gguf_writer.add_head_count_kv(self.hparams["num_key_value_heads"])
        # 添加层归一化 RMS 衰减值为 self.hparams["rms_norm_eps"]
        self.gguf_writer.add_layer_norm_rms_eps(self.hparams["rms_norm_eps"])
        # 添加绳子频率基数为 self.hparams["rope_theta"]
        self.gguf_writer.add_rope_freq_base(self.hparams["rope_theta"])
        # 添加文件类型为 self.ftype
        self.gguf_writer.add_file_type(self.ftype)

        # 如果稀疏阈值不为空
        if params.sparse_threshold is not None:
            # 添加稀疏阈值
            self.gguf_writer.add_sparse_threshold(params.sparse_threshold)

# 定义 FalconModel 类，继承自 Model 类
class FalconModel(Model):
    # 设置 GGUF 参数，接受一个 PredictorParams 对象作为参数
    def set_gguf_parameters(self, params: PredictorParams):
        # 获取隐藏层的数量
        block_count = self.hparams.get("num_hidden_layers")
        # 如果隐藏层数量为空，则使用旧的名称获取
        if block_count is None:
            block_count = self.hparams["n_layer"]  # old name

        # 获取注意力头的数量
        n_head = self.hparams.get("num_attention_heads")
        # 如果注意力头数量为空，则使用旧的名称获取
        if n_head is None:
            n_head = self.hparams["n_head"]  # old name

        # 获取键值头的数量
        n_head_kv = self.hparams.get("num_kv_heads")
        # 如果键值头数量为空，则使用旧的名称获取，如果仍为空，则默认为1
        if n_head_kv is None:
            n_head_kv = self.hparams.get("n_head_kv", 1)  # old name

        # 添加名称为 "Falcon" 的信息
        self.gguf_writer.add_name("Falcon")
        # 添加上下文长度为 2048 的信息，该信息不在 config.json 中
        self.gguf_writer.add_context_length(2048)
        # 添加张量数据布局为 "jploski" 的信息，用于 qkv 张量转换
        self.gguf_writer.add_tensor_data_layout("jploski")
        # 添加嵌入长度为隐藏层大小的信息
        self.gguf_writer.add_embedding_length(self.hparams["hidden_size"])
        # 添加前馈长度为隐藏层大小的四倍的信息
        self.gguf_writer.add_feed_forward_length(4 * self.hparams["hidden_size"])
        # 添加块数量的信息
        self.gguf_writer.add_block_count(block_count)
        # 添加注意力头数量的信息
        self.gguf_writer.add_head_count(n_head)
        # 添加键值头数量的信息
        self.gguf_writer.add_head_count_kv(n_head_kv)
        # 添加层归一化的 epsilon 值的信息
        self.gguf_writer.add_layer_norm_eps(self.hparams["layer_norm_epsilon"])
        # 添加文件类型的信息
        self.gguf_writer.add_file_type(self.ftype)

        # 如果参数中稀疏阈值不为空，则添加稀疏阈值的信息
        if params.sparse_threshold is not None:
            self.gguf_writer.add_sparse_threshold(params.sparse_threshold)
# 定义一个数据类，用于存储预测器参数
@dataclass
class PredictorParams:
    sparse_threshold: float | None = None

    # 从配置文件中加载预测器参数
    @staticmethod
    def loadPredictorJson(config_path: Path) -> PredictorParams:
        # 读取配置文件内容
        config = json.load(open(config_path))
        # 返回预测器参数对象
        return PredictorParams(
            sparse_threshold = config.get("sparse_threshold"),
        )

    # 从模型实例中加载预测器参数
    @staticmethod
    def load(model_instance: Model) -> PredictorParams:
        # 获取配置文件路径
        config_path   = model_instance.dir_mlp_pred  / "config.json"

        # 如果配置文件存在，则加载预测器参数
        if config_path.exists():
            params = PredictorParams.loadPredictorJson(config_path)
        # 否则创建一个空的预测器参数对象
        else:
            params = PredictorParams()

        # 返回预测器参数对象
        return params

###### CONVERSION LOGIC ######


# 解析命令行参数
def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(
        description="Convert a huggingface model to a GGML compatible file"
    )
    parser.add_argument(
        "--vocab-only",
        action="store_true",
        help="extract only the vocab",
    )
    parser.add_argument(
        "--outfile",
        type=Path,
        help="path to write to; default: based on input",
    )
    parser.add_argument(
        "--outtype",
        type=str,
        choices=["f32", "f16"],
        default="f16",
        help="output format - use f32 for float32, f16 for float16",
    )
    parser.add_argument(
        "--bigendian",
        action="store_true",
        help="model is executed on big endian machine",
    )
    parser.add_argument(
        "model",
        type=Path,
        help="directory containing model file",
    )
    parser.add_argument(
        "mlp_predictors",
        type=Path,
        help="directory containing MLP predictors for model",
    )

    # 返回解析后的命令行参数
    return parser.parse_args()


# 解析命令行参数
args = parse_args()

# 获取模型目录和 MLP 预测器目录
dir_model = args.model
dir_mlp_pred = args.mlp_predictors

# 如果模型目录不存在，则输出错误信息并退出
if not dir_model.is_dir():
    print(f"Error: {args.model} is not a directory", file=sys.stderr)
    sys.exit(1)

# 如果 MLP 预测器目录不存在，则输出错误信息并退出
if not dir_mlp_pred.is_dir():
    print(f"Error: {args.mlp_predictors} is not a directory", file=sys.stderr)
    # 退出程序并返回状态码 1
    sys.exit(1)
# 定义文件类型映射，将字符串映射到对应的 GGMLQuantizationType 枚举值
ftype_map = {
    "f32": gguf.GGMLQuantizationType.F32,
    "f16": gguf.GGMLQuantizationType.F16,
}

# 如果命令行参数中指定了输出文件名，则使用指定的文件名，否则默认在模型所在目录下生成以输出类型命名的文件
if args.outfile is not None:
    fname_out = args.outfile
else:
    fname_out = dir_model / f"ggml-model-{args.outtype}.gguf"

# 打印加载模型的信息，输出模型所在目录的名称
print(f"Loading model: {dir_model.name}")

# 加载模型的超参数
hparams = Model.load_hparams(dir_model)

# 根据超参数中的第一个模型架构创建模型类
model_class = Model.from_model_architecture(hparams["architectures"][0])

# 创建模型实例，传入模型所在目录、MLP预测目录、输出类型、输出文件名和字节序标志
model_instance = model_class(
    dir_model, dir_mlp_pred, ftype_map[args.outtype], fname_out, args.bigendian
)

# 设置模型参数
print("Set model parameters")
params = PredictorParams.load(model_instance)
model_instance.set_gguf_parameters(params)

# 设置模型分词器
print("Set model tokenizer")
model_instance.set_vocab()

# 如果命令行参数中指定了只导出词汇表，则将模型词汇表导出到指定文件
if args.vocab_only:
    print(f"Exporting model vocab to '{fname_out}'")
    model_instance.write_vocab()
else:
    # 否则将整个模型导出到指定文件
    print(f"Exporting model to '{fname_out}'")
    model_instance.write()

# 后处理：在导出的文件头部写入另一个唯一的文件标识，以区分原始的 GGUF 文件
with open(fname_out, "r+b") as fout:
    POWERINFER_MAGIC = int.from_bytes(b"PWRI", "little")
    fout.write(struct.pack("<I", POWERINFER_MAGIC))

# 打印模型成功导出的信息，输出导出的文件名
print(f"Model successfully exported to '{fname_out}'")
```