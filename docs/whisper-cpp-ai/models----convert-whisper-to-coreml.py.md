# `whisper.cpp\models\convert-whisper-to-coreml.py`

```cpp
# 导入必要的库
import argparse
import torch
import torch.nn.functional as F
import coremltools as ct
from torch import Tensor
from torch import nn
from typing import Dict
from typing import Optional
from ane_transformers.reference.layer_norm import LayerNormANE as LayerNormANEBase
from coremltools.models.neural_network.quantization_utils import quantize_weights
from whisper.model import Whisper, AudioEncoder, TextDecoder, ResidualAttentionBlock, MultiHeadAttention, ModelDimensions
from whisper import load_model

# 用于将编码器和解码器嵌入层的输入维度从线性映射到二维卷积映射
def linear_to_conv2d_map(state_dict, prefix, local_metadata, strict,
                         missing_keys, unexpected_keys, error_msgs):
    """
    Unsqueeze twice to map nn.Linear weights to nn.Conv2d weights
    """
    for k in state_dict:
        # 检查是否是注意力层的权重
        is_attention = all(substr in k for substr in ['attn', '.weight'])
        # 检查是否是多层感知机的权重
        is_mlp = any(k.endswith(s) for s in ['mlp.0.weight', 'mlp.2.weight'])

        # 如果是注意力层或多层感知机的权重，并且形状是二维的，则进行维度转换
        if (is_attention or is_mlp) and len(state_dict[k].shape) == 2:
            state_dict[k] = state_dict[k][:, :, None, None]

# 修正偏置和缩放顺序倒置
def correct_for_bias_scale_order_inversion(state_dict, prefix, local_metadata,
                                           strict, missing_keys,
                                           unexpected_keys, error_msgs):
    state_dict[prefix + 'bias'] = state_dict[prefix + 'bias'] / state_dict[prefix + 'weight']
    return state_dict

# 自定义的 LayerNormANE 类，继承自 LayerNormANEBase 类
class LayerNormANE(LayerNormANEBase):

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        # 注册加载状态字典前钩子，用于修正偏置和缩放顺序倒置
        self._register_load_state_dict_pre_hook(
            correct_for_bias_scale_order_inversion)

# 自定义的 MultiHeadAttentionANE 类，继承自 MultiHeadAttention 类
class MultiHeadAttentionANE(MultiHeadAttention):
    # 初始化函数，接受状态数和头数作为参数
    def __init__(self, n_state: int, n_head: int):
        # 调用父类的初始化函数
        super().__init__(n_state, n_head)
        # 创建查询层，使用卷积操作，输入和输出通道数均为状态数，卷积核大小为1
        self.query =  nn.Conv2d(n_state, n_state, kernel_size=1)
        # 创建键层，使用卷积操作，输入和输出通道数均为状态数，卷积核大小为1，无偏置
        self.key = nn.Conv2d(n_state, n_state, kernel_size=1, bias=False)
        # 创建值层，使用卷积操作，输入和输出通道数均为状态数，卷积核大小为1
        self.value = nn.Conv2d(n_state, n_state, kernel_size=1)
        # 创建输出层，使用卷积操作，输入和输出通道数均为状态数，卷积核大小为1
        self.out = nn.Conv2d(n_state, n_state, kernel_size=1)

    # 前向传播函数，接受输入张量x，可选的辅助输入xa，可选的掩码mask，可选的键值缓存kv_cache作为参数
    def forward(self,
                x: Tensor,
                xa: Optional[Tensor] = None,
                mask: Optional[Tensor] = None,
                kv_cache: Optional[dict] = None):

        # 对输入张量x进行查询操作，得到查询张量q
        q = self.query(x)

        # 如果键值缓存为空或者辅助输入xa为空或者键不在缓存中
        if kv_cache is None or xa is None or self.key not in kv_cache:
            # 如果存在钩子函数（即kv_cache不为空），则在缓存的键值张量前面添加；
            # 否则，对自注意力或交叉注意力进行键值投影
            k = self.key(x if xa is None else xa)
            v = self.value(x if xa is None else xa)

        else:
            # 对于交叉注意力，计算一次键和值，并在后续调用中重复使用
            k = kv_cache[self.key]
            v = kv_cache[self.value]

        # 调用自定义的qkv_attention_ane函数，进行注意力计算，得到加权值张量wv和注意力张量qk
        wv, qk = self.qkv_attention_ane(q, k, v, mask)

        # 对加权值张量进行输出操作，得到最终输出张量，同时返回注意力张量qk
        return self.out(wv), qk
    # 定义一个 QKV 注意力机制的函数，接受输入的查询、键、值张量以及可选的掩码张量
    def qkv_attention_ane(self, q: Tensor, k: Tensor, v: Tensor, mask: Optional[Tensor] = None):

        # 获取输入张量的维度信息
        _, dim, _, seqlen = q.size()

        # 计算每个注意力头的维度
        dim_per_head = dim // self.n_head

        # 计算缩放因子
        scale = float(dim_per_head)**-0.5

        # 对查询张量进行缩放
        q = q * scale

        # 将查询、键、值张量按照每个注意力头的维度进行分割
        mh_q = q.split(dim_per_head, dim=1)
        mh_k = k.transpose(1,3).split(dim_per_head, dim=3)
        mh_v = v.split(dim_per_head, dim=1)

        # 计算查询-键的乘积
        mh_qk = [
            torch.einsum('bchq,bkhc->bkhq', [qi, ki])
            for qi, ki in zip(mh_q, mh_k)
        ]  # (batch_size, max_seq_length, 1, max_seq_length) * n_heads

        # 如果存在掩码张量，则将其应用到查询-键的乘积上
        if mask is not None:
            for head_idx in range(self.n_head):
                mh_qk[head_idx] = mh_qk[head_idx] + mask[:, :seqlen, :, :seqlen]

        # 对查询-键的乘积进行 softmax 操作，得到注意力权重
        attn_weights = [aw.softmax(dim=1) for aw in mh_qk]  # (batch_size, max_seq_length, 1, max_seq_length) * n_heads
        # 根据注意力权重计算最终的注意力值
        attn = [torch.einsum('bkhq,bchk->bchq', wi, vi) for wi, vi in zip(attn_weights, mh_v)]  # (batch_size, dim_per_head, 1, max_seq_length) * n_heads
        # 将每个注意力头的结果拼接在一起
        attn = torch.cat(attn, dim=1)  # (batch_size, dim, 1, max_seq_length)

        # 返回注意力值和查询-键的乘积拼接结果，并转换为浮点型并且断开梯度
        return attn, torch.cat(mh_qk, dim=1).float().detach()
class ResidualAttentionBlockANE(ResidualAttentionBlock):
    # 定义 ResidualAttentionBlockANE 类，继承自 ResidualAttentionBlock 类
    def __init__(self, n_state: int, n_head: int, cross_attention: bool = False):
        # 初始化函数，接受 n_state（状态数量）、n_head（头数量）、cross_attention（是否跨注意力）参数
        super().__init__(n_state, n_head, cross_attention)
        # 调用父类的初始化函数
        self.attn =  MultiHeadAttentionANE(n_state, n_head)
        # 创建 MultiHeadAttentionANE 对象，用于自注意力机制
        self.attn_ln = LayerNormANE(n_state)
        # 创建 LayerNormANE 对象，用于归一化
        self.cross_attn =  MultiHeadAttentionANE(n_state, n_head) if cross_attention else None
        # 根据是否跨注意力创建 MultiHeadAttentionANE 对象
        self.cross_attn_ln =  LayerNormANE(n_state) if cross_attention else None
        # 根据是否跨注意力创建 LayerNormANE 对象

        n_mlp = n_state * 4
        # 计算 MLP 层的输入维度
        self.mlp =  nn.Sequential(
            nn.Conv2d(n_state, n_mlp, kernel_size=1),
            nn.GELU(),
            nn.Conv2d(n_mlp, n_state, kernel_size=1)
        )
        # 创建 MLP 层，包含两个卷积层
        self.mlp_ln = LayerNormANE(n_state)
        # 创建 LayerNormANE 对象，用于 MLP 层的归一化


class AudioEncoderANE(AudioEncoder):
    # 定义 AudioEncoderANE 类，继承自 AudioEncoder 类
    def __init__(self, n_mels: int, n_ctx: int, n_state: int, n_head: int, n_layer: int):
        # 初始化函数，接受 n_mels（梅尔频谱数量）、n_ctx（上下文数量）、n_state（状态数量）、n_head（头数量）、n_layer（层数）参数
        super().__init__(n_mels, n_ctx, n_state, n_head, n_layer)
        # 调用父类的初始化函数

        self.blocks = nn.ModuleList(
            [ResidualAttentionBlockANE(n_state, n_head) for _ in range(n_layer)]
        )
        # 创建包含多个 ResidualAttentionBlockANE 对象的 ModuleList

        self.ln_post = LayerNormANE(n_state)
        # 创建 LayerNormANE 对象，用于最终的归一化

    def forward(self, x: Tensor):
        """
        x : torch.Tensor, shape = (batch_size, n_mels, n_ctx)
            the mel spectrogram of the audio
        """
        # 前向传播函数，接受输入 x，表示音频的梅尔频谱
        x = F.gelu(self.conv1(x))
        # 使用 GELU 激活函数对第一个卷积层的输出进行处理
        x = F.gelu(self.conv2(x))
        # 使用 GELU 激活函数对第二个卷积层的输出进行处理

        assert x.shape[1:] == self.positional_embedding.shape[::-1], "incorrect audio shape"
        # 断言输入的形状与位置编码的形状匹配，用于检查音频形状是否正确

        # 添加位置编码并为 ANE 添加虚拟维度
        x = (x + self.positional_embedding.transpose(0,1)).to(x.dtype).unsqueeze(2)

        for block in self.blocks:
            x = block(x)
        # 遍历每个 ResidualAttentionBlockANE 对象，进行前向传播

        x = self.ln_post(x)
        # 对最终输出进行归一化
        x = x.squeeze(2).transpose(1, 2)
        # 去除虚拟维度并转置维度

        return x
        # 返回输出结果

class TextDecoderANE(TextDecoder):
    # 定义 TextDecoderANE 类，继承自 TextDecoder 类
    # 初始化函数，接受参数 n_vocab: int, n_ctx: int, n_state: int, n_head: int, n_layer: int
    def __init__(self, n_vocab: int, n_ctx: int, n_state: int, n_head: int, n_layer: int):
        # 调用父类的初始化函数，传入参数 n_vocab, n_ctx, n_state, n_head, n_layer
        super().__init__(n_vocab, n_ctx, n_state, n_head, n_layer)

        # 创建一个包含多个 ResidualAttentionBlockANE 对象的 ModuleList，数量为 n_layer
        self.blocks= nn.ModuleList(
            [ResidualAttentionBlockANE(n_state, n_head, cross_attention=True) for _ in range(n_layer)]
        )
        # 创建一个 LayerNormANE 对象，传入参数 n_state
        self.ln= LayerNormANE(n_state)
    # 定义前向传播函数，接受文本标记、编码的音频特征和键值缓存作为输入
    def forward(self, x: Tensor, xa: Tensor, kv_cache: Optional[dict] = None):
        """
        x : torch.LongTensor, shape = (batch_size, <= n_ctx)
            文本标记
        xa : torch.Tensor, shape = (batch_size, n_mels, n_audio_ctx)
            编码的音频特征，用于注意力机制
        """
        # 如果存在键值缓存，则计算偏移量
        offset = next(iter(kv_cache.values())).shape[3] if kv_cache else 0
        # 对文本标记进行标记嵌入和位置嵌入
        x = self.token_embedding(x) + self.positional_embedding[offset : offset + x.shape[-1]]
        # 将 x 转换为与 xa 相同的数据类型
        x = x.to(xa.dtype)

        # 为了适应 ANE，重新格式化数据
        mask = self.mask[None, None, :, :].permute(0,3,1,2)
        x = x.transpose(1,2).unsqueeze(2)

        # 遍历每个块并进行处理
        for block in self.blocks:
            x = block(x, xa, mask=mask, kv_cache=kv_cache)

        # 对结果进行 LayerNorm 处理
        x = self.ln(x)

        # 从 ANE 格式转换回来
        x = x.permute(0,2,3,1).squeeze(0)

        # ANE 只能加载维度大小最多为 16,384 的张量，whisper 使用 51,864 (en) 或 51,865 (multi-lang) 标记，因此需要分块计算
        if self.token_embedding.weight.shape[0] >= 51865:
            # 分成 11 块 - 每块 4715 个
            splits = self.token_embedding.weight.split(self.token_embedding.weight.shape[0]//11, dim=0)
            logits = torch.cat([torch.einsum('bid,jd->bij', x, split) for split in splits]).view(*x.shape[:2], -1)
        else:
            # 分成 12 块 - 每块 4322 个
            assert(self.token_embedding.weight.shape[0] == 51864)
            splits = self.token_embedding.weight.split(self.token_embedding.weight.shape[0]//12, dim=0)
            logits = torch.cat([torch.einsum('bid,jd->bij', x, split) for split in splits]).view(*x.shape[:2], -1)

        # 返回 logits
        return logits
# 定义一个继承自Whisper类的WhisperANE类
class WhisperANE(Whisper):
    # 初始化方法，接受一个ModelDimensions对象作为参数
    def __init__(self, dims: ModelDimensions):
        # 调用父类的初始化方法
        super().__init__(dims)

        # 创建一个AudioEncoderANE对象并赋值给self.encoder
        self.encoder = AudioEncoderANE(
            self.dims.n_mels,
            self.dims.n_audio_ctx,
            self.dims.n_audio_state,
            self.dims.n_audio_head,
            self.dims.n_audio_layer,
        )
        # 创建一个TextDecoderANE对象并赋值给self.decoder
        self.decoder = TextDecoderANE(
            self.dims.n_vocab,
            self.dims.n_text_ctx,
            self.dims.n_text_state,
            self.dims.n_text_head,
            self.dims.n_text_layer,
        )

        # 注册一个预加载状态字典的钩子函数
        self._register_load_state_dict_pre_hook(linear_to_conv2d_map)

    # 前向传播方法，接受mel和tokens两个torch.Tensor对象作为参数，返回一个字典
    def forward(self, mel: torch.Tensor, tokens: torch.Tensor) -> Dict[str, torch.Tensor]:
        # 调用self.encoder对mel进行编码，然后将结果传递给self.decoder进行解码
        return self.decoder(tokens, self.encoder(mel))

    # 安装键值缓存钩子函数，接受一个可选的缓存字典作为参数
    def install_kv_cache_hooks(self, cache: Optional[dict] = None):
        # 如果缓存不为None，则将其复制一份，否则创建一个空字典
        cache = {**cache} if cache is not None else {}
        hooks = []

        # 定义一个保存结果到缓存的函数
        def save_to_cache(module, _, output):
            if module not in cache or output.shape[3] > self.decoder.positional_embedding.shape[0]:
                # 如果模块不在缓存中或输出的维度大于解码器的位置嵌入维度，则直接保存
                cache[module] = output
            else:
                # 否则将输出与缓存拼接后保存
                cache[module] = torch.cat([cache[module], output], dim=3).detach()
            return cache[module]

        # 定义一个安装钩子函数的函数
        def install_hooks(layer: nn.Module):
            if isinstance(layer, MultiHeadAttentionANE):
                # 如果是MultiHeadAttentionANE类型的层，则为key和value注册前向钩子
                hooks.append(layer.key.register_forward_hook(save_to_cache))
                hooks.append(layer.value.register_forward_hook(save_to_cache))

        # 对self.decoder应用安装钩子函数
        self.decoder.apply(install_hooks)
        return cache, hooks

# 将编码器转换为torch脚本，接受hparams、model和quantize三个参数
def convert_encoder(hparams, model, quantize=False):
    # 将模型设置为评估模式
    model.eval()

    # 定义输入数据的形状和数据
    input_shape = (1, hparams.n_mels, 3000)
    input_data = torch.randn(input_shape)
    # 对模型和输入数据进行追踪
    traced_model = torch.jit.trace(model, input_data)
    # 将 PyTorch 模型转换为 TVM 模型
    model = ct.convert(
        traced_model,
        convert_to=None if quantize else "mlprogram", # 如果不进行量化，则转换为 "mlprogram"，否则不转换
        inputs=[ct.TensorType(name="logmel_data", shape=input_shape)], # 模型输入为 logmel_data，形状为 input_shape
        outputs=[ct.TensorType(name="output")], # 模型输出为 output
        compute_units=ct.ComputeUnit.ALL # 使用所有计算单元进行计算
    )

    # 如果进行量化
    if quantize:
        # 对模型的权重进行量化，使用 16 位
        model = quantize_weights(model, nbits=16)

    # 返回转换后的模型
    return model
# 将模型转换为特定的解码器
def convert_decoder(hparams, model, quantize=False):
    # 将模型设置为评估模式
    model.eval()

    # 定义 tokens_shape 和 audio_shape 的形状
    tokens_shape = (1, 1)
    audio_shape = (1, hparams.n_audio_state, 1, 1500)

    # 生成随机音频数据
    audio_data = torch.randn(audio_shape)
    # 生成随机 tokens 数据
    token_data = torch.randint(50257, tokens_shape).long()
    # 对模型进行追踪
    traced_model = torch.jit.trace(model, (token_data, audio_data))

    # 将模型转换为指定格式
    model = ct.convert(
        traced_model,
        convert_to=None if quantize else "mlprogram", # 如果量化权重，则转换将失败，原因不明
        inputs=[
            ct.TensorType(name="token_data", shape=tokens_shape, dtype=int),
            ct.TensorType(name="audio_data", shape=audio_shape)
        ]
    )

    # 如果需要量化，则对权重进行量化
    if quantize:
        model = quantize_weights(model, nbits=16)

    # 返回转换后的模型
    return model


if __name__ == "__main__":
    # 创建参数解析器
    parser = argparse.ArgumentParser()
    # 添加参数选项
    parser.add_argument("--model", type=str, help="model to convert (e.g. tiny, tiny.en, base, base.en, small, small.en, medium, medium.en, large-v1, large-v2, large-v3)", required=True)
    parser.add_argument("--encoder-only", type=bool, help="only convert encoder", default=False)
    parser.add_argument("--quantize",     type=bool, help="quantize weights to F16", default=False)
    parser.add_argument("--optimize-ane", type=bool, help="optimize for ANE execution (currently broken)", default=False)
    # 解析参数
    args = parser.parse_args()

    # 检查模型名称是否有效
    if args.model not in ["tiny", "tiny.en", "base", "base.en", "small", "small.en", "small.en-tdrz", "medium", "medium.en", "large-v1", "large-v2", "large-v3"]:
        raise ValueError("Invalid model name")

    # 加载指定模型并将其放在 CPU 上
    whisper = load_model(args.model).cpu()
    # 获取 whisper 模型的维度参数
    hparams = whisper.dims
    print(hparams)

    # 如果需要优化 ANE 执行
    if args.optimize_ane:
        # 创建 WhisperANE 模型
        whisperANE = WhisperANE(hparams).eval()
        # 加载 whisper 模型的状态字典
        whisperANE.load_state_dict(whisper.state_dict())

        # 获取 encoder 和 decoder
        encoder = whisperANE.encoder
        decoder = whisperANE.decoder
    else:
        # 获取 encoder 和 decoder
        encoder = whisper.encoder
        decoder = whisper.decoder

    # 转换编码器
    # 根据给定的超参数和编码器模型，使用指定的量化方式转换编码器模型
    encoder = convert_encoder(hparams, encoder, quantize=args.quantize)
    # 将转换后的编码器模型保存为指定文件名的 CoreML 包
    encoder.save(f"models/coreml-encoder-{args.model}.mlpackage")

    # 如果不仅转换编码器模型
    if args.encoder_only is False:
        # 转换解码器
        decoder = convert_decoder(hparams, decoder, quantize=args.quantize)
        # 将转换后的解码器模型保存为指定文件名的 CoreML 包
        decoder.save(f"models/coreml-decoder-{args.model}.mlpackage")

    # 打印转换完成的提示信息
    print("done converting")
```