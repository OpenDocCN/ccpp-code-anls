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
    # 初始化特殊词汇类的属性
    merges: list[str]  # 用于存储合并的词汇列表
    add_special_token: dict[str, bool]  # 用于存储是否添加特殊标记的字典
    special_token_ids: dict[str, int]  # 用于存储特殊标记的ID的字典

    # 初始化特殊词汇类的方法
    def __init__(
        self, path: str | os.PathLike[str], load_merges: bool = False,
        special_token_types: tuple[str, ...] | None = None,
        n_vocab: int | None = None,
    # 初始化特殊标记的 ID 字典
    self.special_token_ids = {}
    # 初始化要添加的特殊标记字典
    self.add_special_token = {}
    # 初始化词汇表大小
    self.n_vocab = n_vocab
    # 加载合并文件
    self.load_merges = load_merges
    # 初始化合并列表
    self.merges = []
    # 如果特殊标记类型不为空，则使用传入的特殊标记类型，否则使用默认的特殊标记类型
    if special_token_types is not None:
        self.special_token_types = special_token_types
    else:
        self.special_token_types = ('bos', 'eos', 'unk', 'sep', 'pad')
    # 调用 _load 方法加载路径下的文件
    self._load(Path(path))

# 定义特殊词汇表的字符串表示形式
def __repr__(self) -> str:
    return '<SpecialVocab with {} merges, special tokens {}, add special tokens {}>'.format(
        len(self.merges), self.special_token_ids or "unset", self.add_special_token or "unset",
    )

# 将特殊词汇表添加到 GGUFWriter 对象中
def add_to_gguf(self, gw: GGUFWriter, quiet: bool = False) -> None:
    # 如果存在合并列表
    if self.merges:
        # 如果不是安静模式
        if not quiet:
# 打印消息，指示正在添加合并操作的数量
print(f'gguf: Adding {len(self.merges)} merge(s).')

# 将合并操作添加到指定的地方
gw.add_token_merges(self.merges)

# 如果加载了合并操作但是没有找到合并操作，则打印警告信息
elif self.load_merges:
    print(
        'gguf: WARNING: Adding merges requested but no merges found, output may be non-functional.',
        file = sys.stderr,
    )

# 遍历特殊标记的类型和对应的标记ID
for typ, tokid in self.special_token_ids.items():
    # 获取处理特殊标记ID的函数，如果不存在则打印警告信息并跳过
    id_handler: Callable[[int], None] | None = getattr(gw, f'add_{typ}_token_id', None)
    if id_handler is None:
        print(
            f'gguf: WARNING: No handler for special token type {typ} with id {tokid} - skipping',
            file = sys.stderr,
        )
        continue
    # 如果不是安静模式，则打印设置特殊标记类型和对应ID的消息
    if not quiet:
        print(f'gguf: Setting special token type {typ} to {tokid}')
    id_handler(tokid)

# 遍历要添加的特殊标记类型和对应的值
for typ, value in self.add_special_token.items():
    # 获取添加特殊标记的函数，如果不存在则打印警告信息并跳过
    add_handler: Callable[[bool], None] | None = getattr(gw, f'add_add_{typ}_token', None)
# 如果add_handler为None，则打印警告信息并跳过
if add_handler is None:
    print(
        f'gguf: WARNING: No handler for add_{typ}_token with value {value} - skipping',
        file = sys.stderr,
    )
    continue
# 如果不是安静模式，则打印设置add_{typ}_token的信息
if not quiet:
    print(f'gguf: Setting add_{typ}_token to {value}')
# 调用add_handler处理器处理value

# 从指定路径加载数据，尝试从tokenizer_json文件中加载数据
def _load(self, path: Path) -> None:
    self._try_load_from_tokenizer_json(path)
    # 尝试从config_json文件中加载数据
    self._try_load_from_config_json(path)
    # 如果需要加载合并数据且合并数据为空，则尝试从merges_txt文件中加载数据
    if self.load_merges and not self.merges:
        self._try_load_merges_txt(path)

# 尝试从merges_txt文件中加载数据
def _try_load_merges_txt(self, path: Path) -> bool:
    merges_file = path / 'merges.txt'
    # 如果merges_txt文件不存在，则返回False
    if not merges_file.is_file():
        return False
# 使用只读模式打开文件
with open(merges_file, 'r') as fp:
    # 读取文件的第一行并去除空格
    first_line = next(fp, '').strip()
    # 如果第一行不以#开头，则将文件指针移动到文件开头，行号设为0
    if not first_line.startswith('#'):
        fp.seek(0)
        line_num = 0
    # 否则行号设为1
    else:
        line_num = 1
    # 初始化merges列表
    merges = []
    # 遍历文件的每一行
    for line in fp:
        line_num += 1
        # 去除行两端的空格
        line = line.strip()
        # 如果行为空则跳过
        if not line:
            continue
        # 将行按空格分割成最多3部分
        parts = line.split(None, 3)
        # 如果分割后的部分不等于2，则打印警告信息并继续下一行
        if len(parts) != 2:
            print(
                f'gguf: WARNING: {merges_file.name}: Line {line_num}: Entry malformed, ignoring',
                file = sys.stderr,
            )
            continue
        # 将 parts 列表中的第一个和第二个元素合并成一个字符串，并添加到 merges 列表中
        merges.append(f'{parts[0]} {parts[1]}')
        # 将 merges 列表赋值给 self.merges
        self.merges = merges
        # 返回 True
        return True

    # 设置特殊标记
    def _set_special_token(self, typ: str, tid: Any) -> None:
        # 如果 tid 不是整数或者小于 0，则返回
        if not isinstance(tid, int) or tid < 0:
            return
        # 如果 self.n_vocab 为 None 或者 tid 小于 self.n_vocab
        if self.n_vocab is None or tid < self.n_vocab:
            # 如果 typ 在 self.special_token_ids 中，则返回
            if typ in self.special_token_ids:
                return
            # 将 typ 和 tid 添加到 self.special_token_ids 中
            self.special_token_ids[typ] = tid
            return
        # 打印警告信息
        print(
            f'gguf: WARNING: Special token type {typ}, id {tid} out of range, must be under {self.n_vocab} - skipping',
            file = sys.stderr,
        )

    # 尝试从 tokenizer.json 文件中加载数据
    def _try_load_from_tokenizer_json(self, path: Path) -> bool:
        # 构建 tokenizer.json 文件路径
        tokenizer_file = path / 'tokenizer.json'
        # 如果 tokenizer_file 文件不存在
        if not tokenizer_file.is_file():
# 如果条件不满足，返回 False
            return False
        # 打开 tokenizer_file 文件，使用 utf-8 编码
        with open(tokenizer_file, encoding = 'utf-8') as f:
            # 从文件中加载 JSON 数据，赋值给 tokenizer
            tokenizer = json.load(f)
        # 如果 self.load_merges 为真
        if self.load_merges:
            # 获取 tokenizer 中 model 字段下的 merges 值
            merges = tokenizer.get('model', {}).get('merges')
            # 如果 merges 是列表且不为空，并且 merges[0] 是字符串
            if isinstance(merges, list) and merges and isinstance(merges[0], str):
                # 将 merges 赋值给 self.merges
                self.merges = merges
        # 设置 tokenizer_config_file 为 path 下的 tokenizer_config.json 文件
        tokenizer_config_file = path / 'tokenizer_config.json'
        # 获取 tokenizer 中的 added_tokens 值
        added_tokens = tokenizer.get('added_tokens')
        # 如果 added_tokens 为 None 或者 tokenizer_config_file 文件不存在
        if added_tokens is None or not tokenizer_config_file.is_file():
            # 返回 True
            return True
        # 打开 tokenizer_config_file 文件，使用 utf-8 编码
        with open(tokenizer_config_file, encoding = 'utf-8') as f:
            # 从文件中加载 JSON 数据，赋值给 tokenizer_config
            tokenizer_config = json.load(f)
        # 遍历 self.special_token_types
        for typ in self.special_token_types:
            # 获取 tokenizer_config 中的 add_{typ}_token 值
            add_entry = tokenizer_config.get(f'add_{typ}_token')
            # 如果 add_entry 是布尔类型
            if isinstance(add_entry, bool):
                # 将 add_entry 赋值给 self.add_special_token[typ]
                self.add_special_token[typ] = add_entry
            # 获取 tokenizer_config 中的 {typ}_token 值
            entry = tokenizer_config.get(f'{typ}_token')
            # 如果 entry 是字符串类型
            if isinstance(entry, str):
                # 将 entry 赋值给 tc_content
                tc_content = entry
            # 如果 entry 是一个字典，则获取其 content 值
            elif isinstance(entry, dict):
                entry_content = entry.get('content')
                # 如果 entry_content 不是字符串类型，则继续下一次循环
                if not isinstance(entry_content, str):
                    continue
                # 将 entry_content 赋值给 tc_content
                tc_content = entry_content
            else:
                # 如果 entry 不是字典类型，则继续下一次循环
                continue
            # 在 added_tokens 中查找第一个匹配 tc_content 的 token，并获取其 id
            maybe_token_id = next(
                (atok.get('id') for atok in added_tokens if atok.get('content') == tc_content),
                None,
            )
            # 调用 _set_special_token 方法，设置特殊的 token 类型和 id
            self._set_special_token(typ, maybe_token_id)
        # 返回 True
        return True

    # 从指定路径的 config.json 文件中加载配置信息
    def _try_load_from_config_json(self, path: Path) -> bool:
        # 拼接出配置文件的路径
        config_file = path / 'config.json'
        # 如果配置文件不存在，则返回 False
        if not config_file.is_file():
            return False
        # 打开配置文件，使用 utf-8 编码
        with open(config_file, encoding = 'utf-8') as f:
# 从文件对象中加载 JSON 数据，存储到变量config中
config = json.load(f)
# 遍历特殊标记类型列表，设置特殊标记的ID值
for typ in self.special_token_types:
    self._set_special_token(typ, config.get(f'{typ}_token_id'))
# 返回True，表示函数执行成功
return True
```