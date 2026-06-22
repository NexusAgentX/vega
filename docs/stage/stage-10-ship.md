# Stage 10 — 可分发的 Vega

## 是什么 Vega

打包为单一二进制文件，用户下载即用 —— 不装 Python、不配 uv、不拉依赖。配合完整 E2E 测试和发布流程，Vega 从「git clone 才能跑」升级为「curl 下载、解压、运行」的产品。

## 新增能力

- PyInstaller 单二进制打包（macOS arm64 / x86_64、Linux x86_64、Windows x86_64）
- GitHub Release 自动化构建（CI/CD）
- 完整 E2E 测试套件（从零开始覆盖所有用户路径）
- 安装文档与快速上手

## 前置依赖

[stage 9](./stage-9-scale.md) 必须完成。本 stage 是收尾。

## 交付清单

### PyInstaller 配置

`vega.spec`（PyInstaller 规格文件）：

```python
# 关键点
# - --onefile 单文件输出
# - --name vega
# - 把 LLM prompts、Alembic migrations、FTS5 trigger SQL 打进包
# - 隐藏 litellm / sqlmodel / fastapi 的动态导入
# - platform-specific hooks（litellm 子依赖、fastapi/pydantic 等）
```

特殊处理：
- Alembic migrations 目录要 copy 进包（`datas=[('vega/migrations', 'migrations')]`）
- litellm 各 provider adapter 是动态导入，需 `hiddenimports`
- SQLite 外部运行时不依赖（Python 自带）
- PyInstaller 找不到的 C 扩展（如 pdfplumber 的 pdfminer）单独处理

### CLI 增强

```
vega --version                      # 打印版本号 + 构建 commit + 构建时间
vega --help                         # 完整命令树
vega doctor                         # 环境自检：数据库可写？磁盘空间？网络？LLM 配置？
```

`vega doctor` 是新命令，首次运行或报错时引导用户排查环境问题。

### GitHub Actions

`.github/workflows/release.yml`：

- trigger：push tag `v*.*.*`
- matrix：macos-latest (arm64 + x86_64 via Rosetta)、ubuntu-latest、windows-latest
- 步骤：
  1. checkout
  2. setup-uv
  3. `uv sync --frozen`
  4. `uv run pytest` 全测试
  5. `uv run pyinstaller vega.spec`
  6. 压缩 `vega` 二进制为 `vega-<version>-<os>-<arch>.tar.gz` / `.zip`
  7. 生成 SHA256 校验
  8. 上传到 GitHub Release

### E2E 测试

`tests/e2e/` —— 完整用户路径测试，不是单元测试：

```
tests/e2e/
├── test_fresh_user.py              # 全新用户从零开始的完整路径
├── test_kb_growth.py               # 持续 ingest 100 篇文档
├── test_search_journey.py          # 搜索 → 摘要 → 原文
├── test_link_journey.py            # 建立关系 → 图遍历
├── test_inbox_journey.py           # 审核 100 篇文档
├── test_chat_journey.py            # 多轮对话 + 热更新
├── test_okf_roundtrip.py           # 导出 → 清库 → 导入 → 数据无损
└── test_s3_journey.py              # 切换 S3 后端 + 数据迁移
```

每个 e2e 测试用一个临时 `~/.vega/`（`tmp_path` fixture），完全隔离。

### 安装文档

`docs/install.md`：

```markdown
# Vega 安装

## 方式一：预编译二进制（推荐）

# macOS (Apple Silicon)
curl -L https://github.com/NexusAgentX/vega/releases/latest/download/vega-darwin-arm64.tar.gz | tar xz
sudo mv vega /usr/local/bin/

# Linux
curl -L https://github.com/NexusAgentX/vega/releases/latest/download/vega-linux-x86_64.tar.gz | tar xz
sudo mv vega /usr/local/bin/

# 验证
vega --version

## 方式二：从源码（开发者）

git clone https://github.com/NexusAgentX/vega
cd vega
uv sync
uv run vega --version

## 首次使用

vega doctor        # 环境自检
vega ingest ./hello.md --yolo
vega list
```

### 文件清单

- `vega.spec` — PyInstaller 配置
- `vega/__init__.py` — 加 `__version__`
- `vega/cli/commands/doctor.py` — 自检命令
- `vega/cli/commands/version.py`
- `.github/workflows/release.yml`
- `.github/workflows/ci.yml` — PR/push 跑测试
- `tests/e2e/` — 整个新建
- `docs/install.md`
- `scripts/build.sh` — 本地构建辅助
- `scripts/sha256.sh` — 校验和生成

## 验收标准

### 单二进制

```bash
# 1. 本地构建
uv run pyinstaller vega.spec
ls dist/vega                        # 期望: 单个可执行文件

# 2. 无 Python 环境运行（docker 验证）
docker run --rm -v $(pwd)/dist:/dist alpine /dist/vega --version
# 期望: 正常打印版本（alpine 无 Python）

# 3. 首次运行
./dist/vega doctor
# 期望: 全绿，或明确告知缺哪些（如 LLM 未配置）

# 4. 完整功能链
./dist/vega ingest ./readme.md --yolo
./dist/vega list
./dist/vega search "vega"
./dist/vega export --okf /tmp/bundle/
./dist/vega import --okf /tmp/bundle/ --yolo
# 期望: 全部正常
```

### GitHub Release

```bash
# 5. 触发 release
git tag v0.1.0
git push origin v0.1.0
# 等 CI 跑完，检查 release 页面：
# - 5 个二进制（darwin-arm64, darwin-x86_64, linux-x86_64, windows-x86_64.exe, ...）
# - SHA256SUMS.txt
# - 自动生成的 changelog
```

### E2E 测试

```bash
# 6. 完整 e2e
pytest tests/e2e/ -v --tb=short
# 期望: 全过

# 7. 全量测试
pytest tests/ -v
# 期望: 全过（stage 0-10 所有测试）
```

### 跨平台

```bash
# 8. macOS Apple Silicon
# 在 M1 上构建，运行 vega，OK

# 9. Linux x86_64
# CI 构建，docker 运行，OK

# 10. Windows
# CI 构建，本机运行，OK（注意路径分隔符、~/.vega 解析）
```

### 单元 + E2E 测试覆盖

- E2E `test_fresh_user`：模拟全新用户从 install 到 chat 的完整路径
- E2E `test_okf_roundtrip`：stage 8 的往返测试 plus 100 篇文档规模
- `vega doctor` 检测：数据库可写、磁盘空间 > 100MB、网络连通、LLM 配置（可选）

## 不做的事

- ❌ Homebrew / apt / chocolatey 包管理器上架 — 未来按需
- ❌ Web UI（Phase 2） — 单独项目
- ❌ 桌面 GUI（Tauri / Electron） — 不在路线图
- ❌ 移动端 — 不在路线图
- ❌ 多架构 Docker 镜像 — 未来按需
- ❌ 签名 / 公证（macOS notarization、Windows code signing） — 未来按需（用户自己信任）
- ❌ 自动更新（in-place upgrade） — 不在路线图（重新下载即升级）

## 完成判定

任意用户能在 60 秒内完成：下载 → 解压 → `vega --version` → `vega ingest` → `vega search` → 得到结果。**Vega 从 git 仓库升级为可分发的产品**。

每个 stage 都是一个完整可用的 Vega，stage 10 是这个 MVP pyramid 的顶点 —— 一个成熟、可分发、可信赖的本地 AI 知识库。
