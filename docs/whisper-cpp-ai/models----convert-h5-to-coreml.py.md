# `whisper.cpp\models\convert-h5-to-coreml.py`

```cpp
# 导入必要的库
import argparse  # 导入用于解析命令行参数的模块
import importlib.util  # 导入用于导入模块的模块

# 根据文件路径和模块名创建模块规范
spec = importlib.util.spec_from_file_location('whisper_to_coreml', 'models/convert-whisper-to-coreml.py')
# 根据模块规范加载模块
whisper_to_coreml = importlib.util.module_from_spec(spec)
# 执行加载的模块
spec.loader.exec_module(whisper_to_coreml)

# 从whisper模块中导入load_model函数
from whisper import load_model

# 从copy模块中导入deepcopy函数
from copy import deepcopy
# 导入torch库
import torch
# 从transformers库中导入WhisperForConditionalGeneration类
from transformers import WhisperForConditionalGeneration
# 从huggingface_hub库中导入metadata_update函数

# 定义用于重命名键的映射关系
WHISPER_MAPPING = {
    "layers": "blocks",
    "fc1": "mlp.0",
    "fc2": "mlp.2",
    "final_layer_norm": "mlp_ln",
    "layers": "blocks",
    ".self_attn.q_proj": ".attn.query",
    ".self_attn.k_proj": ".attn.key",
    ".self_attn.v_proj": ".attn.value",
    ".self_attn_layer_norm": ".attn_ln",
    ".self_attn.out_proj": ".attn.out",
    ".encoder_attn.q_proj": ".cross_attn.query",
    ".encoder_attn.k_proj": ".cross_attn.key",
    ".encoder_attn.v_proj": ".cross_attn.value",
    ".encoder_attn_layer_norm": ".cross_attn_ln",
    ".encoder_attn.out_proj": ".cross_attn.out",
    "decoder.layer_norm.": "decoder.ln.",
    "encoder.layer_norm.": "encoder.ln_post.",
    "embed_tokens": "token_embedding",
    "encoder.embed_positions.weight": "encoder.positional_embedding",
    "decoder.embed_positions.weight": "decoder.positional_embedding",
    "layer_norm": "ln_post",
}

# 定义函数用于重命名字典中的键
def rename_keys(s_dict):
    # 获取字典的所有键
    keys = list(s_dict.keys())
    # 遍历所有键
    for key in keys:
        new_key = key
        # 根据映射关系重命名键
        for k, v in WHISPER_MAPPING.items():
            if k in key:
                new_key = new_key.replace(k, v)

        # 打印重命名前后的键
        print(f"{key} -> {new_key}")

        # 更新字典中的键名
        s_dict[new_key] = s_dict.pop(key)
    return s_dict

# https://github.com/bayartsogt-ya/whisper-multiple-hf-datasets/blob/main/src/multiple_datasets/hub_default_utils.py
# 将 HF 模型转换为 Whisper 模型
def convert_hf_whisper(hf_model_name_or_path: str, whisper_state_path: str):
    # 从预训练的 HF 模型名称或路径中加载 Whisper 模型
    transformer_model = WhisperForConditionalGeneration.from_pretrained(hf_model_name_or_path)
    # 获取模型配置信息
    config = transformer_model.config

    # 构建维度信息字典
    dims = {
        'n_mels': config.num_mel_bins,
        'n_vocab': config.vocab_size,
        'n_audio_ctx': config.max_source_positions,
        'n_audio_state': config.d_model,
        'n_audio_head': config.encoder_attention_heads,
        'n_audio_layer': config.encoder_layers,
        'n_text_ctx': config.max_target_positions,
        'n_text_state': config.d_model,
        'n_text_head': config.decoder_attention_heads,
        'n_text_layer': config.decoder_layers
    }

    # 深拷贝模型的状态字典
    state_dict = deepcopy(transformer_model.model.state_dict())
    # 重命名模型的键
    state_dict = rename_keys(state_dict)

    # 保存维度信息和模型状态字典到指定路径
    torch.save({"dims": dims, "model_state_dict": state_dict}, whisper_state_path)

# 从 models/convert-whisper-to-coreml.py 转移而来
if __name__ == "__main__":
    # 创建参数解析器
    parser = argparse.ArgumentParser()
    # 添加命令行参数
    parser.add_argument("--model-name", type=str, help="name of model to convert (e.g. tiny, tiny.en, base, base.en, small, small.en, medium, medium.en, large-v1, large-v2, large-v3)", required=True)
    parser.add_argument("--model-path", type=str, help="path to the model (e.g. if published on HuggingFace: Oblivion208/whisper-tiny-cantonese)", required=True)
    parser.add_argument("--encoder-only", type=bool, help="only convert encoder", default=False)
    parser.add_argument("--quantize",     type=bool, help="quantize weights to F16", default=False)
    parser.add_argument("--optimize-ane", type=bool, help="optimize for ANE execution (currently broken)", default=False)
    # 解析命令行参数
    args = parser.parse_args()

    # 检查模型名称是否有效
    if args.model_name not in ["tiny", "tiny.en", "base", "base.en", "small", "small.en", "medium", "medium.en", "large-v1", "large-v2", "large-v3"]:
        raise ValueError("Invalid model name")

    # 设置 PyTorch 模型保存路径
    pt_target_path = f"models/hf-{args.model_name}.pt"
    # 调用函数将 HF 模型转换为 PyTorch 模型
    convert_hf_whisper(args.model_path, pt_target_path)

    # 加载 PyTorch 模型并将其放在 CPU 上
    whisper = load_model(pt_target_path).cpu()
    # 获取模型的参数信息
    hparams = whisper.dims
    # 打印参数信息
    print(hparams)

    # 如果需要优化为 ANE 模型
    if args.optimize_ane:
        # 创建 WhisperANE 模型并加载参数
        whisperANE = whisper_to_coreml.WhisperANE(hparams).eval()
        whisperANE.load_state_dict(whisper.state_dict())

        # 获取 ANE 模型的编码器和解码器
        encoder = whisperANE.encoder
        decoder = whisperANE.decoder
    else:
        # 获取原始模型的编码器和解码器
        encoder = whisper.encoder
        decoder = whisper.decoder

    # 将编码器转换为 CoreML 模型
    encoder = whisper_to_coreml.convert_encoder(hparams, encoder, quantize=args.quantize)
    # 保存编码器的 CoreML 模型
    encoder.save(f"models/coreml-encoder-{args.model_name}.mlpackage")

    # 如果不仅需要编码器
    if args.encoder_only is False:
        # 将解码器转换为 CoreML 模型
        decoder = whisper_to_coreml.convert_decoder(hparams, decoder, quantize=args.quantize)
        # 保存解码器的 CoreML 模型
        decoder.save(f"models/coreml-decoder-{args.model_name}.mlpackage")

    # 打印转换完成的信息
    print("done converting")
```