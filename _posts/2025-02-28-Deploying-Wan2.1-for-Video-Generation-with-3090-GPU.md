---
layout: post
title: Deploying Wan2.1 for Video Generation with 3090 GPU
catalog: true
tag: [Kubernetes, GPU, AI]
---

<!-- TOC depthFrom:2 orderedList:true -->

- [Deploying Wan2.1 for Video Generation with 3090 GPU](#deploying-wan21-for-video-generation-with-3090-gpu)
	- [1. Environment Requirements](#1-environment-requirements)
	- [2. Model Download](#2-model-download)
	- [3. Clone Repository](#3-clone-repository)
	- [4. Install Dependencies](#4-install-dependencies)
	- [5. Generate Videos](#5-generate-videos)
		- [5.1. Using the Generate Script](#51-using-the-generate-script)
		- [5.2. Using Gradio UI Interface](#52-using-gradio-ui-interface)
			- [5.2.1. Launch Gradio Service](#521-launch-gradio-service)
			- [5.2.2. Access UI, Input Prompt, and Generate Video](#522-access-ui-input-prompt-and-generate-video)

<!-- /TOC -->

# Deploying Wan2.1 for Video Generation with 3090 GPU

## 1. Environment Requirements

|Name|Specification|Notes|
|---|---|---|
|Memory VRAM|22GB|22GB is not enough, needs more memory. If not available, swap memory can be used as an alternative|
|GPU|3090|NVIDIA|
|VRAM|24GB||
|CUDA|12.5|CUDA >= 11.7 required, otherwise it will fail|

## 2. Model Download

Use [hfs.sh script to download the model](https://gist.github.com/padeoe/697678ab8e528b85a2a7bddafea1fa4f)

```bash
./hfd.sh Wan-AI/Wan2.1-T2V-1.3B --tool aria2c -x 10 --hf_token xxxxxxx --hf_username xxxxxxx
```

## 3. Clone Repository

```bash
git clone git@github.com:Wan-Video/Wan2.1.git
```

## 4. Install Dependencies

> Note: CUDA >= 11.7 torch >= 2.4.0

```bash
cd Wan2.1
pip install -r requirements.txt
```

## 5. Generate Videos

### 5.1. Using the Generate Script

> Note: It's essential to configure --offload_model True --t5_cpu otherwise GPU memory will OOM when saving the video

```bash
python generate.py  --task t2v-1.3B --size 832*480 --ckpt_dir ./Wan2.1-T2V-1.3B --offload_model True --t5_cpu --sample_shift 8 --sample_guide_scale 6 --prompt "Two anthropomorphic cats in comfy boxing gear and bright gloves fight intensely on a spotlighted stage."
[2025-02-28 08:33:08,062] INFO: Generation job args: Namespace(task='t2v-1.3B', size='832*480', frame_num=81, ckpt_dir='./Wan2.1-T2V-1.3B', offload_model=True, ulysses_size=1, ring_size=1, t5_fsdp=False, t5_cpu=True, dit_fsdp=False, save_file=None, prompt='Two anthropomorphic cats in comfy boxing gear and bright gloves fight intensely on a spotlighted stage.', use_prompt_extend=False, prompt_extend_method='local_qwen', prompt_extend_model=None, prompt_extend_target_lang='ch', base_seed=6930324173022001627, image=None, sample_solver='unipc', sample_steps=50, sample_shift=8.0, sample_guide_scale=6.0)
[2025-02-28 08:33:08,063] INFO: Generation model config: {'__name__': 'Config: Wan T2V 1.3B', 't5_model': 'umt5_xxl', 't5_dtype': torch.bfloat16, 'text_len': 512, 'param_dtype': torch.bfloat16, 'num_train_timesteps': 1000, 'sample_fps': 16, 'sample_neg_prompt': '色调艳丽，过曝，静态，细节模糊不清，字幕，风格，作品，画作，画面，静止，整体发灰，最差质量，低质量，JPEG压缩残留，丑陋的，残缺的，多余的手指，画得不好的手部，画得不好的脸部，畸形的，毁容的，形态畸形的肢体，手指融合，静止不动的画面，杂乱的背景，三条腿，背景人很多，倒着走', 't5_checkpoint': 'models_t5_umt5-xxl-enc-bf16.pth', 't5_tokenizer': 'google/umt5-xxl', 'vae_checkpoint': 'Wan2.1_VAE.pth', 'vae_stride': (4, 8, 8), 'patch_size': (1, 2, 2), 'dim': 1536, 'ffn_dim': 8960, 'freq_dim': 256, 'num_heads': 12, 'num_layers': 30, 'window_size': (-1, -1), 'qk_norm': True, 'cross_attn_norm': True, 'eps': 1e-06}
[2025-02-28 08:33:08,063] INFO: Input prompt: Two anthropomorphic cats in comfy boxing gear and bright gloves fight intensely on a spotlighted stage.
[2025-02-28 08:33:08,065] INFO: Creating WanT2V pipeline.
[2025-02-28 08:35:19,682] INFO: loading ./Wan2.1-T2V-1.3B/models_t5_umt5-xxl-enc-bf16.pth
[2025-02-28 08:37:19,359] INFO: loading ./Wan2.1-T2V-1.3B/Wan2.1_VAE.pth
[2025-02-28 08:37:24,292] INFO: Creating WanModel from ./Wan2.1-T2V-1.3B
[2025-02-28 08:38:17,205] INFO: Generating video ...
100%|███████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 50/50 [08:15<00:00,  9.91s/it]
[2025-02-28 08:47:29,986] INFO: Saving generated video to t2v-1.3B_832*480_1_1_Two_anthropomorphic_cats_in_comfy_boxing_gear_and__20250228_084729.mp4
[2025-02-28 08:47:32,176] INFO: Finished.
```

The video generation is complete. You can find the generated video in the current directory:

```bash
t2v-1.3B_832*480_1_1_Two_anthropomorphic_cats_in_comfy_boxing_gear_and__20250228_084729.mp4
```

<video width="832" height="480" controls>
  <source src="/img/posts/使用3090显卡部署Wan2.1生成视频/两只猫猫打架.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

### 5.2. Using Gradio UI Interface

#### 5.2.1. Launch Gradio Service

> This requires a prompt extension API. You can use either the dashscope API or your own API. Here we'll use the dashscope API https://dashscope.console.aliyun.com/overview

```bash
DASH_API_KEY=xxxx  python gradio/t2v_1.3B_singleGPU.py --prompt_extend_method 'dashscope' --ckpt_dir ./Wan2.1-T2V-1.3B/
Step1: Init prompt_expander...done
Step2: Init 1.3B t2v model...done
* Running on local URL:  http://0.0.0.0:7860

To create a public link, set `share=True` in `launch()`.
100%|███████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| 50/50 [08:14<00:00,  9.88s/it]
```

#### 5.2.2. Access UI, Input Prompt, and Generate Video

![gradio](/img/posts/使用3090显卡部署Wan2.1生成视频/gradio.jpg)

Example prompt:
```bash
Over the snow-covered Meili Snow Mountains, multiple fighter jets fly in formation, performing various aerial maneuvers and stunts against a backdrop of clear blue skies and majestic peaks. The scene is dynamic and awe-inspiring.
```

![t2v](/img/posts/使用3090显卡部署Wan2.1生成视频/result.jpg)

<video width="480" height="832" controls>
  <source src="/img/posts/使用3090显卡部署Wan2.1生成视频/战斗机飞越梅里雪山.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>
