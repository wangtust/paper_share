# Paper Share

这是一个三人共同维护的论文阅读分享仓库，用来沉淀不同研究方向的论文列表、阅读笔记和讨论记录。仓库使用 MkDocs Material 构建成静态网站，日常内容主要写在 `docs/` 目录下的 Markdown 文件中。

## 目录结构

```text
paper_share/
├── README.md
├── mkdocs.yml
├── requirements.txt
├── docs/
│   ├── index.md
│   ├── reading-list.md
│   ├── workflow.md
│   ├── note-templates/
│   │   └── paper-note.md
│   ├── topics/
│   │   ├── llm.md
│   │   ├── rag.md
│   │   ├── agent.md
│   │   ├── multimodal.md
│   │   ├── cv.md
│   │   ├── nlp.md
│   │   └── systems.md
│   ├── papers/
│   │   ├── llm/
│   │   ├── rag/
│   │   ├── agent/
│   │   ├── multimodal/
│   │   ├── cv/
│   │   ├── nlp/
│   │   └── systems/
│   └── assets/
│       └── images/
└── bib/
    └── references.bib
```

## 内容组织原则

- `docs/papers/` 按领域存放具体论文笔记，每篇论文一个 Markdown 文件。
- `docs/topics/` 存放领域综述、阅读路线和该方向的关键问题。
- `docs/reading-list.md` 维护总论文清单，便于快速查看大家最近读了什么。
- `docs/note-templates/paper-note.md` 是论文笔记模板，新增笔记时优先复制它。
- `docs/assets/images/` 存放论文截图、框架图和实验结果图。
- `bib/references.bib` 可选维护 BibTeX 引用信息。

文件命名建议：

```text
docs/papers/<topic>/YYYY-MM-DD-paper-short-name.md
```

示例：

```text
docs/papers/rag/2026-05-16-self-rag.md
docs/papers/llm/2026-05-16-attention-is-all-you-need.md
```

## 本地预览

第一次使用：

```bash
git clone git@github.com:wangtust/paper_share.git
cd paper_share
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

然后打开：

```text
http://127.0.0.1:8000
```

之后每次进入仓库：

```bash
cd paper_share
source .venv/bin/activate
mkdocs serve
```

## 日常协作流程

每次开始写之前先同步：

```bash
git pull --rebase
```

新增一篇论文笔记：

```bash
cp docs/note-templates/paper-note.md docs/papers/rag/2026-05-16-example-paper.md
```

写完后本地检查：

```bash
mkdocs serve
```

提交并推送：

```bash
git status
git add docs/ mkdocs.yml README.md requirements.txt bib/
git commit -m "add rag paper note: example paper"
git push
```

## 协作约定

- 三个人尽量分别新增自己的论文笔记文件，减少同时修改同一个文件。
- `mkdocs.yml`、`docs/index.md`、`docs/reading-list.md` 属于公共入口文件，修改前建议先在群里说一声。
- 新增领域时，同时补充 `docs/topics/<topic>.md` 和 `docs/papers/<topic>/index.md`，再更新 `mkdocs.yml` 导航。
- 不建议直接提交大量 PDF。优先记录 arXiv、OpenReview、ACL Anthology、DOI、GitHub 等链接。
- 图片按论文单独建目录，例如 `docs/assets/images/2026-05-16-self-rag/framework.png`。
- 重要结构调整建议开 Pull Request；普通新增笔记可以直接 push。

## 发布网站

如果使用 GitHub Pages 发布，可以在本地执行：

```bash
mkdocs gh-deploy
```

也可以后续配置 GitHub Actions，在 `main` 分支更新后自动部署。
