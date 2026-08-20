# 单人 Git 分支管理

## 分支命令

```bash
git branch                  # 查看本地分支
git branch feature-x        # 创建分支（不切换）
git checkout feature-x      # 切换到分支
git switch feature-x        # 新版切换命令，更直观
git checkout -b feature-x   # 创建并切换，一步到位
git branch -d feature-x     # 删除已合并的分支
```

## 合并分支

```bash
git checkout main
git merge feature-x
```

合并分两种结果：

- 快进合并（Fast-forward）：main 没有新提交，指针直接前移，历史一条直线。
- 三方合并：两边都有新提交，会生成一个新的合并提交。

想让历史留下"合并过"的痕迹，可用 `git merge --no-ff feature-x` 强制生成合并提交。

## 处理冲突

当 main 和 feature-x 改了**同一个文件的同一处**，Git 无法自动合并，会停下来标出冲突：

```
<<<<<<< HEAD
main 分支的改动
=======
feature-x 的改动
>>>>>>> feature-x
```

`<<<<<<<` 到 `=======` 之间是当前分支（main）的内容，`=======` 到 `>>>>>>>` 之间是被合并分支的内容。手动改成最终想要的结果、删掉标记，然后：

```bash
git add <冲突文件>
git commit
```

## Tips

- 新版推荐用 `git switch` / `git restore` 取代 `checkout`，语义更清晰。
- 解决冲突后一定要 `git add` 再 `git commit`，否则合并会一直停在冲突状态。
- 合并前先 `git diff main...feature-x` 看一眼差异，避免合错。
