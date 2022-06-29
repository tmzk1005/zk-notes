---
title: "一个自动实现git快进合并的shell脚本"
date: 2020-12-20T11:27:39+08:00
draft: false
tags: ["git", "shell"]
---

用法: `mg <src_branch> <dst_branch>`

```bash
#!/usr/bin/env bash

set -e

# 项目路径,根据需要自己修改为合适的值
PROJECT_DIR=/home/zk/sangfor/code/ngsoc

# 远程仓库名称,一般默认为origin,可以自定义
REPO_NAME=origin

# 待合并的远程分支名称,由参数1指定
SRC_BRANCH=$1

# 合并请求的目标分支,由参数2指定
TARGET_BRANCH=$2

if [[ $# != 2 ]]; then
    echo "Usage: mg <src_branch> <dst_branch>"
    exit 1
fi

cd "$PROJECT_DIR"

x=$(git status -s | wc -l)
if [[ $x -gt 0 ]]; then
    echo "\033[5;31;40m[ERROR] \033[0m请先保持git工作空间为一个干净的工作空间再执行此命令, 可以先执行命令'git stash'暂存当前工作"
    exit 1
fi

git fetch "$REPO_NAME" --prune


x=$(git --no-pager branch -r --list "$REPO_NAME/$SRC_BRANCH" | wc -l)
if [[ $x -lt 1 ]]; then
    echo -e "\033[5;31;40m[ERROR] \033[0m远程仓库$REPO_NAME不存在分支$SRC_BRANCH, 请检查分支名是否正确"
    exit 1
fi

x=$(git --no-pager branch -r --list "$REPO_NAME/$TARGET_BRANCH" | wc -l)
if [[ $x -lt 1 ]]; then
    echo -e "\033[5;31;40m[ERROR] \033[0m远程仓库$REPO_NAME不存在分支$TARGET_BRANCH, 请检查分支名是否正确"
    exit 1
fi

MG_BASE=$(git merge-base "$REPO_NAME/$SRC_BRANCH" "$REPO_NAME/$TARGET_BRANCH")
TARGET_HASH=$(git merge-base "$REPO_NAME/$TARGET_BRANCH" "$REPO_NAME/$TARGET_BRANCH")

echo $MG_BASE
echo $TARGET_HASH

if [[ "$MG_BASE" != "$TARGET_HASH" ]]; then
    # 不能直接快进合并,需要先rebase,先判断即将进行的rebase是否会发生冲突
    x=$(git format-patch "$MG_BASE".."$REPO_NAME/$TARGET_BRANCH" --stdout | git apply --3way --check 2>&1 | wc -l)
    if [[ $x -gt 0 ]]; then
        echo -e "\033[5;31;40m[ERROR] \033[0m不能快进合并,需要rebase,但是rebase操作有不能自动解决的冲突, 请通知合并请求提交者先本地rebase其代码到最新目标分支,并解决冲突后重新>提交代码"
        exit 1
    fi
fi

function __git_prompt_git() {
  GIT_OPTIONAL_LOCKS=0 command git "$@"
}

function git_current_branch() {
  local ref
  ref=$(__git_prompt_git symbolic-ref --quiet HEAD 2> /dev/null)
  local ret=$?
  if [[ $ret != 0 ]]; then
    [[ $ret == 128 ]] && return  # no git repo.
    ref=$(__git_prompt_git rev-parse --short HEAD 2> /dev/null) || return
  fi
  echo ${ref#refs/heads/}
}

CUR_BRANCH=$(git_current_branch)

TMP_B='tttttttttt'
# rebase远程SRC_BRANCHE
# 新建临时分支跟踪远程SRC_BRANCH,用于rebase远程SRC分支到最新的DST分支之上
git checkout -b "$TMP_B" "$REPO_NAME/$SRC_BRANCH"
git rebase "$REPO_NAME/$TARGET_BRANCH"
git push "$REPO_NAME" "$TMP_B":"$SRC_BRANCH" --force
git checkout "$CUR_BRANCH"
git branch -D "$TMP_B"


flag_create_local_target=0
x=$(git --no-pager branch --list "$TARGET_BRANCH" | wc -l)
if [[ $x -lt 1 ]]; then
    # 本地没有分支跟踪远程的TARGET_BRANCH,新建之
    git checkout -b "$TARGET_BRANCH" "${REPO_NAME}/$TARGET_BRANCH"
    flag_create_local_target=1
fi

git rebase "${REPO_NAME}/$SRC_BRANCH" "$TARGET_BRANCH"
git push "$REPO_NAME" "$TARGET_BRANCH":"$TARGET_BRANCH"

# 删除已经合并分支
git push "${REPO_NAME}" :"${SRC_BRANCH}"

git checkout "$CUR_BRANCH"

if [[ "$flag_create_local_target" -eq 1 ]]; then
    git branch -D "$TARGET_BRANCH"
fi

echo -e "\n\n\033[1;32;40m分支 $SRC_BRANCH 已经成功合入 $REPO_NAME/$TARGET_BRANCH\033[0m\n"

```

