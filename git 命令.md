## git 代理配置

### 设置代理服务器和端口

```bash
git config --global http.proxy 'http://127.0.0.1:7890'
git config --global http.proxy 'https://127.0.0.1:7890'
```

### 查看 Git 配置

```bash
git config --list
```

## 远端仓库强制覆盖本地

### 从远端仓库下载最新文件

```bash
git fetch --all
```

### 覆盖本地文件

```bash
git reset --hard origin/main
```

如果本地不在 main 分支，则执行：

```bash
git reset --hard origin/<branch_name>
```

其中 `branch_name` 为分支名称

### 本地仓库强制推送远端

```bash
git push --force origin main
```

### 创建空分支

```bash
git checkout --orphan <branch_name>
git rm -rf .
```

### 添加远程仓库

```bash
git remote add <origin_name> https://github.com/user/repo.git
```

### 将当前分支与远程分支关联

```bash
git branch --set-upstream-to=<origin_name>/<branch_name> <local_branch_name>
```
