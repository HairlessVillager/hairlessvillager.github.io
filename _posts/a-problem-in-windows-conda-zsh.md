---
title: 在 Zsh 环境下配置 Conda/Mamba 踩坑记录
date: 2024-09-10 09:33
---

相关 Issue：
- https://github.com/ohmyzsh/ohmyzsh/issues/12416
- https://github.com/conda/conda/issues/13915
- https://github.com/conda/conda/issues/9922

## 其一

在 zsh 中运行命令 `mamba activate base` 提示运行 `mamba init zsh`，运行后重启终端发现无效。

观察输出，发现`.zshrc`配置文件的位置不对，输出在了 Windows 的 home 目录下（即`C:\user\username\.zshrc`），而我希望输出在 Zsh 的 home 目录下（即`D:\cygwin\home\username\.zshrc`）。

这个问题解决很简单，把前者新增内容添加到后者即可。

## 其二

再次运行`mamba init zsh`，提示 `` (eval):10: parse error near `^M' ``。

看到`^M`很容易就能猜出是换行符相关的问题。

参考上面 3 个 issue 后，给出我自己的解决方法：

1. 在 Zsh 的 home 目录下创建 `conda-shell-zsh-hook.zsh`文件，在保存时注意使用 LF 作为换行符：

```sh
export PYTHONIOENCODING=UTF-8
export CONDA_EXE="$(cygpath 'D:\Anaconda\anaconda3\Scripts\conda.exe')"
export _CE_M=''
export _CE_CONDA=''
export CONDA_PYTHON_EXE="$(cygpath 'D:\Anaconda\anaconda3\python.exe')"

# Copyright (C) 2012 Anaconda, Inc
# SPDX-License-Identifier: BSD-3-Clause
__conda_exe() (
    "$CONDA_EXE" $_CE_M $_CE_CONDA "$@" | tr -d '\r'
)

__conda_hashr() {
    if [ -n "${ZSH_VERSION:+x}" ]; then
        \rehash
    elif [ -n "${POSH_VERSION:+x}" ]; then
        :  # pass
    else
        \hash -r
    fi
}

__conda_activate() {
    if [ -n "${CONDA_PS1_BACKUP:+x}" ]; then
        # Handle transition from shell activated with conda <= 4.3 to a subsequent activation
        # after conda updated to >= 4.4. See issue #6173.
        PS1="$CONDA_PS1_BACKUP"
        \unset CONDA_PS1_BACKUP
    fi
    \local ask_conda
    ask_conda="$(PS1="${PS1:-}" __conda_exe shell.posix "$@")" || \return
    \eval "$ask_conda"
    __conda_hashr
}

__conda_reactivate() {
    \local ask_conda
    ask_conda="$(PS1="${PS1:-}" __conda_exe shell.posix reactivate)" || \return
    \eval "$ask_conda"
    __conda_hashr
}

conda() {
    \local cmd="${1-__missing__}"
    case "$cmd" in
        activate|deactivate)
            __conda_activate "$@"
            ;;
        install|update|upgrade|remove|uninstall)
            __conda_exe "$@" || \return
            __conda_reactivate
            ;;
        *)
            __conda_exe "$@"
            ;;
    esac
}

if [ -z "${CONDA_SHLVL+x}" ]; then
    \export CONDA_SHLVL=0
    # In dev-mode CONDA_EXE is python.exe and on Windows
    # it is in a different relative location to condabin.
    if [ -n "${_CE_CONDA:+x}" ] && [ -n "${WINDIR+x}" ]; then
        PATH="$(\dirname "$CONDA_EXE")/condabin${PATH:+":${PATH}"}"
    else
        PATH="$(\dirname "$(\dirname "$CONDA_EXE")")/condabin${PATH:+":${PATH}"}"
    fi
    \export PATH

    # We're not allowing PS1 to be unbound. It must at least be set.
    # However, we're not exporting it, which can cause problems when starting a second shell
    # via a first shell (i.e. starting zsh from bash).
    if [ -z "${PS1+x}" ]; then
        PS1=
    fi
fi

conda activate base
```

2. 在`.zshrc`目录中增加以下内容（如已存在则替换）：

```sh
# >>> conda initialize >>>
# !! Contents within this block are managed by 'conda init' !!
if [ -f '/cygdrive/d/Anaconda/anaconda3/Scripts/conda.exe' ]; then
    export PYTHONIOENCODING=UTF-8
    source ~/conda-shell-zsh-hook.zsh
fi

if [ -f "/cygdrive/d/Anaconda/anaconda3/etc/profile.d/mamba.sh" ]; then
    . "/cygdrive/d/Anaconda/anaconda3/etc/profile.d/mamba.sh"
fi
# <<< conda initialize <<<
```

启动 Zsh 后无报错，且在右侧显示环境名称，问题解决。