---
title: "我的vimrc"
date: 2021-01-30T11:22:39+08:00
draft: false
tags: ["vimrc"]
toc: false
---

我的vimrc, 设置基本的语法高亮，自动对齐，以及显示行尾空白字符，不设置过于复杂的功能，又能在默认的基础上得到较好的vim使用体验。

```bash
syntax on
filetype on
filetype plugin on
filetype indent on

set nu
set autoindent
set cindent
set tabstop=4
set softtabstop=4
set shiftwidth=4
set expandtab
set smarttab
set enc=utf-8
set ruler
set showcmd
set showmatch
set matchtime=1
set cmdheight=2
set scrolloff=3
set magic
set cursorline
set noeb
set confirm
set statusline=%<%f\ %h%m%r%=%-14.(%l,%c%V%)\ %P
set fileencodings=utf-8,gbk,latin1

highlight WhiteSpacesEOF ctermbg=red guibg=red
match WhiteSpacesEOF /\s\+$/
```
