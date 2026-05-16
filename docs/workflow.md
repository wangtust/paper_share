# 协作规范

## 每次开始前

先同步主分支，避免基于旧版本写内容：

```bash
git pull --rebase
```

## 新增论文笔记

复制模板：

```bash
cp docs/note-templates/paper-note.md docs/papers/rag/2026-05-16-example-paper.md
```

命名格式：

```text
docs/papers/<领域>/YYYY-MM-DD-paper-short-name.md
```

写完后更新 `docs/reading-list.md`，把论文加入总清单。

## 本地预览

```bash
mkdocs serve
```

浏览器访问：

```text
http://127.0.0.1:8000
```

## 提交

```bash
git status
git add docs/ mkdocs.yml README.md requirements.txt bib/
git commit -m "add rag paper note: example paper"
git push
```

## 减少冲突的约定

- 日常尽量只新增自己的论文笔记文件。
- 修改 `mkdocs.yml`、首页、总论文清单前，先确认没有其他人正在改。
- 大结构调整走 Pull Request。
- 不提交大体积 PDF，优先使用论文链接。
- 图片放在 `docs/assets/images/<论文短名>/` 下。
