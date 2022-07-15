---
title: "Vscode Settings"
date: 2022-07-13T23:25:41+08:00
draft: true
toc: false
tags: []
---

我的vscode配置备忘

## 用户空间配置

用户空间配置即不分具体的项目的一些通用公共配置

```json
{
    "editor.fontSize": 18,
    "editor.fontFamily": "DejaVu Sans Mono",
    "editor.renderWhitespace": "all",
    "workbench.tree.indent": 16,
    "java.jdt.ls.java.home": "/usr/lib/jvm/java-17-openjdk-amd64",
    "java.dependency.packagePresentation": "flat",
    "explorer.excludeGitIgnore": true,
    "explorer.fileNesting.enabled": true,
    "search.showLineNumbers": true,
    "java.dependency.syncWithFolderExplorer": true,
    "java.configuration.updateBuildConfiguration": "automatic",
    "vscode_custom_css.imports": [
        "file:///home/kang/.vscode/extensions/custom-css/custom.main.css"
    ],
    "vscode_custom_css.staging_dir": "/home/kang/.vscode/extensions/custom-css",
}
```

## 项目空间配置

具体项目相关的配置

### Java

```json
{
    "java.format.settings.url": "file:///path/to/java-code-format.xml",
    "java.completion.importOrder": [
        "java",
        "javax",
        "",
        "com.projectname",
        "#"
    ],
}
```

### 其他语言待用到时补充更新

## 自定义侧边栏字体

使用[Custom CSS and JS Loader](https://marketplace.visualstudio.com/items?itemName=be5invis.vscode-custom-css)插件自定义侧边栏的字体:

在vscode存放插件的目录里（`~/.vscode/extensions`）新建一个`custom-css`目录，存放下面的css文件：

> 必须放在vscode存放插件的目录里，不能在外面

custom.main.css
```css
body {
    font-family: "DejaVu Sans Mono";
}

.monaco-list {
    font-family: "DejaVu Sans Mono";
}
```

然后用户空间配置：

```json
{
    "vscode_custom_css.imports": [
        "file:///home/kang/.vscode/extensions/custom-css/custom.main.css"
    ],
    "vscode_custom_css.staging_dir": "/home/kang/.vscode/extensions/custom-css",
}
```