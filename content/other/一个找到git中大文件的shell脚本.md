---
title: "一个找到git中大文件的shell脚本"
date: 2020-12-20T11:27:39+08:00
draft: false
tags: ["git", "shell"]
---


```bash
#!/bin/bash

# 进入到git仓库根目录，替换为实际路径，或使用参数动态传递
cd /home/zk/sangfor/code/ngsoc || exit 1

# 设置bash环境变量IFS为换行符，方便后面循环处理git命令的输出
IFS=$'\n'

# 使用verify-pack输出所有blob文件信息
objects=$(git verify-pack -v .git/objects/pack/pack-*.idx | grep blob | sort -k3nr | head -n 20)

output='文件大小 SHA 路径'
for x in $objects
do
    size=$(($(echo "$x" | cut -d ' ' -f 5)/1024))
    sha=$(echo "$x" | cut -d ' ' -f 1)
    path=$(git rev-list --all --objects | grep "$sha")
    # path的内容已经包含了sha和文件路径，由空格分隔
    output="$output\n$size $path"
done

echo -e "$output" | column -t -s ' '
```