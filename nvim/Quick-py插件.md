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

---

## 实现

### 寻找虚拟环境

1. 获取当前文件的位置
2. 向上寻找目录，直到找到`venv`或`.venv`目录(虚拟环境目录可自行配置)

这两个目的的实现并不难，因为我们其中也使用了`{"ahmedkhalf/project.nvim"}`这个项目，他可以帮我们打开项目的文件，不过需要自己配置寻找`venv`目录的逻辑。

我们通过使用`vim.fn.fnamemodify()`,`vim.fn.expand()`,`vim.fn.isdirectory()`来实现。

```lua
local function find_local_venv(start_dir)
    local dir = start_dir or vim.fn.expand('%:p:h') -- 获取当前文件所在目录
    if dir == '' then dir = vim.fn.getcwd() end -- 如果没有就使用当前工作目录
    while dir and dir ~= '/' and dir ~= '' do -- 递归查找
        for _, name in ipairs(config.venv_names) do -- 遍历设置的虚拟环境名称
            local cand = dir .. '/' .. name -- 将虚拟环境名称与目录拼接
            if vim.fn.isdirectory(cand) == 1 then -- 验证拼接的目录是否存在
                return dir, cand
            end
        end
        dir = vim.fn.fnamemodify(dir, ':h') -- 获取上一级目录
    end
    return nil, nil
end
```

### 设置虚拟环境

1. 获取虚拟环境下的`python`,`pyright`,`activate`命令
2. 设置环境变量`VIRTUAL_ENV`,设置python路径`vim.g.python3_host_prog`


```lua
function M.get_venv()
    if M.cached_venv_dir and vim.fn.executable(config.python_path) == 1 then -- 缓存的虚拟环境可用
        return M.cached_venv_dir
    end

    local buf_dir = vim.fn.expand('%:p:h') -- 获取当前文件所在目录
    local root_dir, venv = find_local_venv(buf_dir) -- 获取虚拟环境目录
    if not root_dir then
        vim.notify("[Quick-py] 未找到 .venv 或 venv", vim.log.levels.WARN)
        return nil
    end

    venv = vim.fn.resolve(venv) -- 将路径展开
    venv = vim.fn.simplify(venv) -- 处理路径中无用的字符
    local is_win = vim.fn.has('win32') == 1 -- 判断系统类型
    if is_win then -- windows系统需要将路径中的分隔符替换为反斜杠
        venv = venv:gsub('/', '\\'):gsub('\\+$', '')
    else
        venv = venv:gsub('\\', '/'):gsub('/+$', '')
    end

    if M.cached_venv_dir and M.cached_venv_dir ~= venv then  -- 缓存的虚拟环境不匹配
        vim.lsp.stop_client(vim.lsp.get_active_clients({ name = 'pyright' })) -- 停止全局pyright服务
        M.lsp_started = false
    end

    -- 三元条件判断
    -- 若is_win为true（Windows系统），则路径为venv目录下的\\Scripts\\python.exe 否则（Unix/Linux系统），路径为venv目录下的/bin/python
    local pybin = is_win and (venv .. '\\Scripts\\python.exe') or (venv .. '/bin/python')
    if vim.fn.executable(pybin) == 0 then -- 验证python可执行
        vim.notify("[Quick-py] Python 不可执行: " .. pybin, vim.log.levels.ERROR)
        return nil
    end

    vim.env.VIRTUAL_ENV = venv -- 设置环境变量
    if is_win then
        vim.env.PATH = venv .. "\\Scripts;" .. vim.env.PATH
    else
        vim.env.PATH = venv .. "/bin:" .. vim.env.PATH
    end
    config.python_path = pybin -- 缓存python路径
    vim.g.python3_host_prog = pybin -- 缓存python路径到全局变量
    M.cached_root = root_dir -- 缓存根目录
    M.cached_venv_dir = venv -- 缓存虚拟环境目录
    vim.notify("[Quick-py] 已激活虚拟环境: " .. venv, vim.log.levels.INFO)
    return venv
end
```

### 终端自动激活虚拟环境

这个功能需要用到`自动命令`即`autocmd`

我们可以用`vim.api.nvim_create_autocmd()`来创建自动命令，在打开终端的时候自动执行激活虚拟环境的脚本。

我们需要用到`vim.api.nvim_create_augroup()`和`vim.api.nvim_create_autocmd()`来创建自动命令组和自动命令。

`augroup`的作用是将多个自动命令分组，方便管理。

`autocmd`的作用是当满足条件时执行指定的命令。

