# 运行命令

- 项目：pyannote-audio（说话人日志 toolkit）
$12026-09-09
- 运行方式：直接运行（`uv` + 本地 CLI / Python API）
- 硬件评估：**满足**
  - 官方：pretrained pipeline 默认可 CPU；GPU 可选（`pipeline.to(cuda)`）
  - 官方速度基准写的是 H100 80GB，那是性能对照，不是最低门槛
  - 本机：RTX 4080 16GB；评估时已占用约 6.2GB，剩余约 10GB；说话人日志模型通常只需数 GB
  - 内存约 32GB、E: 盘空间充足

## 环境准备

```powershell
# 官方 PyPI / HF 超时，改用国内镜像
$env:UV_INDEX_URL = 'https://pypi.tuna.tsinghua.edu.cn/simple'
$env:HF_ENDPOINT = 'https://hf-mirror.com'
$env:HF_HOME = 'E:\huggingface_cache'
$env:PYANNOTE_METRICS_ENABLED = '0'

# ffmpeg（torchcodec 需要）
$env:Path = "E:\Programs\ffmpeg-master-latest-win64-gpl\bin;$env:Path"

cd E:\AI\local-voice\pyannote-audio
uv sync --extra cli

```powershell
# 锁文件里的 torch 是 CPU 轮；装完依赖后强制换成 CUDA 12.8
uv pip install --python .\.venv\Scripts\python.exe --reinstall `
  https://download.pytorch.org/whl/cu128/torch-2.10.0%2Bcu128-cp310-cp310-win_amd64.whl `
  https://download.pytorch.org/whl/cu128/torchaudio-2.10.0%2Bcu128-cp310-cp310-win_amd64.whl
```

启动脚本使用 `uv run --no-sync`，避免再次把 torch 同步回 CPU。
```

预训练 pipeline（`pyannote/speaker-diarization-community-1`）需：

1. 在 https://hf.co/pyannote/speaker-diarization-community-1 接受用户协议
2. 创建 token：https://hf.co/settings/tokens
3. 本机登录或设环境变量：

```powershell
hf auth login
# 或
$env:HF_TOKEN = 'hf_xxx'
```

## 启动

- 推荐：`pwsh -NoProfile -File .\start.ps1`（菜单；默认 [1] `smoke`，不长时间占卡）
- 服务 Id：`smoke`、`diarize`（需 `HF_TOKEN`）
- 可选：`-AudioPath` / `-Device auto|cuda|cpu|mps`
- 自动化：`pwsh -NoProfile -File .\start.ps1 -Service diarize`
- 等价手动命令（冒烟）：

```powershell
$env:Path = "E:\Programs\ffmpeg-master-latest-win64-gpl\bin;$env:Path"
uv run --no-sync python -c "import torch; from pyannote.audio import Pipeline; print('torch', torch.__version__, 'cuda', torch.cuda.is_available())"
```

- 对音频跑 community-1（需 HF_TOKEN）：

```powershell
uv run --no-sync pyannote-audio apply pyannote/speaker-diarization-community-1 .\src\pyannote\audio\sample\sample.wav --device cuda $env:HF_TOKEN
```

## 验证

- 冒烟：`start.ps1` 打印 `torch 2.10.0+cu128 cuda True；torchcodec 能解码 sample.wav`，且能 `import pyannote.audio`
- 完整：对 `sample.wav` 输出若干 `SPEAKER_XX` 时间段（RTTM），无 CUDA OOM
- CLI 退出码为 0

## 备注

- 镜像：PyPI → 清华；Hugging Face → `https://hf-mirror.com`（官方 huggingface.co / pypi.org 探测超时）
- ffmpeg：`E:\Programs\ffmpeg-master-latest-win64-gpl\bin`
- 本仓库是库 + CLI，无固定 Web 端口；`start.ps1` 的 `-Port` 仅保留模板兼容，不影响运行
- 当前 GPU 已有其它 Python 进程占显存；若 OOM，先结束占卡进程或改 `--device cpu`
- 勿把 `HF_TOKEN` 写进仓库文件