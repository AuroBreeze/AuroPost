---
time: 2025-05-26 12:47:00
tags: [vim,nvim.lua]
---

# Quick-py插件

## 前言

最近几天在折腾`nvim`,想要用`nvim`来写`python`程序。

一开始是打算用别人的插件来直接写`python`代码的，可是我发现他们的插件我有点用不懂，再就是没有整合，没有激活虚拟环境，后运行代码的插件。

所以，我就打算自己写一个简单的插件来实现这个功能。

## 知识

阅读下面的内容默认读者已经懂了以下知识：

- `lua`语言的使用
- `nvim`的基本使用
- `lsp`服务
- `python`的虚拟环境
- `nvim`的部分api

>[!NOTE]
>使用的`nvim`的`api`也会在下面讲解。

## 实现功能

- 自动激活虚拟环境
- 一键运行代码
- 自定义执行命令(比如`django`的`python manage.py runserver`)

## 重要API介绍

```lua
vim.fn.expand('%:p:h')
-- 获取当前文件所在的目录

-- 示例输出：C:/code/AuroCC/utils
--  %   当前文件名
-- :p	转换为绝对路径（Path）
-- :h	获取目录部分（Head）
-- :t	获取文件名部分（Tail）
-- :r	去掉扩展名（Root）
-- :e	获取扩展名（Extension)
```

```lua
vim.fn.getcwd()
-- 获取当前工作目录

-- 示例输出：C:/code/AuroCC
```


```lua
vim.fn.isdirectory(cand)
-- 判断cand是否为目录
-- cand为要判断的目录路径
-- 示例输出：true/false
```

```lua
vim.fn.fnamemodify(path, mods)
-- 获取上级目录地址
-- path为文件路径
-- mods为修改符号 同expand
-- print(vim.fn.fnamemodify('C:/code/AuroCC/utils', ':h'))
-- 示例输出：C:/code/AuroCC
```

```lua
vim.fn.executable(cmd)
-- 判断cmd是否可执行
-- cmd为要判断的命令
-- print(vim.fn.executable('python'))
-- 示例输出：true/false
```

```lua
vim.fn.resolve(venv)
-- 析符号链接（symlink），将路径转换为物理存在的绝对路径。
-- 如果路径中包含 ~、.、.. 或符号链接，resolve() 会将其展开为完整的物理路径。
-- 要求路径必须存在，否则会返回空字符串。
-- print(vim.fn.resolve('~/.virtualenvs/AuroCC'))
-- 示例输出：/home/user/.virtualenvs/AuroCC
```

```lua
vim.fn.simplify(path)
-- 简化路径，将路径中多余的部分去掉。
-- 例如：simplify('C:/code/AuroCC/utils/../../utils') = 'C:/code/utils'
```

```lua
vim.defer_fn(fn, ms)
-- 延迟执行函数
-- fn为要执行的函数
-- ms为延迟时间（毫秒）
```

```lua
vim.fn.chansend(chan, data)
-- 向通道发送数据(向CMD窗口发送数据)
-- chan为通道号
-- data为要发送的数据
```

```lua
vim.fn.shellescape(str)
-- 将字符串转换为shell命令的参数。
-- 例如：shellescape('hello world') = 'hello\ world'
```

## 实现

### 自动激活虚拟环境






