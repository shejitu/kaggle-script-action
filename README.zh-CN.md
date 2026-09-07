# kaggle-script-action 使用说明（中文版）

> 这是 `KevKibe/kaggle-script-action` 的中文补充说明，原版英文 README 见 [README.md](README.md)。
> 本说明由 shejitu 整理，配合自己的调度仓库 [shejitu/kaggle-hf-lab](https://github.com/shejitu/kaggle-hf-lab) 使用。

## 一句话总结

**把你的 GitHub 仓库代码送到 Kaggle 的免费 GPU/TPU 上运行，日志自动回传 GitHub Actions**——不用打开 Kaggle 网页，`git push` 即触发。

## 它内部到底做了什么（核心机制）

这个 Action 是一个 composite action，运行时分四步：

1. **生成 Notebook**：把你的仓库地址、分支、自定义脚本拼进一个临时 ipynb，内容固定为三个 cell：
   ```python
   !git clone --branch <你的分支> <你的仓库>.git     # 在 Kaggle 服务器上克隆你的仓库
   !cd /kaggle/working/<仓库名> && pip install -r requirements.txt   # 装依赖
   !cd /kaggle/working/<仓库名> && <你的 custom_script>              # 跑你的脚本
   ```
2. **推送 kernel**：用 Kaggle CLI（`kaggle kernels push`）把 Notebook 推到你的 Kaggle 账号下，kernel id 固定为 `<username>/<标题转小写空格变横线>`（如 `xwdfyx/hf-model-gpu-test`）。每次触发都是同一 kernel 的**新版本**，历史版本在 Kaggle 上可回看。
3. **轮询状态**：每隔 `sleep_time` 秒查一次 kernel 状态，直到 complete / error / cancel。
4. **拉取日志**：`kaggle kernels output` 把运行日志拉回来，尾部 50 行逐条解析显示在 GitHub Actions 日志里，遇到 Error/Traceback 会把 job 标记为失败。

## 必备前提

| 项目 | 说明 |
|---|---|
| Kaggle 账号 | 在 kaggle.com → Settings → API → Create New Token 生成 `kaggle.json` |
| GitHub Secrets | 把 kaggle.json 里的 username / key 分别存为 `KAGGLE_USERNAME`、`KAGGLE_KEY` |
| 仓库必须**公开** | 因为 `git clone` 是在 Kaggle 服务器上执行的，私有仓库它会拉不到 |
| requirements.txt | 仓库根目录必须有（哪怕为空注释），否则 pip 安装 cell 会失败 |
| Kaggle 手机验证 | 用 GPU 前需在 Kaggle 账号设置里完成手机验证 |
| GPU 配额 | Kaggle 免费用户每周约 30 小时 GPU（30T/20T 两种规格） |

## 完整 workflow 示例（GPU + 联网 + 自定义脚本）

```yaml
name: Run on Kaggle GPU
on:
  workflow_dispatch:
  push:
    branches: [main]

jobs:
  run-on-kaggle-gpu:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: KevKibe/kaggle-script-action@v1.0.5
        with:
          username: ${{ secrets.KAGGLE_USERNAME }}
          key: ${{ secrets.KAGGLE_KEY }}
          title: "My GPU Job"                # 决定 kernel slug
          custom_script: |
            python my_test.py --model 'distilbert-base-uncased-finetuned-sst-2-english'
          enable_gpu: true
          enable_tpu: false
          enable_internet: true
          sleep_time: 30
```

## 全部参数速查

| 参数 | 必填 | 默认 | 说明 |
|---|---|---|---|
| `username` | ✅ | - | Kaggle 用户名 |
| `key` | ✅ | - | Kaggle API token |
| `title` | ✅ | - | kernel 标题（决定 slug） |
| `custom_script` | ✅ | `print('Success')` | 要执行的命令，在 `/kaggle/working/<仓库名>` 下运行 |
| `working_subdir` | 可选 | `""` | 仓库内子目录作为工作目录 |
| `enable_gpu` | 可选 | `false` | 启用 GPU |
| `enable_tpu` | 可选 | `false` | 启用 TPU |
| `enable_internet` | 可选 | `true` | 启用外网（HF 在线拉模型必须开） |
| `dataset_sources` | 可选 | - | 挂载 Kaggle 数据集，格式 `{username}/{slug}` |
| `kernel_sources` | 可选 | - | 挂载其他 kernel 的输出 |
| `competition_sources` | 可选 | - | 挂载比赛数据 |
| `sleep_time` | 可选 | `15` | 状态轮询间隔（秒） |

## 踩坑提醒（实测）

1. **custom_script 里别用双引号**：脚本被原样拼进 ipynb 的 JSON，双引号会破坏 JSON 结构导致推送失败，一律用单引号。
2. **kernel slug 固定**：同一 title 每次触发都覆盖同一 kernel 的版本；想并行跑不同任务，用不同 title。
3. **模型加载路径**：开了 internet 时可直接 `pipeline(model='HF模型ID')` 在线拉；离线挂载则加 `dataset_sources`，模型在 `/kaggle/input/` 下。
4. **大模型超时**：Kaggle 单 session 有时长上限，超大模型训练要评估时间；GitHub 侧 job 也有 6 小时上限。
5. **日志抓取**：只有 kernel 跑完后才能拉到完整日志；脚本报错时 kernel 状态为 error，Action 直接失败，错误细节在 Actions 日志里。
6. **配额**：GPU 是稀缺资源，`workflow_dispatch` 手动触发比 push 自动触发更省（不会被无关 commit 白白烧配额）。

## 和 Hugging Face 的联动姿势

- **在线模式**（推荐起步）：`enable_internet: true`，脚本里 `transformers` 直接从 HF Hub 拉模型。
- **离线模式**：在 HF 模型页点 "Use this model → Kaggle"，或用 Kaggle 的 Import from HuggingFace，把模型存成私有 Dataset/Model；然后在 workflow 的 `dataset_sources` 里挂载，脚本从 `/kaggle/input/` 加载。
- **端到端例子**：见 [shejitu/kaggle-hf-lab](https://github.com/shejitu/kaggle-hf-lab) 的 `hf_model_test.py` + workflow 配置。
