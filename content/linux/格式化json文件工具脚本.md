---
title: "格式化json文件工具脚本"
date: 2022-04-19T15:14:34+08:00
draft: false
tags: []
toc: false
description: ""
---

用python3实现的格式化json文件的小工具脚本，放在自己的path下即可

```python3
#!/usr/bin/python3
# coding=utf-8

import json
import os
import shutil
import sys

usage = "Usage: %s [file name]" % sys.argv[0]

if len(sys.argv) != 2:
    print(usage)
    print("\n")
    sys.exit(-1)

file_name = sys.argv[1]

if not os.path.exists(file_name):
    print("file %s not exists\n" % file_name)
    sys.exit(-1)

with open(file_name, 'r') as f:
    try:
        json_data = json.load(f)
    except:
        print("解析json失败")
        sys.exit(-1)

tmp_file_name = file_name + ".tmp"

succ = True

with open(tmp_file_name, 'w') as f:
    try:
        json.dump(json_data, f, ensure_ascii=False, indent=4)
    except:
        print("生成格式化json失败\n")
        succ = False

if not succ:
    if os.path.exists(tmp_file_name):
        os.remove(tmp_file_name)
    sys.exit(-1)
else:
    shutil.move(tmp_file_name, file_name)
    print("成功\n")
```