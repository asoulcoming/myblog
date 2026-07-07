---
title: VSCode 里 Python 项目一直爆红？大概率是解释器没选对
date: 2026-07-07 16:00:00
---

接手一个 Poetry 管理的 Python 项目，VSCode 打开后 `import` 全线爆红。折腾了一圈，最后发现原因只有一个：**VSCode 的 Python 解释器没指向项目的虚拟环境**。

## 问题表现

```python
# worker.py
from hatchet_sdk import Hatchet, Context  # 红色波浪线
from pureblueai.hatchet.middleware.s3 import load_input  # 红色波浪线
```

Pylance 报 `reportMissingImports`，看起来像是依赖没装。

## 排查过程（走的弯路）

- `pip install hatchet-sdk` → `command not found: pip`。以为是包管理器的问题，实际上项目用 Poetry，`pip` 要去 `poetry run pip` 或 `poetry shell` 里用。
- `poetry install` → 卡在私有包 `pureblueai-sdk`（Codeup 私有仓库，没配 SSH）。以为所有包都没装上，改了 `pyproject.toml` 去测试，后来发现根本不需要 —— 除 `pureblueai-sdk` 外其余 52 个公开包早已在 `.venv` 里。
- `poetry lock` 更新锁文件后装包状态不一致，最后 `poetry sync` 才修复。

## 真正的根因

从头到尾只有一个问题：**VSCode 没选对解释器**。

Poetry 把所有依赖装在项目的 `.venv/` 里，VSCode 默认可能用的是系统 Python（`/opt/homebrew/bin/python3`），自然找不到包。

## 解决方案

`Cmd+Shift+P` → `Python: Select Interpreter` → 选择：

```
./.venv/bin/python
```

选完后 Pylance 重新索引，爆红秒消。

## 验证依赖是否真的装好了

```bash
# 检查 Poetry 虚拟环境路径
poetry env info

# 直接测试导入
.venv/bin/python -c "import hatchet_sdk; print('OK')"
```

## 总结

| 我以为是 | 实际是 |
|---|---|
| pip 没装、依赖全丢了 | pip 在 Poetry 虚拟环境里，`poetry run pip` 就能用 |
| 私有包卡住了所有安装 | Poetry 只跳过装不上的包，已有的不会删 |
| 改了 pyproject.toml 才能装公开包 | 完全多余，包早就在 `.venv` 里了 |
| **VSCode 解释器没指向 .venv** | **这才是唯一的问题** |

VSCode + Pylance 的爆红，99% 的情况检查一下右下角的解释器就够了。
