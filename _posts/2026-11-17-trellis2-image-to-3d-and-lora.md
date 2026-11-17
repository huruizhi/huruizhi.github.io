---
layout: post
title: "TRELLIS.2 图片转 3D，从生成第一个 GLB 到微调自己的品类"
date: 2026-11-17 23:51:23 +0800
categories: [AI, models]
tags: ["TRELLIS.2", "Image-to-3D", "O-Voxel", "PBR", "LoRA"]
description: "从单张图片生成带 PBR 材质的 GLB，理解 TRELLIS.2 的生成流程，并用自有网格训练几何 LoRA。"
wechat_source: "BlogWechat/TRELLIS.2-博客正式稿.md"
source_url: "https://github.com/microsoft/TRELLIS.2"
---

手里只有一张镂空灯罩的产品图，想做一个能转到背面看的 3D 预览，可以先试试 TRELLIS.2。微软公开的 4B 模型接收一张图，生成由三角形组成的网格，并为它配上颜色、金属度和粗糙度等材质属性。导出 GLB 后，网格与贴图可以装在同一个文件里，交给 Blender 或支持 glTF 的查看器。 [微软项目页](https://microsoft.github.io/TRELLIS.2/) · [官方权重](https://huggingface.co/microsoft/TRELLIS.2-4B)

## 先生成一个能打开的 3D 模型

在微软的在线演示里选一张示例图，也可以上传主体完整、边缘清楚的单物体图。带透明背景的 PNG 可以把现成的前景遮罩交给模型，普通照片会先经过自动抠图。点 **Generate** 查看预览，接着用 **Extract GLB** 和 **Download GLB** 把文件下载下来。 [微软在线演示](https://huggingface.co/spaces/microsoft/TRELLIS.2) · [图像预处理源码](https://github.com/microsoft/TRELLIS.2/blob/main/trellis2/pipelines/trellis2_image_to_3d.py#L119-L154)

把灯罩转到侧面和背面，看看镂空是否贯通，边缘有没有粘在一起。原图看不到的部分会按模型从训练数据里学到的形状补出来。再给灯罩换一个环境光。基于物理的渲染（PBR）会根据金属度、粗糙度等材质属性计算反光，比如磨砂表面的反光会更分散。透明度也值得单独确认。官方导出的 GLB 把透明信息留在贴图里，材质默认按不透明处理。资产用贴图做镂空或半透明时，需要在 3D 软件里启用相应的透明模式。 [模型卡的局限](https://huggingface.co/microsoft/TRELLIS.2-4B#known-limitations) · [官方导出说明](https://github.com/microsoft/TRELLIS.2#-usage)

需要批量生成时，可以把原版部署到 Linux GPU 机器上。微软文档给出的起步配置是 24 GB 显存，推荐 CUDA 12.4。以下命令以已配置 CUDA Toolkit 和 Conda 的 Ubuntu 为例。先在 Hugging Face 申请访问 DINOv3-L 和 RMBG-2.0 权重，再用同一个账号登录。官方示例会生成 `sample.glb` 和旋转预览视频 `sample.mp4`。 [安装文档](https://github.com/microsoft/TRELLIS.2#-installation) · [示例源码](https://github.com/microsoft/TRELLIS.2/blob/main/example.py) · [模型依赖配置](https://huggingface.co/microsoft/TRELLIS.2-4B/blob/main/pipeline.json)

```bash
git clone --recursive https://github.com/microsoft/TRELLIS.2.git
cd TRELLIS.2
. ./setup.sh --new-env --basic --flash-attn --nvdiffrast --nvdiffrec --cumesh --o-voxel --flexgemm
hf auth login
python example.py
```

## 一张图怎么变成网格和材质

想象把灯罩放进由小方格组成的空间。只记哪些格子碰到了灯罩，就很难说清一片薄金属在格子里到底怎么走。O-Voxel 还会记录表面穿过哪些格子边，以及格子里用于重建表面的顶点位置。解码时把相邻顶点连起来，就能还原穿过这些格子的表面。带开口的壳和薄片都可以用这种方式表示。两层表面挤进同一个格子时，细节仍可能混在一起。颜色和材质属性也存进这些表面格子，所以它们和几何共用同一套三维坐标。 [论文 §3.1 与附录 F](https://arxiv.org/html/2512.14692v1)

生成时先用图像编码器 DINOv3 提取图片特征。第一个模型先圈出灯罩表面大概占哪些格子，第二个模型在这些位置生成更细的几何，第三个模型同时参考图像和几何来预测材质。三个阶段生成的是经过压缩的三维数据，也叫潜变量。解码器再把几何变回三角网格，把材质还原成可以烘焙到贴图上的属性。已有网格时，也可以把它编码后直接交给材质阶段，按参考图生成新的贴图。 [论文 §3.2–3.3](https://arxiv.org/html/2512.14692v1#S3) · [用已有网格生成材质](https://github.com/microsoft/TRELLIS.2/blob/main/example_texturing.py)

`1024³` 指三维网格每个轴有 1024 个格子。导出多少三角面、贴图多大，可以单独调。微软在 H100 上报告 `512³` 的生成约需 3 秒，`1024³` 约需 17 秒，导出时还要处理网格、展开 UV 并烘焙贴图。UV 展开可以理解为把三维表面摊到一张平面图上，好让贴图找到对应位置。做批处理时，可以分别记录模型生成、GLB 导出和从图片到文件的总耗时。 [官方耗时表](https://github.com/microsoft/TRELLIS.2#1-high-quality-resolution--efficiency) · [导出参数](https://github.com/microsoft/TRELLIS.2/blob/main/example.py)

## 有自己的 3D 资产，怎么做微调

假如做的是一整套灯具，基础模型总把某类灯罩的开孔填上，就可以试着用现有灯具网格微调。每件物体准备一个网格文件，GLB 和 OBJ 都可以。有网格就可以用 Blender 渲染对应的图片，给同一个物体换方向和光照。微软为每件资产渲染 16 个视角，每次从中抽一张作为输入，目标是同一件网格编码出来的三维潜变量。手头只有照片时，还需要对应的 3D 网格来得到这些潜变量。 [论文数据准备](https://arxiv.org/html/2512.14692v1#A3) · [训练数据读取源码](https://github.com/microsoft/TRELLIS.2/blob/main/trellis2/datasets/components.py#L100-L131)

LoRA 会固定原模型的参数，只训练一小组附加参数。Hugging Face 用鞋类扫描件做了一个现成的例子。它分别给粗结构和细几何两个生成阶段加 LoRA，材质继续由原模型生成。实验用其中 179 件扫描件训练，另留出 9 件评测。两段 LoRA 在单张 A100 80 GB 上分别训练了 7.2 和 3.6 小时。留出集上，衡量生成表面与真值表面匹配程度的 F-score 从 0.276 升到 0.803。评测允许的表面距离是扫描物包围盒对角线的 2%，换一个品类或阈值需要重新评估。 [Hugging Face 鞋类实验](https://huggingface.co/blog/3d-lora-finetuning#results)

公开代码提供了带依赖的 Docker 镜像。把自己的网格放进容器的 `/workspace/work/my_meshes`，并传入已经获得 DINOv3 权重访问权限的 `HF_TOKEN`。进入作者提供的脚本目录后，先做一次预处理，再分别启动两个训练任务。下面用作者验证过的 24 GB L4 批量设置跑 5 步，足以检查能否读到数据和保存适配器。 [容器与本地脚本](https://github.com/suvadityamuk/trellis2-finetuning#one-script-version) · [显存设置](https://github.com/suvadityamuk/trellis2-finetuning#gpu-memory)

```bash
python preprocess.py --meshes /workspace/work/my_meshes --out /workspace/work/data
python finetune.py --flow ss --latents /workspace/work/data \
  --set batch_size=2 grad_accum=8 max_steps=5 i_save=5
python finetune.py --flow shape --latents /workspace/work/data \
  --set batch_size=2 grad_accum=8 max_steps=5 i_save=5
```

评测时按物体留出样本，同款灯罩的颜色变体放在同一侧，避免训练和评测重复看到同一个形状。用同一张图和同一个随机种子分别生成，对照真值网格检查开孔和背面。若想微调材质，还需要有独立的基础色、金属度和粗糙度等属性。阴影已经烘焙到颜色贴图上时，直接用它做材质监督可能会把光照也当成颜色来学。 [留出集与评测脚本](https://github.com/suvadityamuk/trellis2-finetuning#evaluate) · [材质监督说明](https://huggingface.co/blog/3d-lora-finetuning#which-components-to-fine-tune)

## 从头复现还差什么

微软把 4B 权重和代码以 MIT 许可发布，也公开了 `train.py`、训练配置和数据处理工具。论文中的训练先让形状与材质的编解码器学会重建 3D 资产，再用编码得到的潜变量训练三个生成器。论文报告编解码器用 16 张 H100，每个生成器用 32 张 H100。生成训练的数据规模在正文中约为 80 万件资产，来自 Objaverse-XL、TexVerse 等多个来源。 [官方训练入口](https://github.com/microsoft/TRELLIS.2#-training) · [论文 §4 与附录 C](https://arxiv.org/html/2512.14692v1#S4)

复现时要先拿到并处理资产。官方发布页没有列出整套原始训练集的可下载快照。仓库的 `build_metadata.py` 会尝试加载各个数据源的适配器，而主分支缺少对应目录。即使资产已经到手，也要先补适配器或自己产出训练器需要的元数据和潜变量。微软原版还依赖 DINOv3 和 RMBG-2.0，各有权重访问条件与使用条款。 [缺失模块记录](https://github.com/microsoft/TRELLIS.2/issues/83) · [DINOv3 模型卡](https://huggingface.co/facebook/dinov3-vitl16-pretrain-lvd1689m) · [RMBG-2.0 模型卡](https://huggingface.co/briaai/RMBG-2.0)

要决定是否值得训练，可以先让基础模型处理几张能代表目标品类的照片，记录失败发生在哪些部位。若换个角度就能把开孔交代清楚，先改善输入图。若不同样本总在同一类结构上出错，再拿手头的网格做配对训练，把同样的问题放到留出集上，看微调后的形状是否更接近真值。
