---
title: NeoVim 中 Conda 环境下配置 LSP 踩坑记录
date: 2024-05-06 23:05
tags:
---

## 问题引入

笔者在 Windows 环境下使用 Anaconda 包管理软件管理 Python 的第三方库，但是一直以来 NeoVim 的 LSP 不支持 Conda 除 base 环境的其他环境的第三方包（即 envs/ 下的环境），一直以来都很困扰我。今天实在忍无可忍了，决定把这个问题探个究竟。

我使用的是 [python-lsp-server](https://github.com/python-lsp/python-lsp-server) 这个 LSP。`nvim-lspconfig.lua` 部分配置展示：
```lua
return {
    {
        'neovim/nvim-lspconfig',
        config = function()
            -- Setup language servers.
            local lspconfig = require('lspconfig')
            local coq = require("coq")
            lspconfig.pylsp.setup(coq.lsp_ensure_capabilities {
                cmd = {
                    "D:/Anaconda/anaconda3/Scripts/pylsp.exe",
                },
                settings = {
                    pylsp = {
                        plugins = {
                            pycodestyle  = {
                                maxLineLength = 120,
                            },
                            pydocstyle = {
                                convention = "numpy",
                            },
                        }
                    }
                }
            })
            ...
```

## 在脚本里打日志

省略中间的各种弯路，最终在 python-lsp-server 的 GitHub 上找到一个相关的 discussions [#177](https://github.com/python-lsp/python-lsp-server/discussions/177)。[@skeledrew](https://github.com/python-lsp/python-lsp-server/discussions/177#discussioncomment-2443852) 提出他写了一个[脚本](https://gitlab.com/-/snippets/2279333)，能替换 `pylsp/workspace.py` 中的函数，使其找到祖先路径下的 `.env` 文件并加载。

这个脚本质量不算很高，很多写法不规范，但是最大的问题是他根本就不能工作，而且我打开 NeoVim 后 CPU 占用飙升。好在作者给了一个 [write](https://gitlab.com/-/snippets/2279333#L149) 函数，可以勉强当日志用。确定到问题出在后面的 while 循环是死循环，`cur_path`变量一直是`WindowsPath('e:/')`。当然很容易通过计数器变量来限制循环次数，这个问题先按下不表。

观察日志输出：
```text
No activate command found in e:\code\python\.env
Loopping... cur_path=WindowsPath('e:/code/python')
Loopping... cur_path=WindowsPath('e:/code')
Loopping... cur_path=WindowsPath('e:/')
Loopping... cur_path=WindowsPath('e:/')
Loopping... cur_path=WindowsPath('e:/')
Loopping... cur_path=WindowsPath('e:/')
...
```

我的工作目录在 `E:/code/python/chatcollector/chatcollector/`，这里显然是有问题的，多走了两次父目录。经过我不断在 pylsp 包中打日志，最后确认为 `pylsp.python_lsp.PythonLSPServer.m_initialize` 的入参为 `E:/code/python/chatcollector/`，是正常工作目录的一级父目录。这说明调用 pylsp 的调用者在传参的时候出现了问题。

## 从 pylsp 到 nvim-lspconfig

找这个调用者的过程也是极其艰辛，以 `m_initialize` 为关键词是无法在一般的搜索引擎上找到结果的。最后还是在 `pylsp.workspace.Workspace` 找到了 `M_INITIALIZE_PROGRESS` 这个类属性，对应的值为 `"window/workDoneProgress/create"`。我注意到这里的字符串格式应该是和 LSP 相关，于是在 NeoVim 中使用命令 `:LspLog` 打开日志，并且把日志等级设为 DEBUG。最后找到了多条形如以下的日志：

```log
[DEBUG][2024-05-06 15:14:07] .../vim/lsp/rpc.lua:387	"rpc.receive"	{  id = "a81c18d5-a48f-4ea6-bc76-3fb014022ebd",  jsonrpc = "2.0",  method = "window/workDoneProgress/create",  params = {    token = "3d640408-5c1f-4e8f-9a73-9fe2bf4c13e5"  }}
```

然后我联想到我的 pylsp 是配置在 [neovim/nvim-lspconfig](https://github.com/neovim/nvim-lspconfig) 上的，于是打开 neovim/nvim-lspconfig 的默认 pylsp 配置，找到了以下代码：

```lua
    root_dir = function(fname)
      local root_files = {
        'pyproject.toml',
        'setup.py',
        'setup.cfg',
        'requirements.txt',
        'Pipfile',
      }
      return util.root_pattern(unpack(root_files))(fname) or util.find_git_ancestor(fname)
    end,
```

它的意思是，从当前的目录向祖先方向找，如果这个目录包含 `pyproject.toml` 之类的文件就判断它为工作目录。

修改如下：

```lua
local util = require('lspconfig.util')
...
    root_dir = function(fname)
        local root_files = {
            '.env',
            'pyproject.toml',
            'setup.py',
            'setup.cfg',
            'requirements.txt',
            'Pipfile',
        }
        return util.root_pattern(unpack(root_files))(fname) or util.find_git_ancestor(fname)
    end,
```

## 再到脚本里打日志

用上文提到的脚本替换 `pylsp/workspace.py` 文件后发现，实际使用时还是无法提示，打开日志 `pylsp_env_path_patcher.log` 显示：

```text
Error getting env list: FileNotFoundError(2, '系统找不到指定的文件。', None, 2, None)
```

发现原来是通过子进程系统调用 `conda info -e` 时报错，我 Conda 没加到系统路径里。用绝对路径替代之。结果又是死循环，加计数器和日志调试，发现中间有一次循环中 `env_fp=WindowsPath('e:/code/python/chatcollector/chatcollector/.env')`，这里确实是我 .env 文件的位置，但是它忽略了，原因是 "No python interpreter at D:\Anaconda\anaconda3\envs\django"。关键代码如下：

```python
if not (env_path / "bin/python").is_file():
    write(f"No python interpreter at {env_path}")
    continue
```

这里应该是 Linux 系统下的写法，移植到 Windows 下：

```python
if not (env_path / "python.exe").is_file():
    write(f"No python interpreter at {env_path}")
    continue
```

## LspLog 到 pylsp 再到 jedi 的一个 PR

依旧没有语法提示。查看 `:LspLog`（这里已经将\r\n转义并且用正确的编码打开）：

```log
[ERROR][2024-05-06 16:20:42] .../vim/lsp/rpc.lua:734	"rpc"	"D:\\Anaconda\\anaconda3\\Scripts\\pylsp.exe"	"stderr"	'2024-05-06 16:20:42,165 中国标准时间 - WARNING - pylsp.config.config - Failed to load hook pylsp_hover: D:\\Anaconda\\anaconda3\\envs\\django\\Scripts\\python.exe seems to be missing.
Traceback (most recent call last):
  File "D:\\Anaconda\\anaconda3\\Lib\\site-packages\\pylsp\\config\\config.py", line 39, in _hookexec
    return self._inner_hookexec(hook_name, methods, kwargs, firstresult)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "D:\\Anaconda\\anaconda3\\Lib\\site-packages\\pluggy\\_manager.py", line 327, in traced_hookexec
    return outcome.get_result()
           ^^^^^^^^^^^^^^^^^^^^
  File "D:\\Anaconda\\anaconda3\\Lib\\site-packages\\pluggy\\_result.py", line 60, in get_result
    raise ex[1].with_traceback(ex[2])
  File "D:\\Anaconda\\anaconda3\\Lib\\site-packages\\pluggy\\_result.py", line 33, in from_call
    result = func()
             ^^^^^^
  File "D:\\Anaconda\\anaconda3\\Lib\\site-packages\\pluggy\\_manager.py", line 324, in <lambda>
    lambda: oldcall(hook_name, hook_impls, kwargs, firstresult)
            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "D:\\Anaconda\\anaconda3\\Lib\\site-packages\\pluggy\\_callers.py", line 60, in _multicall
    return outcome.get_result()
           ^^^^^^^^^^^^^^^^^^^^
  File "D:\\Anaconda\\anaconda3\\Lib\\site-packages\\pluggy\\_result.py", line 60, in get_result
    raise ex[1].with_traceback(ex[2])
  File "D:\\Anaconda\\anaconda3\\Lib\\site-packages\\pluggy\\_callers.py", line 39, in _multicall
    res = hook_impl.function(*args)
          ^^^^^^^^^^^^^^^^^^^^^^^^^
  File "D:\\Anaconda\\anaconda3\\Lib\\site-packages\\pylsp\\plugins\\hover.py", line 14, in pylsp_hover
    definitions = document.jedi_script(use_document_path=True).infer(**code_position)
                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "D:\\Anaconda\\anaconda3\\Lib\\site-packages\\pylsp\\workspace.py", line 33, in wrapper
    return method(self, *args, **kwargs)
           ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "D:\\Anaconda\\anaconda3\\Lib\\site-packages\\pylsp\\workspace.py", line 534, in jedi_script
    self.get_enviroment(environment_path, env_vars=env_vars)
  File "D:\\Anaconda\\anaconda3\\Lib\\site-packages\\pylsp\\workspace.py", line 566, in get_enviroment
    environment = jedi.api.environment.create_environment(
                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "D:\\Anaconda\\anaconda3\\Lib\\site-packages\\jedi\\api\\environment.py", line 367, in create_environment
    return Environment(_get_executable_path(path, safe=safe), env_vars=env_vars)
                       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  File "D:\\Anaconda\\anaconda3\\Lib\\site-packages\\jedi\\api\\environment.py", line 380, in _get_executable_path
    raise InvalidPythonEnvironment("%s seems to be missing." % python)
jedi.api.environment.InvalidPythonEnvironment: D:\\Anaconda\\anaconda3\\envs\\django\\Scripts\\python.exe seems to be missing.
'
```

我的 `python.exe` 路径是 `D:\\Anaconda\\anaconda3\\envs\\django\\python.exe`，观察代码发现还是 pylsp 的问题。定位到以下代码：

```py
environment_path = jedi_settings.get("environment") or get_conda_env_path(self)
```

打个日志，输出为 "environment_path='D:\\Anaconda\\anaconda3\\envs\\django'"，结果没有问题。看来需要怀疑 jedi 了。定位到以下代码：

```py
def _get_executable_path(path, safe=True):
    """
    Returns None if it's not actually a virtual env.
    """

    if os.name == 'nt':
        python = os.path.join(path, 'Scripts', 'python.exe')
    else:
        python = os.path.join(path, 'bin', 'python')
    if not os.path.exists(python):
        raise InvalidPythonEnvironment("%s seems to be missing." % python)

    _assert_safe(python, safe)
    return python
```

看了一下，好像没有别人提出这个问题，于是顺便拉了个 [PR](https://github.com/davidhalter/jedi/pull/1994)。

再试了一下，好像没问题了。

## 收尾工作

首先重新安装以下被刚刚的日志破坏得千疮百孔的包：
```text
python-lsp-server
jedi
```
然后需要打两个补丁，首先运行脚本 `pyls_env_path_patcher.py`，给 `pylsp\workspace.py` 打补丁，然后查看我提的 [PR](https://github.com/davidhalter/jedi/pull/1994/files)，按照 diff 手动修改代码打补丁（如果你的 jedi 版本高于 0.19.1 可能不用修改），最后在 NeoVim 输入命令 `:LspRestart` 重启 LSP，就能修复这个问题了。