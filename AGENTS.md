# 提交与发布流程

每次 Git 提交或发布前，都必须从 `source/` 目录重新构建，并清理、替换仓库根目录的构建产物：

```bash
cd source
npm run build
rm -rf ../assets && cp -R dist/assets ../assets && cp dist/index.html ../index.html
```

只有构建成功后才能执行清理和复制；任一步骤失败都应停止提交或发布并修复问题。清理范围仅限仓库根目录的 `assets/`。

同步完成后运行 `git diff --check`，将最新 `index.html` 和 `assets/` 的新增、修改、删除一并纳入提交。
