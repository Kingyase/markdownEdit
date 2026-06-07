# markdownEdit
一个轻量的markdown编辑器，对快速理解英文版的skill.md有很大帮助，支持渲染预览，鼠标悬停单词翻译中文，预览后谷歌浏览器翻译，预览后转html,word,pdf， ，离线全文翻译...
# MarkdownEdit v0.1.0

轻量 Markdown 编辑器，内置离线英译中。仅支持 64 位 Windows 10/11。

---

## 功能一览

| 功能 | 说明 |
|------|------|
| **文件管理** | 新建 / 打开(支持拖拽) / 保存 / 另存为，最近文件（8 项持久化） |
| **双栏实时编辑** | 左编辑右预览，150ms 防抖，滚动同步 |
| **翻译对照视图**（`Ctrl+1`） | 段落级神经网络翻译（Argos Translate），译文逐段渐进显示 |
| **渲染预览视图**（`Ctrl+2`） | GitHub 风格 HTML 渲染，代码高亮 |
| **单词悬停翻译** | 鼠标悬停英文单词 ≥500ms，弹出 ECDICT 离线词典释义 |
| **Chrome 翻译** | 预览右键 → "用 Chrome 打开并翻译"，自动触发 Chrome 翻译提示 |
| **导出** | HTML / PDF / Word |

---

## 截屏 / 工作流

```
┌─────────────────────┬──────────────────────────────────┐
│  编辑器 (左)         │  QStackedWidget (右)              │
│                     │  ┌────────────────────────────┐  │
│  # Hello            │  │  翻译对照 VIEW_TRANSLATION │  │
│                     │  │  你好                       │  │
│  This is a test.    │  │  这是一个测试。              │  │
│                     │  │  …(pending)                 │  │
│                     │  ├────────────────────────────┤  │
│                     │  │  渲染预览 VIEW_PREVIEW     │  │
│                     │  │  <h1>Hello</h1>             │  │
│                     │  │  <p>This is a test.</p>     │  │
│                     │  └────────────────────────────┘  │
└─────────────────────┴──────────────────────────────────┘
```

---

## 快速开始

### 环境要求

- Python 3.10+ (64-bit)
- Windows 10/11 64-bit

### 安装

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```

### 准备离线资源

**① 翻译模型**（约 67MB，自动下载）：

```bash
python scripts/fetch_argos_model.py
```

产物：`resources/models/translate-en_zh-1_9.argosmodel`

**② 英中词典**（约 50MB，需手动下载 CSV 后转换）：

从 [skywind3000/ECDICT](https://github.com/skywind3000/ECDICT) 下载 `ecdict.csv`，然后：

```bash
python scripts/build_ecdict.py path/to/ecdict.csv
```

产物：`resources/dict/ecdict.sqlite`

### 运行

```bash
python -m src.main
```

### 测试

```bash
pytest
```

---

## 项目结构

```
markdowEdit/
├── src/
│   ├── main.py                   # QApplication 入口
│   ├── ui/
│   │   ├── main_window.py       # QMainWindow + 菜单 + 初始化
│   │   ├── editor.py             # MarkdownEditor（防抖 contentSettled）│   │   ├── editor_tab.py         # Editorab（QSplitter + QStackedWidget）
│   │   ├── preview.py            # PreviewView（右键菜单 + Chome）
│   │   └── transation_view.py    # TransationView（纯译文）
│   ├── render/
│   │   ├── md_enderer.py         # render / render_bilingual / render_transation_only
│   │   └── paragraphs.py          # split_paragraphs（代码块保护）│   ├── transate/
│   │   ├── engine.py              # TransationEngine（Argos → CTransate2 批量推理）
│   │   ├── worker.py              # QThread + LRU + 队列批量出队
│   │   ├── dict_cache.py          # ECDICT SQLite 只读查询）│   │   └── hover.py               # HoverTransator（eventFiter + QToolTip）
│   └── utils/
│       ├── debounce.py            # QTimer 防抖
│       ├── paths.py               # 资源定位（支持 PyInstaler _MEIPASS）
│       └── chrome.py              # 定位 Chome + 强制 lang=en
├── resources/
│   ├── models/                    # Argos 翻译模型（67MB）│   └── dict/                       # ECDICT 词典（50MB）
├── scripts/
│   ├── fetch_argos_model.py       # 下载翻译模型
│   └── build_ecdict.py             # CSV → SQLite
├── build/
│   └── markdownedit.spec          # PyInstaller 配置
├── tests/                          # pytest 用例（15 个）
├── requirements.txt
└── README.md
```

---

## 架构设计

### 两层翻译

```
┌──────────────────────────────┐
│        编辑器输入 Markdown     │
└──────┬───────────────────────┘
       │
       ├── 段落级 → split_paragraphs() → TranslationWorker(QThread)
       │              └→ Argos Translate (CTranslate2)
       │              └→ 批量推理（多段落合并到一次 translate_batch）
       │              └→ 结果逐段 pyqtSignal 回主线程
       │
       └── 单词级 → HoverTranslator(eventFilter)
                      └→ ECDICT SQLite 查询（<30ms）
                      └→ QToolTip 展示释义
```

### 视图切换

左右分栏布局，右栏使用 `QStackedWidget`：

| 视图 | 快捷键 | 说明 |
|------|--------|------|
| `VIEW_TRANSLATION` | `Ctrl+1` | 左原文，右纯中文译文 |
| `VIEW_PREVIEW` | `Ctrl+2` | 左原文，右 HTML 渲染 |

### 关键设计取舍

1. **共享编辑器，右栏切视图**：避免双套编辑状态
2. **神经网络只做段落级，单词级走词典**：悬停 <30ms，不抢占翻译队列
3. **资源缺失优雅降级**：Argos 模型缺失 → 禁用翻译视图，默认渲染预览；ECDICT 缺失 → 禁用悬停，状态栏提示
4. **批量翻译优化**：所有段落的句子合并到一次 CTranslate2 `translate_batch` 调用，避免逐段串行推理
5. **Chrome 翻译**：改写 `<html lang="en">` + `--lang=zh-CN` 启动参数，让 Chrome 自动弹出翻译提示

---

## 快捷键

| 快捷键 | 功能 |
|--------|------|
| `Ctrl+N` | 新建文件 |
| `Ctrl+O` | 打开文件 |
| `Ctrl+S` | 保存 |
| `Ctrl+Shift+S` | 另存为 |
| `Ctrl+Q` | 退出 |
| `Ctrl+1` | 翻译对照视图 |
| `Ctrl+2` | 渲染预览视图 |

---

## 测试

```bash
pytest
```

15 个测试用例覆盖：MD 渲染、词典查询、段落切分、Chrome lang 注入。

---

## 打包

```bash
pyinstaller build/markdownedit.spec
```

产物：`dist/markdownedit/markdownedit.exe`（含模型与词典，合计约 1.5GB，因 argostranslate 间接依赖 stanza / spacy / onnxruntime / bitsandbytes 较大）。

---

## 技术栈

| 模块 | |
|------|------|
| 语言 / 运行时 | Python 3.12 (64-bit) |
| GUI | PyQt5 5.15 + PyQtWebEngine |
| Markdown 渲染 | `markdown` + `Pygments` |
| 段落翻译 | Argos Translate (CTranslate2 神经网络 en→zh) |
| 单词翻译 | ECDICT SQLite 词典（~77 万词条） |
| 打包 | PyInstaller (`--onedir`) |
