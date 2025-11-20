---
title: git 教程
published: 2025-01-19
description: git 从入门到精通 + 任务书
# image: ./images/cover1.png
tags: [git, 教程]
category: git
draft: false
---

# Git
## 安装与配置
```sh
# Linux 安装
sudo apt install -y vim git

# 全局配置用户信息
git config --global user.name "aLinChe"
git config --global user.email "1129332011@qq.com"
git config --global core.editor vim
git config --global color.ui true
```

## 创建仓库与基础操作
```sh
git init [project_name]
/// or
git clone https://github.com/DragonOS-Community/DragonOS

git status
git add .
git commit -m "feat(add-func): feat template-function-add to ..."

git fetch
git pull --all
git pull --rebase

git push
git push -f
```

## 分支操作
```sh
# 创建新分支并切换到该分支
git checkout -b [branch-name]
# 切换到对应分支
git checkout [branch-name]

git branch -vv
git branch -a

# 删除本地分支
git branch -d [branch-name] # 安全删除(已合并)
git branch -D [branch-name] # 强制删除(未合并)
```

## 合并与变基策略
```sh
git checkout master
git merge feat/add_branch    # 合并指定分支(feat/add_branch)到当前分支(master)
git push                     # 推送合并结果
git checkout feat/add_branch # 切换回原开发分支
/// or
git checkout feat/add_branch
git rebase master                    # 将当前分支变基到master
git branch -f master feat/add_branch # 移动master指针 or 
git checkout master                  # 切换到master
git push                             # 推送更新
git checkout feat/add_branch         # 切换回原开发分支
```

## 提交历史修改
```sh
# 撤销与重置
git reset HEAD~n         # 撤销最近 n 次提交
git reset --hard HEAD~n  # 丢弃最近 n 次提交
git reset [hashed]
git reset --keep [hashed]

git revert [hashed]      # 创建新提交来撤销更改

git push origin [branch-name]
git branch -d [branch-name]
git branch -D [branch-name]
git push origin -d [branch-name]

git rebase -i HEAD~n
squash
# pick   - 保留提交
# reword - 修改提交信息
# edit   - 修改提交内容
# squash - 合并到前一个提交
# drop   - 丢弃提交
```

## 储藏与拣选操作
```sh
# 临时储藏更改
git stash
git stash save "描述信息"
git stash list
git stash pop
git stash apply stash@{n}
git stash drop stash@{n}
```

```sh
# 选择性应用提交
git cherry-pick [commit-hash]

git cherry-pick [start-hash]^..[end-hash]

git cherry-pick [hash1] [hash2] [hash3]
```

## 远程仓库管理
```sh
git remote -v
git remote add [shortname] [url]
git remote set-url --push upstream no_push # 删掉 upstream_url 防止意外 push -f
git branch -u [remote]/[branch] [当前分支]
git branch --set-upstream-to=upstream/master origin/master
git branch -vv -a

git remote show origin
git push -u origin feat/add # 首次推送并建立关联 ## 等效于 git branch -u origin/feat/add 
git branch --unset-upstream # 取消错误的追踪

# 删除远程分支
git push origin -d [branch-name]

git fetch upstream
git checkout master
git rebase upstream/master

git archive

```

## 子模块操作
```sh
git submodule add https://github.com/user/submodule.git # 添加子模块

git clone --recurse-submodules https://github.com/user/repo.git # 克隆包含子模块的仓库

git submodule update --init --recursive # 更新子模块
```

## 修改历史二进制文件以减小clone大小
```sh
pip install git-filter-repo # ~/.local/bin/git-filter-repo
git filter-repo --file-info-callback '
   if filename == b"vmlinuz":
      with open("bin/vmlinuz/vmlinuz", "rb") as f:
            new_content = f.read()
      new_blob_id = value.insert_file_with_contents(new_content)
      return (filename, mode, new_blob_id)
   else:
      return (filename, mode, blob_id)
   '
```

## 最佳实践总结
1. 提交规范​​：使用语义化提交信息（feat:, fix:, docs:, 等）
2. ​分支策略​​：
   - main/master分支用于生产环境
   - develop分支作为集成分支
   - 功能分支（feat/*）用于功能开发
3. ​定期同步​​：频繁执行fetch和pull --rebase避免冲突
4. ​保持历史整洁​​：本地分支使用(rebase)变基，公共分支使用(merge)合并
5. ​​原子化 commit 提交​​：适当使用 squash 
6. PR 前必做 rebase 整理



# Git 实战任务书
## Task 1：原子化提交练习
情景​​：你正在开发用户注册功能，已完成三个部分：
1. 创建用户模型（models/user.py）
2. 实现注册API（api/auth.py）
3. 添加注册页面（UI/register.html）

要求​​：
1. 为每个部分创建独立的提交
2. 提交信息遵循语义化规范（feat/fix/docs等）
3. 确保每个提交都是可独立构建的

## Task 2：交互式变基合并
​​情景​​：在Task 1的后续开发中，你为注册功能`依次`创建了5个提交：
1. feat(user): 基础模型
2. fix(user): 修复密码字段
3. feat(auth): API框架
4. fix(user): 修复密码字段格式
5. feat(ui): 注册页面UI
6. fix(auth): 解决验证问题

要求​​：
1. 将5个提交合并为3个有意义的提交：
2. 用户模型相关（1+2+4）
3. API相关（3+5）
4. UI相关（5）
5. 重写提交信息为专业格式

## Task 3：热修复集成
​​情景​​：生产环境发现紧急bug（急急急急急）：
- main分支版本：v1.2.0
- 当前开发分支：feat/user-profile
- 需要同时修复：
   1. 修复用户模型结构（models/user.py）
   2. 添加错误提示信息（UI/errors.py）

​​要求​​：
1. 保存你现在开发到一半的文件
2. 为热修复创建独立分支
3. 提交修复到main分支
4. 将修复同步到原来的开发分支

## Task 4：团队协作
要求：
1. 学会使用`合适的工具`(比如 VSCode 图形化界面、相关的Github插件)：解决冲突，view PR，comment等等