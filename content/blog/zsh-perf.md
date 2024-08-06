+++
title = "Extreme ZSH Performance"
date = "2024-08-06"
+++

This blog post is for those who want to improve their ZSH startup time. With a clean `.zshrc`, ZSH is quite fast. But once you start adding various things to it, it can slow down dramatically.

TLDR: Organize your `.zshrc` well and use [zsh-defer](https://github.com/romkatv/zsh-defer).


# Performance Analysis

While you could use everything from `zprof` to `hyperfine` and `time`, the most comprehensive way is likely via [zsh-bench](https://github.com/romkatv/zsh-bench). Just run
```sh
git clone https://github.com/romkatv/zsh-bench ~/zsh-bench && cd zsh-bench
```
in your terminal. Running the `zsh-bench` script prints the following for me:
```sh
==> benchmarking login shell of user araaha ...
creates_tty=1
has_compsys=1
has_syntax_highlighting=1
has_autosuggestions=1
has_git_prompt=0
first_prompt_lag_ms=237.871
first_command_lag_ms=263.405
command_lag_ms=40.244
input_lag_ms=11.732
exit_time_ms=90.718
```
on an AMD Ryzen 5 5500U.

The lines we want to focus on are the latter four:
```sh
first_prompt_lag_ms=237.871
first_command_lag_ms=263.405
command_lag_ms=37.558
input_lag_ms=11.732
```

The line `first_prompt_lag_ms` measures the time until the prompt appears. Similarly, `first_command_lag_ms` measures the time until you can start typing a command. `command_lag_ms` measures the time between the execution of a command and the first moment a prompt appears thereafter. Lastly, `input_lag_ms` measures the time between keystrokes. As reference, using a totally clean `.zshrc`, the following is printed out:
```sh
==> benchmarking login shell of user araaha ...
creates_tty=0
has_compsys=0
has_syntax_highlighting=0
has_autosuggestions=0
has_git_prompt=0
first_prompt_lag_ms=12.648
first_command_lag_ms=13.152
command_lag_ms=0.537
input_lag_ms=0.357
exit_time_ms=10.877
```

# Organizing ZSH
Originally, I had a `.zshrc` with hundreds of lines. I managed to reduce it to just 32.
In my opinion, making use of `$ZDOTDIR` to place your zsh config in one folder. In any case, I have mine exported as
```sh
export ZDOTDIR=/home/araaha/.config/zsh
```
in `/etc/zsh/zshenv`. That way, my `.zshrc` doesn't clog up `$HOME`. The layout is as follows:
```sh
zsh
 ├─ modules
 │  ├─ bindkeys.zsh
 │  ├─ zstyle.zsh
 │  ├─ options.zsh
 │  ├─ prompt.zsh
 │  ├─ compinit.zsh
 │  ├─ zvm.zsh
 │  ├─ zoxide.zsh
 │  ├─ exports.zsh
 │  ├─ tty.zsh
 │  ├─ history.zsh
 │  └─ aliases.zsh
 └─ plugins
    ├─ zsh-completion-generator
    ├─ zsh-defer
    ├─ functions.zsh
    ├─ fast-syntax-highlighting
    ├─ zsh-autosuggestions
    ├─ fzf.zsh
    └─ zsh-vi-mode
```
Every plugin I have is placed in `plugins` and everything else in `modules`. For example, `exports.zsh` includes exports I have. Similarly, `aliases.zsh` includes aliases I have.

# Zsh-defer
Now, rewriting your `.zshrc` is very simple. Personally, the first three lines in my `.zshrc` are:
```sh
export ZSH="$HOME/.config/zsh"
export PLUG="$ZSH/plugins"
export MOD="$ZSH/modules"
```
We can start sourcing our modules and plugins and use [zsh-defer](https://github.com/romkatv/zsh-defer) as much as possible. [zsh-defer](https://github.com/romkatv/zsh-defer) allows us to defer ZSH commands. It has to come before sourcing anything you want to defer. In some cases, you'll end up breaking things by deferring. In my case, I can't defer the [zsh-vi-mode](https://github.com/jeffreytse/zsh-vi-mode) plugin. I also don't defer my prompt, zsh opts and my history opts. In any case, the latter three don't siginificantly affect startup time.

By the end, I ended up with
```sh
export ZSH="$HOME/.config/zsh"
export PLUG="$ZSH/plugins"
export MOD="$ZSH/modules"

source "$PLUG/zsh-defer/zsh-defer.plugin.zsh"

#zvm must be above zsh-vi-mode
source "$MOD/zvm.zsh"
source "$PLUG/zsh-vi-mode/zsh-vi-mode.zsh"

zvm_after_init_commands+=("[[ -t 0 && $- = *i* ]] && stty -ixon")

source "$MOD/prompt.zsh"
source "$MOD/options.zsh"
source "$MOD/history.zsh"

zsh-defer source "$MOD/exports.zsh"
zsh-defer source "$MOD/aliases.zsh"
zsh-defer source "$MOD/bindkeys.zsh"

zsh-defer source "$MOD/zoxide.zsh"
zsh-defer source "$MOD/compinit.zsh"
zsh-defer source "$MOD/zstyle.zsh"

zsh-defer source "$PLUG/fzf.zsh"
zsh-defer source "$PLUG/functions.zsh"
zsh-defer source "$PLUG/zsh-autosuggestions/zsh-autosuggestions.zsh"
zsh-defer source "$PLUG/fast-syntax-highlighting/fast-syntax-highlighting.zsh"
```
which led to this result from [zsh-bench](https://github.com/romkatv/zsh-bench):
```sh
==> benchmarking login shell of user araaha ...
creates_tty=0
has_compsys=0
has_syntax_highlighting=0
has_autosuggestions=0
has_git_prompt=0
first_prompt_lag_ms=84.171
first_command_lag_ms=92.650
command_lag_ms=39.395
input_lag_ms=11.381
exit_time_ms=13.126
```

The plugins that affect performance the most are [zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions) and [zsh-vi-mode](https://github.com/jeffreytse/zsh-vi-mode). Without them, `command_lag_ms` is almost zero and `first_prompt_lag_ms`, `first_command_lag_ms` are respectively ~10-20ms.

# Additional Optimizations
## Compinit
`compinit` siginificantly affects startup time. We can cache it once a day so that to speed things up:
```sh
autoload -Uz compinit
if [ $(date +'%j') != $(date -r $ZSH/.zcompdump +'%j') ]; then
    compinit
else
    compinit -C
fi
```

## Zcompile
Compiling your plugins via `zcompile` can make an impact. E.g., using `zcompile` on [zsh-vi-mode](https://github.com/jeffreytse/zsh-vi-mode) reduces my startup time by ~10ms.

## Git Prompt
Using an async git prompt will likely improve performance.

## Oh-My-ZSH
OMZ has dozens of plugins, most of which you probably don't use. It's better to create your own config.
