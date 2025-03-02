+++
author = "GreyWind"
title = "Development Environment (Utlra)"
date = "2024-03-31"
description = "Development Environment (Utlra)"
tags = [
    "development",
]
image = "dev.webp"
+++
## Font

### [fira code](https://github.com/tonsky/FiraCode)

free monospaced font with programming ligatures

## Command line tools

### zsh

Zsh is an extended Bourne shell with many improvements, including some features of `Bash`, `ksh`, and `tcsh`.

**install**

Ubuntu

```bash
apt install zsh
```

MacOS default shell is zsh

[**On My Zsh**](https://ohmyz.sh/)

Oh My Zsh is a delightful, open source, community-driven framework for managing your Zsh configuration.

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

### [fish](https://fishshell.com/)

`fish` is a smart and user-friendly command line shell for Linux, macOS, and the rest of the family.

* `fish` is not compatible `bash`.
    

Ubuntu

```bash
apt install fish
```

Mac

```bash
brew install fish
```

### [tmux](https://github.com/tmux/tmux/wiki)

tmux is a terminal multiplexer. It lets you switch easily between several programs in one terminal, detach them (they keep running in the background) and reattach them to a different terminal.

* The prefix key `Ctrl + b` is not very good, you can change it to `Ctrl + a`.
    

Modify `~/.tmux.conf`

```plaintext
set -g prefix C-a
unbind C-b # 解除绑定 Ctrl-b
bind C-a send-prefix

# tmux 1.6 之后支持设置第二个前缀指令
# 设置一个不常用的键 ` 作为前缀键
set-option -g prefix2 `

# 设置 tmux 默认 shell
set -g default-shell /opt/homebrew/bin/fish
set -g default-command /opt/homebrew/bin/fish
```

### [eza](https://github.com/eza-community/eza)

A modern, maintained replacement for `ls`.

### [bat](https://github.com/sharkdp/bat)

A `cat(1)` clone with syntax highlighting and Git integration.

### [fzf](https://github.com/junegunn/fzf)

`fzf` is a general-purpose command-line fuzzy finder.

### [modern unix](https://github.com/ibraheemdev/modern-unix)

A collection of modern/faster/saner alternatives to common unix commands.

## [Warp](https://www.warp.dev/) (Terminal)

`Warp` is a modern, Rust-based terminal with `AI` built in so you and your team can build great software, faster.

## IDE & Edit

### [VS Code](https://code.visualstudio.com/)

#### extensions

* Power Mode
    
* Remote SSH
    
* Vim
    
* WakaTime
    
* Vibrancy Continued
    
* rust-analyzer
    
* VSCode Animations
    
* clangd
    
* TONGYI Lingma
    
* Git Graph
    
* indent-rainbow
    
* viscode-icons
    
* Prettier - Code formatter
    

### [OpenLens](https://k8slens.dev/)

Meet the new standard for cloud native software development & operations.  
With over 1 million users, Lens is the most popular **Kubernetes IDE** in the world.

### [Zed](https://zed.dev/) (Mac)

**Zed** is a high-performance, multiplayer code editor from the creators of Atom and Tree-sitter.

### [CodeEdit](https://www.codeedit.app/) (Mac)

**CodeEdit** is an exciting new code editor written entirely and unapologetically for macOS.

## Git Client

### [fork](https://git-fork.com/)