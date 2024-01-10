# `PowerInfer\gguf-py\gguf\vocab.py`

```
# 导入必要的模块和类
from __future__ import annotations
import json
import os
import sys
from pathlib import Path
from typing import Any, Callable
from .gguf_writer import GGUFWriter

# 定义一个特殊词汇类
class SpecialVocab:
    # 定义类属性
    merges: list[str]
    add_special_token: dict[str, bool]
    special_token_ids: dict[str, int]

    # 初始化方法
    def __init__(
        self, path: str | os.PathLike[str], load_merges: bool = False,
        special_token_types: tuple[str, ...] | None = None,
        n_vocab: int | None = None,
    ):
        # 初始化实例属性
        self.special_token_ids = {}
        self.add_special_token = {}
        self.n_vocab = n_vocab
        self.load_merges = load_merges
        self.merges = []
        # 如果传入了特殊词汇类型，则使用传入的类型，否则使用默认类型
        if special_token_types is not None:
            self.special_token_types = special_token_types
        else:
            self.special_token_types = ('bos', 'eos', 'unk', 'sep', 'pad')
        # 调用私有方法_load()加载数据
        self._load(Path(path))

    # 定义对象的字符串表示形式
    def __repr__(self) -> str:
        return '<SpecialVocab with {} merges, special tokens {}, add special tokens {}>'.format(
            len(self.merges), self.special_token_ids or "unset", self.add_special_token or "unset",
        )
    # 将数据添加到 GGUFWriter 对象中
    def add_to_gguf(self, gw: GGUFWriter, quiet: bool = False) -> None:
        # 如果存在合并数据
        if self.merges:
            # 如果不是安静模式，则打印合并数据的数量
            if not quiet:
                print(f'gguf: Adding {len(self.merges)} merge(s).')
            # 将合并数据添加到 GGUFWriter 对象中
            gw.add_token_merges(self.merges)
        # 如果存在加载的合并数据
        elif self.load_merges:
            # 打印警告信息，表示请求添加合并数据但未找到合并数据，输出可能无法正常工作
            print(
                'gguf: WARNING: Adding merges requested but no merges found, output may be non-functional.',
                file = sys.stderr,
            )
        # 遍历特殊标记 ID 字典
        for typ, tokid in self.special_token_ids.items():
            # 获取对应类型的处理函数
            id_handler: Callable[[int], None] | None = getattr(gw, f'add_{typ}_token_id', None)
            # 如果处理函数不存在，则打印警告信息并跳过
            if id_handler is None:
                print(
                    f'gguf: WARNING: No handler for special token type {typ} with id {tokid} - skipping',
                    file = sys.stderr,
                )
                continue
            # 如果不是安静模式，则打印设置特殊标记类型的信息
            if not quiet:
                print(f'gguf: Setting special token type {typ} to {tokid}')
            # 调用处理函数设置特殊标记类型
            id_handler(tokid)
        # 遍历添加特殊标记的字典
        for typ, value in self.add_special_token.items():
            # 获取对应类型的处理函数
            add_handler: Callable[[bool], None] | None = getattr(gw, f'add_add_{typ}_token', None)
            # 如果处理函数不存在，则打印警告信息并跳过
            if add_handler is None:
                print(
                    f'gguf: WARNING: No handler for add_{typ}_token with value {value} - skipping',
                    file = sys.stderr,
                )
                continue
            # 如果不是安静模式，则打印设置添加特殊标记的信息
            if not quiet:
                print(f'gguf: Setting add_{typ}_token to {value}')
            # 调用处理函数设置添加特殊标记
            add_handler(value)

    # 从指定路径加载数据
    def _load(self, path: Path) -> None:
        # 尝试从分词器 JSON 文件中加载数据
        self._try_load_from_tokenizer_json(path)
        # 尝试从配置 JSON 文件中加载数据
        self._try_load_from_config_json(path)
        # 如果存在加载的合并数据但未找到合并数据，则尝试从指定路径加载合并数据
        if self.load_merges and not self.merges:
            self._try_load_merges_txt(path)
    # 尝试加载指定路径下的 merges.txt 文件，返回是否成功加载的布尔值
    def _try_load_merges_txt(self, path: Path) -> bool:
        # 拼接路径和文件名，获取 merges.txt 文件的路径
        merges_file = path / 'merges.txt'
        # 如果 merges.txt 文件不存在，则返回 False
        if not merges_file.is_file():
            return False
        # 以只读方式打开 merges.txt 文件
        with open(merges_file, 'r') as fp:
            # 读取文件的第一行并去除首尾空格
            first_line = next(fp, '').strip()
            # 如果第一行不以 '#' 开头，则将文件指针移回文件开头，行号设为 0
            if not first_line.startswith('#'):
                fp.seek(0)
                line_num = 0
            else:
                # 否则行号设为 1
                line_num = 1
            # 初始化 merges 列表
            merges = []
            # 遍历文件的每一行
            for line in fp:
                line_num += 1
                # 去除行首尾空格
                line = line.strip()
                # 如果行为空，则跳过
                if not line:
                    continue
                # 以空格分割行，最多分割成 3 部分
                parts = line.split(None, 3)
                # 如果分割后的部分不是 2 个，则打印警告信息并跳过
                if len(parts) != 2:
                    print(
                        f'gguf: WARNING: {merges_file.name}: Line {line_num}: Entry malformed, ignoring',
                        file = sys.stderr,
                    )
                    continue
                # 将分割后的部分拼接成字符串并添加到 merges 列表中
                merges.append(f'{parts[0]} {parts[1]}')
        # 将 merges 列表赋值给 self.merges
        self.merges = merges
        # 返回 True
        return True

    # 设置特殊标记的方法，参数为标记类型和标记 id，无返回值
    def _set_special_token(self, typ: str, tid: Any) -> None:
        # 如果标记 id 不是整数或小于 0，则返回
        if not isinstance(tid, int) or tid < 0:
            return
        # 如果 self.n_vocab 为 None 或者标记 id 小于 self.n_vocab
        if self.n_vocab is None or tid < self.n_vocab:
            # 如果标记类型已经在 special_token_ids 中，则返回
            if typ in self.special_token_ids:
                return
            # 将标记类型和标记 id 添加到 special_token_ids 中
            self.special_token_ids[typ] = tid
            return
        # 打印警告信息，标记类型和 id 超出范围，跳过
        print(
            f'gguf: WARNING: Special token type {typ}, id {tid} out of range, must be under {self.n_vocab} - skipping',
            file = sys.stderr,
        )
    # 尝试从 tokenizer.json 文件加载数据，返回布尔值表示是否成功加载
    def _try_load_from_tokenizer_json(self, path: Path) -> bool:
        # 拼接得到 tokenizer.json 文件路径
        tokenizer_file = path / 'tokenizer.json'
        # 如果文件不存在，则返回 False
        if not tokenizer_file.is_file():
            return False
        # 打开 tokenizer.json 文件，使用 utf-8 编码
        with open(tokenizer_file, encoding = 'utf-8') as f:
            # 加载 JSON 数据到 tokenizer 变量
            tokenizer = json.load(f)
        # 如果设置了加载 merges，则获取 merges 数据
        if self.load_merges:
            merges = tokenizer.get('model', {}).get('merges')
            # 如果 merges 是列表且不为空，并且第一个元素是字符串，则设置 self.merges
            if isinstance(merges, list) and merges and isinstance(merges[0], str):
                self.merges = merges
        # 拼接得到 tokenizer_config.json 文件路径
        tokenizer_config_file = path / 'tokenizer_config.json'
        # 获取 added_tokens 数据
        added_tokens = tokenizer.get('added_tokens')
        # 如果 added_tokens 为 None 或者 tokenizer_config.json 文件不存在，则返回 True
        if added_tokens is None or not tokenizer_config_file.is_file():
            return True
        # 打开 tokenizer_config.json 文件，使用 utf-8 编码
        with open(tokenizer_config_file, encoding = 'utf-8') as f:
            # 加载 JSON 数据到 tokenizer_config 变量
            tokenizer_config = json.load(f)
        # 遍历特殊标记类型列表
        for typ in self.special_token_types:
            # 获取 add_{typ}_token 数据
            add_entry = tokenizer_config.get(f'add_{typ}_token')
            # 如果 add_entry 是布尔值，则设置 self.add_special_token[typ]
            if isinstance(add_entry, bool):
                self.add_special_token[typ] = add_entry
            # 获取 {typ}_token 数据
            entry = tokenizer_config.get(f'{typ}_token')
            # 如果 entry 是字符串，则设置 tc_content
            if isinstance(entry, str):
                tc_content = entry
            # 如果 entry 是字典，则获取 content 数据作为 tc_content
            elif isinstance(entry, dict):
                entry_content = entry.get('content')
                # 如果 entry_content 不是字符串，则继续下一次循环
                if not isinstance(entry_content, str):
                    continue
                tc_content = entry_content
            else:
                continue
            # 只需要第一个匹配项
            # 在 added_tokens 中查找 content 与 tc_content 相同的元素，并获取其 id
            maybe_token_id = next(
                (atok.get('id') for atok in added_tokens if atok.get('content') == tc_content),
                None,
            )
            # 设置特殊标记类型和对应的 token_id
            self._set_special_token(typ, maybe_token_id)
        # 返回 True
        return True
    # 从指定路径下的config.json文件中尝试加载配置信息，返回布尔值表示是否加载成功
    def _try_load_from_config_json(self, path: Path) -> bool:
        # 拼接路径和文件名，获取config.json文件的路径
        config_file = path / 'config.json'
        # 如果config.json文件不存在，则返回False
        if not config_file.is_file():
            return False
        # 以utf-8编码打开config.json文件
        with open(config_file, encoding = 'utf-8') as f:
            # 从文件中加载JSON数据并存储在config变量中
            config = json.load(f)
        # 遍历self.special_token_types列表中的特殊标记类型
        for typ in self.special_token_types:
            # 从config中获取特殊标记类型对应的标记ID，并调用_set_special_token方法设置特殊标记
            self._set_special_token(typ, config.get(f'{typ}_token_id'))
        # 加载成功，返回True
        return True
```