---
title: Hello World
---
## 发布博客

> 源码存放于source分支下，编译后的文件存放于master分支下。

```bash
# 在source分支下
hexo new 'hello world'

vim source/_posts/hello-world.md

# 编辑完
git add -A
git commit -m 'comment'
git push

hexo deploy
```

## 修改theme


```bash
# 修改 `theme/${theme}/_config.yml`
hexo g
hexo deploy
```
