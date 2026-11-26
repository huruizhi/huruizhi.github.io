---
layout: post
title: "PreNormLoop-0.6B 模型结构"
date: 2026-11-26 22:30:00 +0800
permalink: /ai/architecture/2026/11/26/prenormloop-structure-notebook/
categories: [AI, architecture]
tags: [Transformer, Loop Transformer, PreNormLoop, Notebook]
description: "用 Transformers 在 meta 设备上查看 18 个共享块的真实模块树，逐行追踪第一轮末层如何把隐藏状态交给第二轮首层。"
source_url: "https://huggingface.co/osunlp/PreNormLoop-0.6B"
---

<style>

.post-content .notebook-cell {
  display: grid;
  gap: .6rem;
  min-width: 0;
  margin: 1.15rem 0 1.6rem;
}
.post-content .notebook-panel {
  min-width: 0;
  overflow: hidden;
  border: 1px solid var(--panel-border);
  border-inline-start: 3px solid var(--panel-mark);
  border-radius: .65rem;
  background: var(--panel-bg);
}
.post-content .notebook-input {
  --panel-bg: #101a29;
  --panel-border: #2c4668;
  --panel-mark: #6ca8ef;
  --panel-label: #a8cfff;
}
.post-content .notebook-output {
  --panel-bg: #101e18;
  --panel-border: #2d5143;
  --panel-mark: #54bf98;
  --panel-label: #abebca;
}
.post-content .notebook-panel__label {
  padding: .38rem .8rem;
  border-bottom: 1px solid var(--panel-border);
  font-size: .82rem;
  font-weight: 700;
  color: var(--panel-label);
}
.post-content .notebook-panel pre {
  margin: 0;
  padding: .65rem .8rem;
  overflow-x: auto;
  border: 0;
  border-radius: 0;
  background: transparent;
}
.post-content .notebook-panel pre.has-copy { padding-inline-end: 4rem; }
.post-content .notebook-output pre { color: #d7eadf; }
.post-content .notebook-output pre + pre { border-top: 1px solid var(--panel-border); }
.post-content .notebook-output .table-wrap { margin: .45rem .8rem; max-width: 100%; }
.post-content .notebook-output table { min-width: 100%; margin: 0; }
.post-content .notebook-tree { margin: 1rem 0 1.6rem; }
.post-content .notebook-tree > summary { cursor: pointer; }
.post-content .notebook-tree .notebook-cell { margin: .7rem 0 0; }
.post-content .notebook-input code .c { color: #8B949E; font-style: italic } /* Comment */
.post-content .notebook-input code .err { color: #F85149 } /* Error */
.post-content .notebook-input code .esc { color: #E6EDF3 } /* Escape */
.post-content .notebook-input code .g { color: #E6EDF3 } /* Generic */
.post-content .notebook-input code .k { color: #FF7B72 } /* Keyword */
.post-content .notebook-input code .l { color: #A5D6FF } /* Literal */
.post-content .notebook-input code .n { color: #E6EDF3 } /* Name */
.post-content .notebook-input code .o { color: #FF7B72; font-weight: bold } /* Operator */
.post-content .notebook-input code .x { color: #E6EDF3 } /* Other */
.post-content .notebook-input code .p { color: #E6EDF3 } /* Punctuation */
.post-content .notebook-input code .ch { color: #8B949E; font-style: italic } /* Comment.Hashbang */
.post-content .notebook-input code .cm { color: #8B949E; font-style: italic } /* Comment.Multiline */
.post-content .notebook-input code .cp { color: #8B949E; font-weight: bold; font-style: italic } /* Comment.Preproc */
.post-content .notebook-input code .cpf { color: #8B949E; font-style: italic } /* Comment.PreprocFile */
.post-content .notebook-input code .c1 { color: #8B949E; font-style: italic } /* Comment.Single */
.post-content .notebook-input code .cs { color: #8B949E; font-weight: bold; font-style: italic } /* Comment.Special */
.post-content .notebook-input code .gd { color: #FFA198; background-color: #490202 } /* Generic.Deleted */
.post-content .notebook-input code .ge { color: #E6EDF3; font-style: italic } /* Generic.Emph */
.post-content .notebook-input code .ges { color: #E6EDF3; font-weight: bold; font-style: italic } /* Generic.EmphStrong */
.post-content .notebook-input code .gr { color: #FFA198 } /* Generic.Error */
.post-content .notebook-input code .gh { color: #79C0FF; font-weight: bold } /* Generic.Heading */
.post-content .notebook-input code .gi { color: #56D364; background-color: #0F5323 } /* Generic.Inserted */
.post-content .notebook-input code .go { color: #8B949E } /* Generic.Output */
.post-content .notebook-input code .gp { color: #8B949E } /* Generic.Prompt */
.post-content .notebook-input code .gs { color: #E6EDF3; font-weight: bold } /* Generic.Strong */
.post-content .notebook-input code .gu { color: #79C0FF } /* Generic.Subheading */
.post-content .notebook-input code .gt { color: #FF7B72 } /* Generic.Traceback */
.post-content .notebook-input code .g-Underline { color: #E6EDF3; text-decoration: underline } /* Generic.Underline */
.post-content .notebook-input code .kc { color: #79C0FF } /* Keyword.Constant */
.post-content .notebook-input code .kd { color: #FF7B72 } /* Keyword.Declaration */
.post-content .notebook-input code .kn { color: #FF7B72 } /* Keyword.Namespace */
.post-content .notebook-input code .kp { color: #79C0FF } /* Keyword.Pseudo */
.post-content .notebook-input code .kr { color: #FF7B72 } /* Keyword.Reserved */
.post-content .notebook-input code .kt { color: #FF7B72 } /* Keyword.Type */
.post-content .notebook-input code .ld { color: #79C0FF } /* Literal.Date */
.post-content .notebook-input code .m { color: #A5D6FF } /* Literal.Number */
.post-content .notebook-input code .s { color: #A5D6FF } /* Literal.String */
.post-content .notebook-input code .na { color: #E6EDF3 } /* Name.Attribute */
.post-content .notebook-input code .nb { color: #E6EDF3 } /* Name.Builtin */
.post-content .notebook-input code .nc { color: #F0883E; font-weight: bold } /* Name.Class */
.post-content .notebook-input code .no { color: #79C0FF; font-weight: bold } /* Name.Constant */
.post-content .notebook-input code .nd { color: #D2A8FF; font-weight: bold } /* Name.Decorator */
.post-content .notebook-input code .ni { color: #FFA657 } /* Name.Entity */
.post-content .notebook-input code .ne { color: #F0883E; font-weight: bold } /* Name.Exception */
.post-content .notebook-input code .nf { color: #D2A8FF; font-weight: bold } /* Name.Function */
.post-content .notebook-input code .nl { color: #79C0FF; font-weight: bold } /* Name.Label */
.post-content .notebook-input code .nn { color: #FF7B72 } /* Name.Namespace */
.post-content .notebook-input code .nx { color: #E6EDF3 } /* Name.Other */
.post-content .notebook-input code .py { color: #79C0FF } /* Name.Property */
.post-content .notebook-input code .nt { color: #7EE787 } /* Name.Tag */
.post-content .notebook-input code .nv { color: #79C0FF } /* Name.Variable */
.post-content .notebook-input code .ow { color: #FF7B72; font-weight: bold } /* Operator.Word */
.post-content .notebook-input code .pm { color: #E6EDF3 } /* Punctuation.Marker */
.post-content .notebook-input code .w { color: #6E7681 } /* Text.Whitespace */
.post-content .notebook-input code .mb { color: #A5D6FF } /* Literal.Number.Bin */
.post-content .notebook-input code .mf { color: #A5D6FF } /* Literal.Number.Float */
.post-content .notebook-input code .mh { color: #A5D6FF } /* Literal.Number.Hex */
.post-content .notebook-input code .mi { color: #A5D6FF } /* Literal.Number.Integer */
.post-content .notebook-input code .mo { color: #A5D6FF } /* Literal.Number.Oct */
.post-content .notebook-input code .sa { color: #79C0FF } /* Literal.String.Affix */
.post-content .notebook-input code .sb { color: #A5D6FF } /* Literal.String.Backtick */
.post-content .notebook-input code .sc { color: #A5D6FF } /* Literal.String.Char */
.post-content .notebook-input code .dl { color: #79C0FF } /* Literal.String.Delimiter */
.post-content .notebook-input code .sd { color: #A5D6FF } /* Literal.String.Doc */
.post-content .notebook-input code .s2 { color: #A5D6FF } /* Literal.String.Double */
.post-content .notebook-input code .se { color: #79C0FF } /* Literal.String.Escape */
.post-content .notebook-input code .sh { color: #79C0FF } /* Literal.String.Heredoc */
.post-content .notebook-input code .si { color: #A5D6FF } /* Literal.String.Interpol */
.post-content .notebook-input code .sx { color: #A5D6FF } /* Literal.String.Other */
.post-content .notebook-input code .sr { color: #79C0FF } /* Literal.String.Regex */
.post-content .notebook-input code .s1 { color: #A5D6FF } /* Literal.String.Single */
.post-content .notebook-input code .ss { color: #A5D6FF } /* Literal.String.Symbol */
.post-content .notebook-input code .bp { color: #E6EDF3 } /* Name.Builtin.Pseudo */
.post-content .notebook-input code .fm { color: #D2A8FF; font-weight: bold } /* Name.Function.Magic */
.post-content .notebook-input code .vc { color: #79C0FF } /* Name.Variable.Class */
.post-content .notebook-input code .vg { color: #79C0FF } /* Name.Variable.Global */
.post-content .notebook-input code .vi { color: #79C0FF } /* Name.Variable.Instance */
.post-content .notebook-input code .vm { color: #79C0FF } /* Name.Variable.Magic */
.post-content .notebook-input code .il { color: #A5D6FF } /* Literal.Number.Integer.Long */
</style>

<p><a href="{{ '/assets/notebooks/prenormloop-0.6b-structure.ipynb' | relative_url }}" download>下载 .ipynb</a> · <a href="{{ '/ai/architecture/loop-transformer/' | relative_url }}">返回 Loop Transformer 交互讲解</a></p>

**18 个独立 Transformer 块，循环 3 轮，共执行 54 次。** 先看整体路径，再从模型对象里读模块、线性层尺寸和跨轮数据流。

Notebook 只读取固定版本的配置与 Python 源码。模型骨架在 `meta` 设备上构建，分配的模型权重存储为 **0 字节**。文中的张量形状由配置和模块属性推导。 [模型仓库](https://huggingface.co/osunlp/PreNormLoop-0.6B) · [发布者实现](https://github.com/OSU-NLP-Group/ARLoop)

## 1. 先看整体结构

Token ID → Token Embedding → **同一个 18 层共享栈 ×3** → 最后一次 RMSNorm → Logits

每个块有两次残差更新，分别由 RMSNorm → Attention 和 RMSNorm → MLP 产生。目标是看清一轮结束后，隐藏状态怎样流入下一轮。 [发布配置](https://huggingface.co/osunlp/PreNormLoop-0.6B/blob/e4ad59d0d03ab78f83d9f52b0c7d7267f372a151/config.json)

<div class="notebook-cell">
<section class="notebook-panel notebook-input" aria-label="代码"><div class="notebook-panel__label">代码</div><pre><code class="language-python"><span class="kn">import</span><span class="w"> </span><span class="nn">hashlib</span>
<span class="kn">import</span><span class="w"> </span><span class="nn">json</span>
<span class="kn">from</span><span class="w"> </span><span class="nn">urllib.request</span><span class="w"> </span><span class="kn">import</span> <span class="n">Request</span><span class="p">,</span> <span class="n">urlopen</span>

<span class="kn">from</span><span class="w"> </span><span class="nn">IPython.display</span><span class="w"> </span><span class="kn">import</span> <span class="n">Markdown</span><span class="p">,</span> <span class="n">display</span>

<span class="n">revision</span> <span class="o">=</span> <span class="s2">"e4ad59d0d03ab78f83d9f52b0c7d7267f372a151"</span>
<span class="n">repo</span> <span class="o">=</span> <span class="s2">"https://huggingface.co/osunlp/PreNormLoop-0.6B"</span>
<span class="n">files</span> <span class="o">=</span> <span class="p">{</span>
    <span class="s2">"config.json"</span><span class="p">:</span> <span class="s2">"1e0b2e41b5ca188cdbddc8c8313d7f2b84c49dbf9772c1ca6626be4efedc78ea"</span><span class="p">,</span>
    <span class="s2">"configuration_baseline.py"</span><span class="p">:</span> <span class="s2">"f1cac2589a855c9e99015957a766f75fa854b7f28928c1dcfe0aab5cc126a092"</span><span class="p">,</span>
    <span class="s2">"modeling_baseline.py"</span><span class="p">:</span> <span class="s2">"ed5139ce9497369f5263ceb58ca913042c8bde9fd1193ff71135f1116359c969"</span><span class="p">,</span>
<span class="p">}</span>


<span class="k">def</span><span class="w"> </span><span class="nf">read_public_text</span><span class="p">(</span><span class="n">filename</span><span class="p">):</span>
    <span class="n">expected_sha256</span> <span class="o">=</span> <span class="n">files</span><span class="p">[</span><span class="n">filename</span><span class="p">]</span>
    <span class="n">request</span> <span class="o">=</span> <span class="n">Request</span><span class="p">(</span>
        <span class="sa">f</span><span class="s2">"</span><span class="si">{</span><span class="n">repo</span><span class="si">}</span><span class="s2">/raw/</span><span class="si">{</span><span class="n">revision</span><span class="si">}</span><span class="s2">/</span><span class="si">{</span><span class="n">filename</span><span class="si">}</span><span class="s2">"</span><span class="p">,</span>
        <span class="n">headers</span><span class="o">=</span><span class="p">{</span><span class="s2">"User-Agent"</span><span class="p">:</span> <span class="s2">"Jupyter architecture inspection/1.0"</span><span class="p">},</span>
    <span class="p">)</span>
    <span class="k">with</span> <span class="n">urlopen</span><span class="p">(</span><span class="n">request</span><span class="p">,</span> <span class="n">timeout</span><span class="o">=</span><span class="mi">25</span><span class="p">)</span> <span class="k">as</span> <span class="n">response</span><span class="p">:</span>
        <span class="n">data</span> <span class="o">=</span> <span class="n">response</span><span class="o">.</span><span class="n">read</span><span class="p">(</span><span class="mi">64</span> <span class="o">*</span> <span class="mi">1024</span> <span class="o">+</span> <span class="mi">1</span><span class="p">)</span>
    <span class="k">if</span> <span class="nb">len</span><span class="p">(</span><span class="n">data</span><span class="p">)</span> <span class="o">&gt;</span> <span class="mi">64</span> <span class="o">*</span> <span class="mi">1024</span> <span class="ow">or</span> <span class="n">hashlib</span><span class="o">.</span><span class="n">sha256</span><span class="p">(</span><span class="n">data</span><span class="p">)</span><span class="o">.</span><span class="n">hexdigest</span><span class="p">()</span> <span class="o">!=</span> <span class="n">expected_sha256</span><span class="p">:</span>
        <span class="k">raise</span> <span class="ne">ValueError</span><span class="p">(</span><span class="sa">f</span><span class="s2">"文件大小或 SHA-256 校验失败：</span><span class="si">{</span><span class="n">filename</span><span class="si">}</span><span class="s2">"</span><span class="p">)</span>
    <span class="nb">print</span><span class="p">(</span><span class="sa">f</span><span class="s2">"</span><span class="si">{</span><span class="n">filename</span><span class="si">:</span><span class="s2">22</span><span class="si">}</span><span class="s2"> </span><span class="si">{</span><span class="nb">len</span><span class="p">(</span><span class="n">data</span><span class="p">)</span><span class="si">:</span><span class="s2">,</span><span class="si">}</span><span class="s2"> bytes"</span><span class="p">)</span>
    <span class="k">return</span> <span class="n">data</span><span class="o">.</span><span class="n">decode</span><span class="p">(</span><span class="s2">"utf-8"</span><span class="p">)</span>


<span class="n">config_text</span> <span class="o">=</span> <span class="n">read_public_text</span><span class="p">(</span><span class="s2">"config.json"</span><span class="p">)</span>
<span class="n">cfg</span> <span class="o">=</span> <span class="n">json</span><span class="o">.</span><span class="n">loads</span><span class="p">(</span><span class="n">config_text</span><span class="p">)</span>
<span class="n">layer_count</span> <span class="o">=</span> <span class="n">cfg</span><span class="p">[</span><span class="s2">"physical_layers"</span><span class="p">]</span>
<span class="n">loop_count</span> <span class="o">=</span> <span class="n">cfg</span><span class="p">[</span><span class="s2">"loop_count"</span><span class="p">]</span>
<span class="n">effective_depth</span> <span class="o">=</span> <span class="n">layer_count</span> <span class="o">*</span> <span class="n">loop_count</span>
<span class="k">assert</span> <span class="n">effective_depth</span> <span class="o">==</span> <span class="n">cfg</span><span class="p">[</span><span class="s2">"num_hidden_layers"</span><span class="p">]</span>

<span class="n">rows</span> <span class="o">=</span> <span class="p">[</span>
    <span class="p">(</span><span class="s2">"独立 Transformer 层"</span><span class="p">,</span> <span class="s2">"physical_layers"</span><span class="p">),</span>
    <span class="p">(</span><span class="s2">"循环次数"</span><span class="p">,</span> <span class="s2">"loop_count"</span><span class="p">),</span>
    <span class="p">(</span><span class="s2">"有效层计算次数"</span><span class="p">,</span> <span class="s2">"num_hidden_layers"</span><span class="p">),</span>
    <span class="p">(</span><span class="s2">"Hidden Size"</span><span class="p">,</span> <span class="s2">"hidden_size"</span><span class="p">),</span>
    <span class="p">(</span><span class="s2">"Attention Query Heads"</span><span class="p">,</span> <span class="s2">"num_attention_heads"</span><span class="p">),</span>
    <span class="p">(</span><span class="s2">"Attention KV Heads"</span><span class="p">,</span> <span class="s2">"num_key_value_heads"</span><span class="p">),</span>
    <span class="p">(</span><span class="s2">"每个 Attention Head 的维度"</span><span class="p">,</span> <span class="s2">"head_dim"</span><span class="p">),</span>
    <span class="p">(</span><span class="s2">"MLP 中间维度"</span><span class="p">,</span> <span class="s2">"intermediate_size"</span><span class="p">),</span>
<span class="p">]</span>
<span class="n">table</span> <span class="o">=</span> <span class="p">[</span><span class="s2">"| 读什么 | 配置字段 | 值 |"</span><span class="p">,</span> <span class="s2">"|---|---|---:|"</span><span class="p">]</span>
<span class="n">table</span><span class="o">.</span><span class="n">extend</span><span class="p">(</span><span class="sa">f</span><span class="s2">"| </span><span class="si">{</span><span class="n">label</span><span class="si">}</span><span class="s2"> | `</span><span class="si">{</span><span class="n">key</span><span class="si">}</span><span class="s2">` | </span><span class="si">{</span><span class="n">cfg</span><span class="p">[</span><span class="n">key</span><span class="p">]</span><span class="si">:</span><span class="s2">,</span><span class="si">}</span><span class="s2"> |"</span> <span class="k">for</span> <span class="n">label</span><span class="p">,</span> <span class="n">key</span> <span class="ow">in</span> <span class="n">rows</span><span class="p">)</span>
<span class="n">display</span><span class="p">(</span><span class="n">Markdown</span><span class="p">(</span><span class="s2">"</span><span class="se">\n</span><span class="s2">"</span><span class="o">.</span><span class="n">join</span><span class="p">(</span><span class="n">table</span><span class="p">)))</span></code></pre></section>
<section class="notebook-panel notebook-output" aria-label="运行输出"><div class="notebook-panel__label">运行输出</div><pre><samp>config.json            861 bytes</samp></pre><div class="table-wrap"><table>
<thead>
<tr>
  <th>读什么</th>
  <th>配置字段</th>
  <th style="text-align:right">值</th>
</tr>
</thead>
<tbody>
<tr>
  <td>独立 Transformer 层</td>
  <td><code>physical_layers</code></td>
  <td style="text-align:right">18</td>
</tr>
<tr>
  <td>循环次数</td>
  <td><code>loop_count</code></td>
  <td style="text-align:right">3</td>
</tr>
<tr>
  <td>有效层计算次数</td>
  <td><code>num_hidden_layers</code></td>
  <td style="text-align:right">54</td>
</tr>
<tr>
  <td>Hidden Size</td>
  <td><code>hidden_size</code></td>
  <td style="text-align:right">1,536</td>
</tr>
<tr>
  <td>Attention Query Heads</td>
  <td><code>num_attention_heads</code></td>
  <td style="text-align:right">16</td>
</tr>
<tr>
  <td>Attention KV Heads</td>
  <td><code>num_key_value_heads</code></td>
  <td style="text-align:right">4</td>
</tr>
<tr>
  <td>每个 Attention Head 的维度</td>
  <td><code>head_dim</code></td>
  <td style="text-align:right">96</td>
</tr>
<tr>
  <td>MLP 中间维度</td>
  <td><code>intermediate_size</code></td>
  <td style="text-align:right">4,608</td>
</tr>
</tbody>
</table></div></section>
</div>

这里 `num_hidden_layers` 表示展开后的执行深度。接下来回到构造代码，看看真正创建了多少个块。

## 2. 查看真实模块树

用发布者定义的 `ARLoopBaselineForCausalLM` 读取实际模块。`model.transformer` 是 `ModuleDict`，包含 Token Embedding `wte`、18 层共享栈 `h` 和最终的 `ln_f`。`shared_value_embedding` 挂在模型根节点，供有 Value 门控的层使用。

| 路径 | 在模型里的职责 |
|---|---|
| `model.transformer.wte` | Token ID → 1,536 维向量，输出投影也读取这张权重表 |
| `model.transformer.h` | 18 个 `PreNormBlock`，每轮按同一顺序调用 |
| `model.transformer.ln_f` | 所有循环结束后，对最后的隐藏状态做 RMSNorm |
| `model.shared_value_embedding` | Token ID → 384 维的 Value 向量，供部分 Attention 层使用 |

构建用 `AutoConfig` 和 `AutoModelForCausalLM.from_config`。`trust_remote_code=True` 载入的是固定版本、经哈希校验的本地 Python 文件，`no_init_weights()` 跳过权重初始化。下面先核对 18 层，再展开完整模块树。 [构造源码](https://huggingface.co/osunlp/PreNormLoop-0.6B/blob/e4ad59d0d03ab78f83d9f52b0c7d7267f372a151/modeling_baseline.py#L103)

<div class="notebook-cell">
<section class="notebook-panel notebook-input" aria-label="代码"><div class="notebook-panel__label">代码</div><pre><code class="language-python"><span class="kn">from</span><span class="w"> </span><span class="nn">pathlib</span><span class="w"> </span><span class="kn">import</span> <span class="n">Path</span>
<span class="kn">from</span><span class="w"> </span><span class="nn">tempfile</span><span class="w"> </span><span class="kn">import</span> <span class="n">TemporaryDirectory</span>

<span class="kn">import</span><span class="w"> </span><span class="nn">torch</span>
<span class="kn">import</span><span class="w"> </span><span class="nn">transformers</span>
<span class="kn">from</span><span class="w"> </span><span class="nn">transformers</span><span class="w"> </span><span class="kn">import</span> <span class="n">AutoConfig</span><span class="p">,</span> <span class="n">AutoModelForCausalLM</span>
<span class="kn">from</span><span class="w"> </span><span class="nn">transformers.initialization</span><span class="w"> </span><span class="kn">import</span> <span class="n">no_init_weights</span>

<span class="n">source_files</span> <span class="o">=</span> <span class="p">{</span>
    <span class="s2">"config.json"</span><span class="p">:</span> <span class="n">config_text</span><span class="p">,</span>
    <span class="s2">"configuration_baseline.py"</span><span class="p">:</span> <span class="n">read_public_text</span><span class="p">(</span><span class="s2">"configuration_baseline.py"</span><span class="p">),</span>
    <span class="s2">"modeling_baseline.py"</span><span class="p">:</span> <span class="n">read_public_text</span><span class="p">(</span><span class="s2">"modeling_baseline.py"</span><span class="p">),</span>
<span class="p">}</span>

<span class="k">with</span> <span class="n">TemporaryDirectory</span><span class="p">(</span><span class="n">prefix</span><span class="o">=</span><span class="s2">"prenormloop_code_"</span><span class="p">)</span> <span class="k">as</span> <span class="n">directory</span><span class="p">:</span>
    <span class="n">source_dir</span> <span class="o">=</span> <span class="n">Path</span><span class="p">(</span><span class="n">directory</span><span class="p">)</span>
    <span class="k">for</span> <span class="n">name</span><span class="p">,</span> <span class="n">content</span> <span class="ow">in</span> <span class="n">source_files</span><span class="o">.</span><span class="n">items</span><span class="p">():</span>
        <span class="p">(</span><span class="n">source_dir</span> <span class="o">/</span> <span class="n">name</span><span class="p">)</span><span class="o">.</span><span class="n">write_text</span><span class="p">(</span><span class="n">content</span><span class="p">,</span> <span class="n">encoding</span><span class="o">=</span><span class="s2">"utf-8"</span><span class="p">)</span>

    <span class="n">model_config</span> <span class="o">=</span> <span class="n">AutoConfig</span><span class="o">.</span><span class="n">from_pretrained</span><span class="p">(</span>
        <span class="n">source_dir</span><span class="p">,</span> <span class="n">local_files_only</span><span class="o">=</span><span class="kc">True</span><span class="p">,</span> <span class="n">trust_remote_code</span><span class="o">=</span><span class="kc">True</span>
    <span class="p">)</span>
    <span class="k">assert</span> <span class="n">Path</span><span class="p">(</span><span class="n">model_config</span><span class="o">.</span><span class="n">name_or_path</span><span class="p">)</span> <span class="o">==</span> <span class="n">source_dir</span>
    <span class="n">rng_before</span> <span class="o">=</span> <span class="n">torch</span><span class="o">.</span><span class="n">get_rng_state</span><span class="p">()</span>
    <span class="k">with</span> <span class="n">torch</span><span class="o">.</span><span class="n">device</span><span class="p">(</span><span class="s2">"meta"</span><span class="p">),</span> <span class="n">no_init_weights</span><span class="p">():</span>
        <span class="n">model</span> <span class="o">=</span> <span class="n">AutoModelForCausalLM</span><span class="o">.</span><span class="n">from_config</span><span class="p">(</span><span class="n">model_config</span><span class="p">,</span> <span class="n">trust_remote_code</span><span class="o">=</span><span class="kc">True</span><span class="p">)</span>
    <span class="k">assert</span> <span class="n">torch</span><span class="o">.</span><span class="n">equal</span><span class="p">(</span><span class="n">rng_before</span><span class="p">,</span> <span class="n">torch</span><span class="o">.</span><span class="n">get_rng_state</span><span class="p">())</span>

<span class="n">parameters</span> <span class="o">=</span> <span class="nb">list</span><span class="p">(</span><span class="n">model</span><span class="o">.</span><span class="n">parameters</span><span class="p">())</span>
<span class="k">assert</span> <span class="n">parameters</span> <span class="ow">and</span> <span class="nb">all</span><span class="p">(</span>
    <span class="n">parameter</span><span class="o">.</span><span class="n">is_meta</span> <span class="ow">and</span> <span class="n">parameter</span><span class="o">.</span><span class="n">untyped_storage</span><span class="p">()</span><span class="o">.</span><span class="n">data_ptr</span><span class="p">()</span> <span class="o">==</span> <span class="mi">0</span>
    <span class="k">for</span> <span class="n">parameter</span> <span class="ow">in</span> <span class="n">parameters</span>
<span class="p">)</span>
<span class="k">assert</span> <span class="nb">all</span><span class="p">(</span><span class="n">buffer</span><span class="o">.</span><span class="n">is_meta</span> <span class="k">for</span> <span class="n">buffer</span> <span class="ow">in</span> <span class="n">model</span><span class="o">.</span><span class="n">buffers</span><span class="p">())</span>
<span class="k">assert</span> <span class="nb">type</span><span class="p">(</span><span class="n">model</span><span class="p">)</span><span class="o">.</span><span class="vm">__name__</span> <span class="o">==</span> <span class="n">cfg</span><span class="p">[</span><span class="s2">"architectures"</span><span class="p">][</span><span class="mi">0</span><span class="p">]</span>

<span class="nb">print</span><span class="p">(</span><span class="sa">f</span><span class="s2">"PyTorch </span><span class="si">{</span><span class="n">torch</span><span class="o">.</span><span class="n">__version__</span><span class="si">}</span><span class="s2"> · Transformers </span><span class="si">{</span><span class="n">transformers</span><span class="o">.</span><span class="n">__version__</span><span class="si">}</span><span class="s2">"</span><span class="p">)</span>
<span class="nb">print</span><span class="p">(</span><span class="sa">f</span><span class="s2">"模型类型：</span><span class="si">{</span><span class="nb">type</span><span class="p">(</span><span class="n">model</span><span class="p">)</span><span class="o">.</span><span class="vm">__name__</span><span class="si">}</span><span class="s2">"</span><span class="p">)</span>
<span class="nb">print</span><span class="p">(</span><span class="sa">f</span><span class="s2">"逻辑参数量：</span><span class="si">{</span><span class="nb">sum</span><span class="p">(</span><span class="n">parameter</span><span class="o">.</span><span class="n">numel</span><span class="p">()</span><span class="w"> </span><span class="k">for</span><span class="w"> </span><span class="n">parameter</span><span class="w"> </span><span class="ow">in</span><span class="w"> </span><span class="n">parameters</span><span class="p">)</span><span class="si">:</span><span class="s2">,</span><span class="si">}</span><span class="s2">"</span><span class="p">)</span>
<span class="nb">print</span><span class="p">(</span><span class="sa">f</span><span class="s2">"</span><span class="si">{</span><span class="nb">len</span><span class="p">(</span><span class="n">parameters</span><span class="p">)</span><span class="si">}</span><span class="s2"> 个参数张量均在 meta，实际参数存储为 0 字节"</span><span class="p">)</span>
<span class="nb">print</span><span class="p">(</span><span class="s2">"CPU 随机数状态未变"</span><span class="p">)</span></code></pre></section>
<section class="notebook-panel notebook-output" aria-label="运行输出"><div class="notebook-panel__label">运行输出</div><pre><samp>configuration_baseline.py 1,909 bytes</samp></pre><pre><samp>modeling_baseline.py   10,004 bytes</samp></pre><pre><samp>PyTorch 2.15.0 · Transformers 5.22.0
模型类型：ARLoopBaselineForCausalLM
逻辑参数量：582,804,096
156 个参数张量均在 meta，实际参数存储为 0 字节
CPU 随机数状态未变</samp></pre></section>
</div>

<div class="notebook-cell">
<section class="notebook-panel notebook-input" aria-label="代码"><div class="notebook-panel__label">代码</div><pre><code class="language-python"><span class="n">shared_layers</span> <span class="o">=</span> <span class="n">model</span><span class="o">.</span><span class="n">transformer</span><span class="o">.</span><span class="n">h</span>
<span class="k">assert</span> <span class="nb">isinstance</span><span class="p">(</span><span class="n">shared_layers</span><span class="p">,</span> <span class="n">torch</span><span class="o">.</span><span class="n">nn</span><span class="o">.</span><span class="n">ModuleList</span><span class="p">)</span>
<span class="k">assert</span> <span class="nb">len</span><span class="p">(</span><span class="n">shared_layers</span><span class="p">)</span> <span class="o">==</span> <span class="n">layer_count</span> <span class="o">==</span> <span class="mi">18</span>
<span class="k">assert</span> <span class="nb">len</span><span class="p">({</span><span class="nb">id</span><span class="p">(</span><span class="n">layer</span><span class="p">)</span> <span class="k">for</span> <span class="n">layer</span> <span class="ow">in</span> <span class="n">shared_layers</span><span class="p">})</span> <span class="o">==</span> <span class="mi">18</span>

<span class="nb">print</span><span class="p">(</span><span class="sa">f</span><span class="s2">"model.transformer.h：</span><span class="si">{</span><span class="nb">type</span><span class="p">(</span><span class="n">shared_layers</span><span class="p">)</span><span class="o">.</span><span class="vm">__name__</span><span class="si">}</span><span class="s2">，</span><span class="si">{</span><span class="nb">len</span><span class="p">(</span><span class="n">shared_layers</span><span class="p">)</span><span class="si">}</span><span class="s2"> 层"</span><span class="p">)</span>
<span class="k">for</span> <span class="n">index</span><span class="p">,</span> <span class="n">layer</span> <span class="ow">in</span> <span class="nb">enumerate</span><span class="p">(</span><span class="n">shared_layers</span><span class="p">):</span>
    <span class="nb">print</span><span class="p">(</span><span class="sa">f</span><span class="s2">"  h[</span><span class="si">{</span><span class="n">index</span><span class="si">:</span><span class="s2">2d</span><span class="si">}</span><span class="s2">]  </span><span class="si">{</span><span class="nb">type</span><span class="p">(</span><span class="n">layer</span><span class="p">)</span><span class="o">.</span><span class="vm">__name__</span><span class="si">}</span><span class="s2">"</span><span class="p">)</span></code></pre></section>
<section class="notebook-panel notebook-output" aria-label="运行输出"><div class="notebook-panel__label">运行输出</div><pre><samp>model.transformer.h：ModuleList，18 层
  h[ 0]  PreNormBlock
  h[ 1]  PreNormBlock
  h[ 2]  PreNormBlock
  h[ 3]  PreNormBlock
  h[ 4]  PreNormBlock
  h[ 5]  PreNormBlock
  h[ 6]  PreNormBlock
  h[ 7]  PreNormBlock
  h[ 8]  PreNormBlock
  h[ 9]  PreNormBlock
  h[10]  PreNormBlock
  h[11]  PreNormBlock
  h[12]  PreNormBlock
  h[13]  PreNormBlock
  h[14]  PreNormBlock
  h[15]  PreNormBlock
  h[16]  PreNormBlock
  h[17]  PreNormBlock</samp></pre></section>
</div>

<details class="notebook-tree" open>
<summary>完整模块树（18 层）</summary>
<div class="notebook-cell">
<section class="notebook-panel notebook-input" aria-label="代码"><div class="notebook-panel__label">代码</div><pre><code class="language-python"><span class="nb">print</span><span class="p">(</span><span class="n">model</span><span class="p">)</span></code></pre></section>
<section class="notebook-panel notebook-output" aria-label="运行输出"><div class="notebook-panel__label">运行输出</div><pre><samp>ARLoopBaselineForCausalLM(
  (transformer): ModuleDict(
    (wte): Embedding(49152, 1536)
    (h): ModuleList(
      (0): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (1): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
          (ve_gate): Linear(in_features=32, out_features=4, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (2): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (3): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
          (ve_gate): Linear(in_features=32, out_features=4, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (4): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (5): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
          (ve_gate): Linear(in_features=32, out_features=4, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (6): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (7): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
          (ve_gate): Linear(in_features=32, out_features=4, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (8): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (9): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
          (ve_gate): Linear(in_features=32, out_features=4, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (10): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (11): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
          (ve_gate): Linear(in_features=32, out_features=4, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (12): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (13): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
          (ve_gate): Linear(in_features=32, out_features=4, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (14): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (15): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
          (ve_gate): Linear(in_features=32, out_features=4, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (16): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
      (17): PreNormBlock(
        (norm_1): RMSNorm()
        (attn): Attention(
          (c_q): Linear(in_features=1536, out_features=1536, bias=False)
          (c_k): Linear(in_features=1536, out_features=384, bias=False)
          (c_v): Linear(in_features=1536, out_features=384, bias=False)
          (c_proj): Linear(in_features=1536, out_features=1536, bias=False)
          (ve_gate): Linear(in_features=32, out_features=4, bias=False)
        )
        (norm_2): RMSNorm()
        (mlp): MLP(
          (fc): Linear(in_features=1536, out_features=9216, bias=False)
          (proj): Linear(in_features=4608, out_features=1536, bias=False)
        )
      )
    )
    (ln_f): RMSNorm()
  )
  (shared_value_embedding): Embedding(49152, 384)
)</samp></pre></section>
</div>
</details>

## 3. 拆开一个 PreNormBlock

以 `model.transformer.h[0]` 为例。输入和输出都保持 `[B, T, 1536]`，其中 `B` 是批大小，`T` 是序列长度。两次残差更新的顺序很重要，MLP 读的是 Attention 已更新过的状态。

~~~text
u = h + Attention(RMSNorm₁(h))
h_next = u + MLP(RMSNorm₂(u))
~~~

- **Attention** 使用 GQA。Q 有 16 个头，K 和 V 各有 4 个头，每头 96 维。Q 投影到 1,536 维，K/V 各投影到 384 维。每组 4 个 Q 头共享一个 KV 头。
- **MLP** 把 1,536 维投影到 9,216 维，拆成两个 4,608 维分支。门控分支经 SiLU 后与内容分支逐元素相乘，最后投影回 1,536 维。
- **Value 门控** 出现在 `h[1]`、`h[3]` 等层。它根据当前状态对共享 Value 向量做缩放，再把结果加到 V 上。

下面的尺寸直接读取真实模块的 `in_features` 和 `out_features`。 [Attention 与 MLP 源码](https://huggingface.co/osunlp/PreNormLoop-0.6B/blob/e4ad59d0d03ab78f83d9f52b0c7d7267f372a151/modeling_baseline.py#L27)

<div class="notebook-cell">
<section class="notebook-panel notebook-input" aria-label="代码"><div class="notebook-panel__label">代码</div><pre><code class="language-python"><span class="n">block</span> <span class="o">=</span> <span class="n">shared_layers</span><span class="p">[</span><span class="mi">0</span><span class="p">]</span>
<span class="n">linears</span> <span class="o">=</span> <span class="p">[</span>
    <span class="p">(</span><span class="s2">"Attention · Q"</span><span class="p">,</span> <span class="n">block</span><span class="o">.</span><span class="n">attn</span><span class="o">.</span><span class="n">c_q</span><span class="p">),</span>
    <span class="p">(</span><span class="s2">"Attention · K"</span><span class="p">,</span> <span class="n">block</span><span class="o">.</span><span class="n">attn</span><span class="o">.</span><span class="n">c_k</span><span class="p">),</span>
    <span class="p">(</span><span class="s2">"Attention · V"</span><span class="p">,</span> <span class="n">block</span><span class="o">.</span><span class="n">attn</span><span class="o">.</span><span class="n">c_v</span><span class="p">),</span>
    <span class="p">(</span><span class="s2">"Attention · 输出投影"</span><span class="p">,</span> <span class="n">block</span><span class="o">.</span><span class="n">attn</span><span class="o">.</span><span class="n">c_proj</span><span class="p">),</span>
    <span class="p">(</span><span class="s2">"MLP · 门控和内容分支"</span><span class="p">,</span> <span class="n">block</span><span class="o">.</span><span class="n">mlp</span><span class="o">.</span><span class="n">fc</span><span class="p">),</span>
    <span class="p">(</span><span class="s2">"MLP · 输出投影"</span><span class="p">,</span> <span class="n">block</span><span class="o">.</span><span class="n">mlp</span><span class="o">.</span><span class="n">proj</span><span class="p">),</span>
<span class="p">]</span>
<span class="n">table</span> <span class="o">=</span> <span class="p">[</span><span class="s2">"| 投影 | 输入维度 | 输出维度 |"</span><span class="p">,</span> <span class="s2">"|---|---:|---:|"</span><span class="p">]</span>
<span class="k">for</span> <span class="n">label</span><span class="p">,</span> <span class="n">layer</span> <span class="ow">in</span> <span class="n">linears</span><span class="p">:</span>
    <span class="n">table</span><span class="o">.</span><span class="n">append</span><span class="p">(</span><span class="sa">f</span><span class="s2">"| </span><span class="si">{</span><span class="n">label</span><span class="si">}</span><span class="s2"> | </span><span class="si">{</span><span class="n">layer</span><span class="o">.</span><span class="n">in_features</span><span class="si">:</span><span class="s2">,</span><span class="si">}</span><span class="s2"> | </span><span class="si">{</span><span class="n">layer</span><span class="o">.</span><span class="n">out_features</span><span class="si">:</span><span class="s2">,</span><span class="si">}</span><span class="s2"> |"</span><span class="p">)</span>
<span class="n">display</span><span class="p">(</span><span class="n">Markdown</span><span class="p">(</span><span class="s2">"</span><span class="se">\n</span><span class="s2">"</span><span class="o">.</span><span class="n">join</span><span class="p">(</span><span class="n">table</span><span class="p">)))</span>

<span class="n">gated</span> <span class="o">=</span> <span class="p">[</span><span class="n">i</span> <span class="k">for</span> <span class="n">i</span><span class="p">,</span> <span class="n">layer</span> <span class="ow">in</span> <span class="nb">enumerate</span><span class="p">(</span><span class="n">shared_layers</span><span class="p">)</span> <span class="k">if</span> <span class="n">layer</span><span class="o">.</span><span class="n">attn</span><span class="o">.</span><span class="n">ve_gate</span> <span class="ow">is</span> <span class="ow">not</span> <span class="kc">None</span><span class="p">]</span>
<span class="nb">print</span><span class="p">(</span><span class="s2">"带 Value 门控的层索引"</span><span class="p">,</span> <span class="n">gated</span><span class="p">)</span>
<span class="nb">print</span><span class="p">(</span><span class="sa">f</span><span class="s2">"Q / KV：</span><span class="si">{</span><span class="n">model</span><span class="o">.</span><span class="n">config</span><span class="o">.</span><span class="n">num_attention_heads</span><span class="si">}</span><span class="s2"> / "</span>
      <span class="sa">f</span><span class="s2">"</span><span class="si">{</span><span class="n">model</span><span class="o">.</span><span class="n">config</span><span class="o">.</span><span class="n">num_key_value_heads</span><span class="si">}</span><span class="s2"> 个头，单头 </span><span class="si">{</span><span class="n">model</span><span class="o">.</span><span class="n">config</span><span class="o">.</span><span class="n">head_dim</span><span class="si">}</span><span class="s2"> 维"</span><span class="p">)</span></code></pre></section>
<section class="notebook-panel notebook-output" aria-label="运行输出"><div class="notebook-panel__label">运行输出</div><div class="table-wrap"><table>
<thead>
<tr>
  <th>投影</th>
  <th style="text-align:right">输入维度</th>
  <th style="text-align:right">输出维度</th>
</tr>
</thead>
<tbody>
<tr>
  <td>Attention · Q</td>
  <td style="text-align:right">1,536</td>
  <td style="text-align:right">1,536</td>
</tr>
<tr>
  <td>Attention · K</td>
  <td style="text-align:right">1,536</td>
  <td style="text-align:right">384</td>
</tr>
<tr>
  <td>Attention · V</td>
  <td style="text-align:right">1,536</td>
  <td style="text-align:right">384</td>
</tr>
<tr>
  <td>Attention · 输出投影</td>
  <td style="text-align:right">1,536</td>
  <td style="text-align:right">1,536</td>
</tr>
<tr>
  <td>MLP · 门控和内容分支</td>
  <td style="text-align:right">1,536</td>
  <td style="text-align:right">9,216</td>
</tr>
<tr>
  <td>MLP · 输出投影</td>
  <td style="text-align:right">4,608</td>
  <td style="text-align:right">1,536</td>
</tr>
</tbody>
</table></div><pre><samp>带 Value 门控的层索引 [1, 3, 5, 7, 9, 11, 13, 15, 17]
Q / KV：16 / 4 个头，单头 96 维</samp></pre></section>
</div>

## 4. 三轮怎么接起来

下面是 `forward` 第 163–172 行的原始源码，移除了片段外层的统一缩进。局部变量 `h` 保存隐藏状态，`self.transformer.h` 保存 18 个可重复调用的模块。 [查看发布者源码](https://huggingface.co/osunlp/PreNormLoop-0.6B/blob/e4ad59d0d03ab78f83d9f52b0c7d7267f372a151/modeling_baseline.py#L163-L172)

~~~python
h = self.transformer.wte(input_ids) * self.config.embedding_scale
ve = self.shared_value_embedding(input_ids)
states = [h] if output_hidden_states else None
for loop in range(self.config.loop_count):
    for i, block in enumerate(self.transformer.h):
        effective = loop * self.config.physical_layers + i
        h = h + block.attn(block.norm_1(h), position_ids, mask, ve, cache, effective)
        h = h + block.mlp(block.norm_2(h))
        if states is not None:
            states.append(h)
~~~

### 逐行读循环

| 源码行 | 这行做什么 |
|---:|---|
| 163 | 输入 Token ID 经过 Embedding 和缩放，得到初始隐藏状态 `h`。这行在两个循环之前，只执行一次。 |
| 164 | 为输入 Token 计算共享的 Value Embedding `ve`，后续传给各层 Attention。 |
| 165 | 如果调用者要求返回中间状态，就用初始 `h` 建一个列表。 |
| 166 | 外层按配置运行 3 轮，`loop` 依次为 0、1、2。 |
| 167 | 每轮都遍历同一个 `ModuleList`，`i` 从 0 到 17。`block` 是当前层模块。 |
| 168 | 计算从 0 开始的执行编号 `effective`。启用 KV Cache 时，Attention 用它标识本次层执行的缓存。 |
| 169 | 对当前 `h` 做 `norm_1`，计算 Attention，再把结果加回 `h` 并写回。 |
| 170 | 对 Attention 更新后的 `h` 做 `norm_2`，计算 MLP，再把结果加回并写回。这次写回得到当前层的最终输出。 |
| 171 | 检查是否需要收集中间状态。 |
| 172 | 需要时就把本层最终输出追加到列表，下一层仍直接接收当前 `h`。 |

### 第一轮最后一层怎样接到第二轮第一层

| 交接位置 | 当前模块与编号 | `h` 在这里是什么 |
|---|---|---|
| **第一轮最后一层跑完** | `loop = 0`、`i = 17`、`effective = 17`，模块是 `self.transformer.h[17]` | 第 170 行刚把第一轮第 18 层的最终输出写回 `h`。 |
| **第二轮第一层开始** | `loop = 1`、`i = 0`、`effective = 18`，模块回到 `self.transformer.h[0]` | 第 169 行右侧的 `block.norm_1(h)` 读到的就是上一行表格里的同一个隐藏状态。 |

**第一轮第 18 层在第 170 行写回 `h` → 外层进入第二轮 → 第二轮第 1 层在第 169 行读取这个 `h`。** 换轮时会重新取层模块，`h` 继续沿用。循环之间也没有再次执行第 163 行的 Embedding 初始化。

## 5. 输出头和 KV Cache 的编号

第 3 轮结束后，`ln_f` 对最终的隐藏状态做 RMSNorm。随后 `F.linear(h, self.transformer.wte.weight)` 用输入 Embedding 的权重算出每个位置对 49,152 个词表项的 Logits。这里可以用模块树对照 `wte` 的形状 `[49152, 1536]`。

源码还有一个 `effective = loop * 18 + i`。开启 KV Cache 时，Attention 用它索引本次执行的缓存。第一层模块 `h[0]` 跑三次，用到的缓存索引分别是 `0、18、36`。模块树仍然只有 18 个块。 [输出实现](https://huggingface.co/osunlp/PreNormLoop-0.6B/blob/e4ad59d0d03ab78f83d9f52b0c7d7267f372a151/modeling_baseline.py#L173) · [KV Cache 实现](https://huggingface.co/osunlp/PreNormLoop-0.6B/blob/e4ad59d0d03ab78f83d9f52b0c7d7267f372a151/modeling_baseline.py#L64)
