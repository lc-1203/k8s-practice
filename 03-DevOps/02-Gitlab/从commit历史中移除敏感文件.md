# 删除commit历史中的敏感信息

当历史文件中存在敏感信息时（如AccessKey，smtp码），即使修改掉敏感信息/删除文件，这些敏感信息仍旧存在于历史commit中，最好的方法是马上删掉AccessKey，smtp码。这里也提供一些方法删除commit里的敏感信息（慎用），但是Github对历史commit有缓存，导致即使项目里面原commit id已经不存在，但还是能通过commit id访问到旧代码，如下图所示。

![image-20220313140121984](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203131401026.png)

## 方式一：filter-branch

### 备份最新的文件

### 执行git filter-branch

其中，PATH-TO-YOUR-FILE-WITH-SENSITIVE-DATA 是你需要删除的敏感信息文件名，执行此操作将会删除该文件，因此请先备份

```shell
git filter-branch --force --index-filter \
'git rm --cached --ignore-unmatch ./03-DevOps/02-Gitlab/docker-compose部署Gitlab.md' \
--prune-empty --tag-name-filter cat -- --all
```

![image-20220313135805715](https://lc-tc.oss-cn-shenzhen.aliyuncs.com/lc-images/202203131358788.png)

### 将备份文件导回并add

```shell
# 将之前备份的文件copy回来
git add ./03-DevOps/02-Gitlab/docker-compose部署Gitlab.md
```

### 覆盖commit（作用于所有分支）

```shell
git push origin --force --all
```

## 方式二：重新初始化

```shell
# 创建了一个新的非父分支
git checkout --orphan latest_branch

# 提交所有文件暂存区
git add -A

# commit
git commit -am "Reinitialize";

# 删除原分支
git branch -D master;

# 将当前新分支重命名为原分支
git branch -m master;

# 推送
git push -f origin master;
```