```lua

local aug = vim.api.nvim_create_augroup('ActiveVenv', { clear = true }) -- 创建自动命令组
-- ActiveVenv为自动命令组名,随便起
vim.api.nvim_create_autocmd('TermOpen', {
    -- TermOpen为自动命令名，需要去查看nvim的文档
    -- 在nvim中使用 :h autocmd 可以查看自动命令的文档
    -- TermOpen 事件及打开终端时触发
    pattern = '*',
    group = aug,
    callback = function()
        local venv = M.get_venv() -- 获取虚拟环境目录
        local chan = vim.b.terminal_job_id -- 获取终端号
        if venv and chan then
            vim.defer_fn(function() -- 延迟执行，等待终端加载完成
                if vim.fn.has('win32') == 1 then
                    vim.fn.chansend(chan, '"' .. venv .. '\\Scripts\\activate.bat"\r') 
                else
                    vim.fn.chansend(chan, 'source ' .. venv .. '/bin/activate\n')
                end
            end, 50)
        end
    end,
})
```

### 使用虚拟环境的pyright


```lua
vim.api.nvim_create_autocmd({ 'BufReadPost', 'BufNewFile' }, {
    pattern = "*.py", -- 匹配Python文件
    group = aug,
    callback = function()
        local venv = M.get_venv()
        if not venv then return end

        if not M.lsp_started then
            local ok, lspconfig = pcall(require, 'lspconfig') -- 加载lspconfig插件
            if ok then
                lspconfig.pyright.setup({
                    cmd = (function()
                        -- local root, _ = M.get_venv() -- 设置环境变量，并返回虚拟环境目录
                        -- local _, venv = find_local_venv(root or vim.fn.getcwd())
                        local venv = vim.env.VIRTUAL_ENV -- 读取环境变量
                        local is_win = vim.fn.has('win32') == 1
                        if is_win then venv = venv:gsub('/', '\\'):gsub('\\+$', '') end
                        local server = is_win and (venv .. '\\Scripts\\pyright-langserver.exe') or
                            (venv .. '/bin/pyright-langserver')
                        if vim.fn.executable(server) == 1 then
                            return { server, '--stdio' } -- 找到pyright-langserver时，使用配置
                        else
                            return { 'pyright-langserver', '--stdio' } -- 未找到pyright-langserver时，使用默认配置
                        end
                    end)(),
                    root_dir = function(fname) -- fname 是 root_dir 函数的输入参数，表示当前文件的路径。它由语言服务器协议（LSP）自动传入，用于定位项目根目录。
                        local root, _ = find_local_venv(fname) -- 寻找虚拟环境目录
                        if root then return root end
                        return lspconfig.util.root_pattern('.git', 'pyproject.toml', 'setup.py')(fname) -- 设置根目录
                    end,
                    on_new_config = function(new_config)
                        -- local _, venv = find_local_venv(new_root_dir)
                        local venv = vim.env.VIRTUAL_ENV -- 读取环境变量
                        if venv then
                            local is_win = vim.fn.has('win32')
                            if is_win == 1 then venv = venv:gsub('/', '\\'):gsub('\\+$', '') end
                            local python_venv_path = is_win and (venv .. '\\Scripts\\python.exe') or
                            (venv .. '/bin/python')
                            new_config.cmd = { new_config.cmd[1], '--stdio' }
                            new_config.settings = new_config.settings or {}
                            new_config.settings.python = { analysis = { pythonPath = python_venv_path } }
                        end
                    end,
                })
                M.lsp_started = true
            end
        end
    end
})
```

### 一键运行代码

创建自己的自定义命令和快捷键

```lua
-- 自定义命令
vim.api.nvim_create_user_command('RunPython', function()
    if not config.python_path then
        vim.notify("[Quick-py] 未激活虚拟环境", vim.log.levels.ERROR)
        return
    end
    local cmd -- 处理自定义命令
    if config.runserver_cmd then
        cmd = config.runserver_cmd
    else 
        cmd = "python" .. ' ' .. vim.fn.shellescape(vim.fn.expand('%:p')) -- 默认命令为运行当前文件
    end
    local ok, betterTerm = pcall(require, 'betterTerm') -- 加载betterTer插件
    if ok then
        local chan = betterTerm.open(0) -- 打开终端号为0的终端
        if not chan then
            betterTerm.open(0) -- 如果终端未打开，先打开
        end
        vim.defer_fn(function()
            betterTerm.send(cmd .. '\r',0) -- 发送运行命令
        end, 200)

        betterTerm.open(0) -- 第一次打开的终端不会显示在屏幕上，需要再次打开一次
    else
        -- 普通终端模式：直接执行（需用户手动激活环境）
        vim.cmd('!' .. cmd)
    end
end, { desc = 'Run current Python file in virtualenv' })

vim.keymap.set("n", "<leader>rp", ":RunPython<CR>", { desc = "Run Python file" })
```


## 总结

具体代码在[这里](https://github.com/AuroBreeze/quick-py)

以上就是`Quick-py`插件的全部内容，希望能帮到大家。







