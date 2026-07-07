---
title: Poetry 虚拟环境有 pip 却没项目包？poetry sync 修好了
date: 2026-07-07 16:00:00
---

接手一个 Poetry 管理的 Python 项目，VSCode 打开后 `import` 全线爆红。排查过程走了一堆弯路，最后发现根因简单得离谱。

## 问题表现

```python
from hatchet_sdk import Hatchet, Context  # reportMissingImports
from openpyxl import Workbook             # reportMissingImports
```

Pylance 报所有第三方包都找不到。

## 排查过程（弯路部分）

**第一反应**：`pip install hatchet-sdk` → `command not found: pip`。项目其实用 Poetry，pip 在虚拟环境里。

**第二反应**：`poetry install`，结果卡在私有包 `pureblueai-sdk`（Codeup 私有仓库，本地没配 SSH），报 `HangupException`。以为这导致所有包都没装，于是改 `pyproject.toml` 注释掉私有包、跑 `poetry lock`、再 `poetry install` —— 显示 "No dependencies to install or update"。

**这里有迷惑性**：`poetry run pip show hatchet-sdk` 显示包已安装在 `.venv/lib/python3.12/site-packages`，看起来一切正常。

## 真正的问题

直接检查 `.venv` 的 `site-packages`：

```bash
$ ls .venv/lib/python3.12/site-packages/
pip
pip-26.1.2.dist-info
```

**只有 pip，没有任何项目依赖。** `.venv` 是个空壳，`poetry run pip show` 的输出有误导性。Poetry 锁文件认为包已装，但实际文件系统里不存在。

## 解决

```bash
poetry sync
```

53 个包重新安装到 `.venv`，然后 VSCode `Cmd+Shift+P` → `Python: Select Interpreter` 选 `.venv/bin/python`，爆红消失。

## 总结

| 做了 | 有用吗 |
|---|---|
| `poetry install`（卡在私有包） | ❌ 公开包也没装上 |
| 改 `pyproject.toml` 注释私有包 | ❌ 多余操作 |
| `poetry lock` | ❌ 反而把锁文件搞坏了 |
| `poetry install --sync` / `poetry sync` | ✅ **这才修好了** |
| VSCode 选 `.venv/bin/python` | ✅ 解释器对了才行 |

`.venv/site-packages` 里只有 pip 就是坏掉的信号。以后遇到类似情况直接 `poetry sync`，省掉前面所有弯路。
