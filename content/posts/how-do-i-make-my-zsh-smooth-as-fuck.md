---
title: "我是如何让我的 Zsh 像丝般顺滑的"
date: 2022-05-30
tags: [shell, zsh]
---

## 引言

我用 Zsh 到现在大约三年了，从抛弃 Oh My Zsh 自行配置开始也有大约两年了，零零散散积攒了不少我觉得值得分享的东西，因此有了这篇 blog 。另外，考虑到我的朋友大多对 Zsh 的使用比较轻度，写 Bash 居多，这篇 blog 也会顺便讲解一些零碎的 Zsh 的小知识。

我的配置文件全都放在 [QuarticCat/dotfiles](https://github.com/QuarticCat/dotfiles) ，有兴趣的可以去翻阅一下。值得一提的是，管理这个 repo 所用的 dotfile manager 也是我用 Zsh 写的。

尽管在 repo 里我把所有 Zsh 的配置放到了一个文件夹里，但它们在我系统中是分开的，结构大致上是这样：

```text
- ~
  - .zshenv
  - $XDG_CONFIG_HOME/zsh
    - all-other-files
```

因为文件太多，全放在 home 目录会很乱，因此我遵循 [XDG Base Directory](https://wiki.archlinux.org/title/XDG_Base_Directory) 把大部分文件转移到了`$XDG_CONFIG_HOME`，只有`.zshenv`必须得放在 home 目录。下面我就一个一个文件介绍一下我都配置了些什么。

## `.zshenv`

关于 Zsh 几个配置文件的区别可以看这篇 [blog](https://shreevatsa.wordpress.com/2008/03/30/zshbash-startup-files-loading-order-bashrc-zshrc-etc/) 。在这里我主要放一些环境变量，这样它们对 DE 启动的 GUI 程序也生效（似乎是因为 SDDM 会自动`source`这个文件）。其中很多变量也可以用`~/.pam_environment`、`~/.xsession`等配置文件管理，但它们都都没有 Zsh 写得舒服，而且合在一起修改也比较方便（当然它们在效果上有轻微的差异）。

回到文件内容。首先是一个自己写的函数，主要是把`source`和`eval`简单包了一层，在文件不存在或指令出错的时候直接跳过。这是因为很多软件需要在 shell 里面挂 hook ，但有时我需要把我的 shell 配置快速移植到远程的开发机上，这些机子没有相应的软件就会报错。

```zsh
include() {
    case $1 in
    -f)
        [[ -f $2 ]] && source $2
        ;;
    -c)
        local output=$($=2) &>/dev/null && eval $output
        ;;
    *)
        echo 'Unknown argument!' >&2
        return 1
        ;;
    esac
}
```

然后是 XDG Base Directory 、输入法（很多教程把它们放在`~/.pam_environment`里）和默认编辑器的配置:

```zsh
export XDG_CONFIG_HOME=~/.config
export XDG_CACHE_HOME=~/.cache
export XDG_DATA_HOME=~/.local/share

export INPUT_METHOD='fcitx5'
export GTK_IM_MODULE='fcitx5'
export QT_IM_MODULE='fcitx5'
export XMODIFIERS='@im=fcitx5'

export EDITOR='vim'
export VISUAL='vim'
```

然后把`.zshrc`的查找路径指向`$XDG_CONFIG_HOME/zsh`，接下来的其他文件就都可以放在那里了。因为 Zsh 默认不 split word ，不需要到处加引号，看起来清晰很多。

```zsh
ZDOTDIR=$XDG_CONFIG_HOME/zsh
```

最后是`$PATH`的设置（以及挂 hook ，此处省略）。Zsh 可以把一个 array 变量绑定到一个 scalar 变量上，两者的值会同步变化。Zsh 默认给你绑定了`$path`和`$fpath`。你可以用`echo ${(t)var}`来查看`$var`的类型。在我的电脑上`$path`的类型默认是`array-tied-special`，我这里用`typeset -U path`给它设置`unique`属性，使得`$PATH`自动去重。

```zsh
typeset -U path
path=(  # no need to export
    ~/.local/bin
    ~/.cargo/bin
    ~/.ghcup/bin
    ~/go/bin
    $path
)
```

## `.zshrc` -> `zshrc.zsh`

我通过在`.zshrc`里`source $ZDOTDIR/zshrc.zsh`来把配置转移到了`zshrc.zsh`，这是因为一些地方不会识别出`.zshrc`这个“后缀”属于哪个语言因而不能正确高亮，另外`.`开头的文件有的地方会被默认忽略。但总的来说其实没有多少必要，也许哪天我又把配置转移回`.zshrc`了也说不定。

### 文件夹路径缩写

```zsh
hash -d config=$XDG_CONFIG_HOME
hash -d cache=$XDG_CACHE_HOME
hash -d data=$XDG_DATA_HOME
hash -d zdot=$ZDOTDIR

hash -d OneDrive=~/OneDrive
hash -d Downloads=~/Downloads
hash -d Workspace=~/Workspace
for p in ~Workspace/*; hash -d $(basename $p)=$p
for p in ~Code/*; hash -d $(basename $p)=$p
```

Zsh 可以通过`hash -d short=long`来设置一个 shortcut ，之后就可以用`~short`的语法访问`long`路径，算是 home 目录 tilde 语法的延伸，非常方便。不仅如此，在 [powerlevel10k](https://github.com/romkatv/powerlevel10k) 等一些 theme 里，如果你的当前路径有对应 shortcut ，那么它就会使用 shortcut 显示。同时你自己也可以实现这个功能，通过`${(D)path_var}`获取`$path_var`变量里所存路径的 shortcut 。

注意 Zsh 在这里的`for`可以省略`do`和`done`。后面还会用到更多可选写法，详情可见 [Zsh 文档](https://zsh.sourceforge.io/Doc/Release/Shell-Grammar.html#Alternate-Forms-For-Complex-Commands)。

### P10k Instant Prompt

启动 powerlevel10k instant prompt 。

```zsh
include -f ~config/p10k-instant-prompt-${(%):-%n}.zsh
```

我特别喜欢 p10k 的这个 feature ，它通过预先启动一个外观一致但功能不完整的 prompt 来让你感觉 shell 已经启动完毕了，此时后台继续执行其余的部分，最后再加载一个完整功能的 prompt 。即使我的 shell 启动时间已是飞快，用了这个 feature 都能明显感觉到提升。有了这个 feature 后 shell 的启动时间就已经不太重要了，感知大大减弱。

### 插件

```zsh
include -f ~zdot/.zgenom/zgenom.zsh

zgenom autoupdate  # every 7 days

if ! zgenom saved; then
    # load plugins and compile ~zdot
fi
```

我用的插件管理器是 [zgenom](https://github.com/jandamm/zgenom) 。这是我目前知道的最快的插件管理器，比知名的 [zinit](https://github.com/zdharma-continuum/zinit) 还快不少，但功能上少非常多，倒也合理。它用了一个比较取巧的方法，在加载完一次插件后，它会把一些解析后的结果写入到一个 init script 里面，之后就只加载这个 init script ，从而省去了一点时间。但这也带来了一些不便，当想要修改插件设置的时候就需要多一些步骤，而且无法在插件的加载中间插入指令。不过它也支持动态加载。我喜欢它的主要原因除了快还有简单，有任何问题我自己看代码都能解决，甚至能发个 PR 。另一个我也很喜欢的轻量级插件管理器是 [zcomet](https://github.com/agkozak/zcomet) 。

现在很多插件管理器（包括我上面提到的三个）还支持预先把 Zsh 文件编译成 Zwc 文件来加速加载。这是一个 Zsh 本身提供的功能，所以并不难实现。

Zinit 用户可能会想要 zinit 里的延迟加载功能。但实际上这个功能已经完全不稀奇了，上面提到的两个轻量级插件管理器应该都可以通过组合 [zsh-defer](https://github.com/romkatv/zsh-defer) 来实现延迟加载。而且延迟加载需要处理的问题比较多，十分麻烦，例如需要重新`compinit`或者加载一些别的东西。再加上我有 instant prompt ，感觉这个功能已经没太大用处了。

我用到的插件大概有下面这些：

- Oh My Zsh 的一小部分

  - lib/completion.zsh：提供了一些补全的基本设置。这个文件里很坑的一点是把`$WORDCHAR`—— Zsh 用来分词的配置——设置成了空，并且为了兼容无法改回来。如果用了它的话记得重新设置`$WORDCHAR`。关于 Zsh 的分词机制会在下面介绍。

  - lib/clipboard.zsh：提供了`clipcopy`和`clippaste`这两个很好用的函数，用来在 terminal 里复制和粘贴剪贴板，用法如`echo 123 | clipcopy`和`clippaste | xxx`。

  - plugins/sudo：双击 Esc 切换`sudo`和无`sudo` / `sudoedit`和`EDITOR` ，如果当前 buffer （就是已经输入但还没提交的那些字符）为空则取上一条指令。

  - plugins/extract：提供了一个`extract`函数和`x`别名，用来自动识别压缩包后缀并解压，用法如`x 123.zip`。

  - plugins/pip：提供了 pip 的自动补全。因为需要把 pip 包索引缓存下来，因此要加载整个插件而不能只加载补全文件。

  - plugins/rust：提供了 rustc、rustup、cargo 的补全。其中后两者通过在插件入口文件里调用`rustup completions zsh`来实现，也就是说 rustup 本身就附带了这些补全文件。实际上如果软件本身附带补全文件的话，系统的包管理器一般会帮你一并打包，因此我这里只加载了 rustc 的补全。

  - plugins/docker-compose：提供了 docker-compose 的补全和一些 subcommand 别名。我只加载了补全，因为有了后面提到的 fzf-tab ，补全变得非常方便，单纯 subcommand 完全没有 alias 的必要。

  有些 OMZ 插件和 theme 会依赖于 lib/git.zsh ，如果遇到 git 相关功能异常可以考虑加上它。

- [nix-zsh-completions](https://github.com/spwhitt/nix-zsh-completions)：提供了 nix 的一些补全和别名，还有一个自动检测 nix-shell 并显示在 prompt 上的 precmd hook 。我只加载了补全，最后一个功能 p10k 已经包含了。

- [zsh-nix-shell](https://github.com/chisui/zsh-nix-shell)：让 nix-shell 使用 Zsh 作为默认 shell 。

- [fzf-tab](https://github.com/Aloxaf/fzf-tab)：使用 [fzf](https://github.com/junegunn/fzf) 来选择补全的插件，好用到起飞。非常多朋友看到我的 terminal 截图来问我这个插件是啥。强烈建议移步 repo 页面看效果演示。此外还有一个值得关注的竞品 [zsh-autocomplete](https://github.com/marlonrichert/zsh-autocomplete) ，我还没尝试过。我大概率不会切换到这个插件，但我可能从那里抄一些好用的补全配置并且彻底抛弃 OMZ 的 completion lib 。

- [zdharma-continuum/fast-syntax-highlighting](https://github.com/zdharma-continuum/fast-syntax-highlighting)：必备插件之一 [zsh-syntax-highlighting](https://github.com/zsh-users/zsh-syntax-highlighting) 的全面上位替代品。原作者删库后由社区继续维护（具体事件见 [reddit](https://www.reddit.com/r/zsh/comments/qinb6j/httpsgithubcomzdharma_has_suddenly_disappeared_i/hil4oww/)），但非常不活跃，没有新功能开发，我发的修复 global alias 高亮问题的 PR 也一直没有下文。

- [zsh-autosuggestions](https://github.com/zsh-users/zsh-autosuggestions)：必备插件之一，建议移步 repo 页面看效果演示。

- [zsh-history-substring-search](https://github.com/zsh-users/zsh-history-substring-search)：同上。注意，这个插件没有帮你绑定按键，而是只提供给你 widget 让你自行绑定。好评如潮！

- [zsh-edit](https://github.com/marlonrichert/zsh-edit)：提供了一个更合理的分词机制以及相应的键位绑定。原先 Zsh 只根据哪些字符属于一个 word 来分词（用`$WORDCHAR`控制），也就是移动到非 word 字符为止，准确率非常低。这个插件把相邻字符的差异也考虑进去作为分词边界，舒服了很多，建议移步 repo 页面看示例。这个插件还提供了一些其他功能，不过我并不关心（甚至想 fork 出一份精简版）。

- [QuarticCat/zsh-autopair](https://github.com/QuarticCat/zsh-autopair)：我自己维护的一个 [hlissner/zsh-autopair](https://github.com/hlissner/zsh-autopair) 的 fork ，去除了 Ctrl-Backspace 的键位绑定。原插件把这个键位绑定成和 Backspace 一样，非常诡异，一般人 Ctrl-Backspace 都是删除一个词。

我没有使用任何的 autojump / z / z.lua / zoxide 一类插件，对我来说`hash -d`和 fzf-tab 的组合已经让我路径输入体验十分舒适了。实际上我在使用 Zsh 的第一年经常用 z.lua ，后来换了 zoxide 。直到有一天我发现我已经很久很久没有打开过 zoxide 了，我就知道我已经不需要它了。

另外，有一个使用 SQLite 来管理 shell history 的软件 [atuin](https://github.com/ellie/atuin) ，也许能替换掉几个 history 相关的插件。我正打算尝试，但我非常怀疑它和 Zsh 的整合效果。

### 补全和函数

```zsh
fpath=(
    ~zdot/completions
    ~zdot/functions
    $fpath
)
autoload -Uz ~zdot/functions/*(:t)
```

函数我没有写在 zshrc 文件里面。Sukka 的 [《我就感觉到快 —— zsh 和 oh my zsh 冷启动速度优化》](https://blog.skk.moe/post/make-oh-my-zsh-fly/) 介绍了一种 lazyload function 的方法，很多人都看过。但实际上 Zsh 本身就提供了这个功能，也就是我最后一行写的`autoload -Uz`，只需要你把函数拆出到文件即可。当你没有使用过这个函数时，执行`which func`可以看到函数定义里只有一行`builtin autoload -XUz`，执行过一次后才加载成函数在文件里的定义。

后面的`~zdot/functions/*`是普通的 glob ，`(:t)`则是 qualifier ，效果类似`basename`。关于 qualifier ，可以看 Aloxaf 的这个教程[《【ZSH 系列教程】历史扩展与修饰符》](https://www.aloxaf.com/2020/11/zsh_history_expansion/)。这个系列教程后面的 parameter expansion 也很值得看。对于 qualifier 和 parameter expansion 这种非常难记的语法我后面就略过不讲了，因为我自己也是每次用到都要查的。

目前我暂时没有自己写的补全（除了后文会提到的那个），使用的函数也都很简单，就不具体介绍了。有兴趣的可以自己去 repo 里翻代码。

### 配置

只挑一些比较重要的写。

#### zsh

```zsh
# zsh misc
setopt auto_cd               # simply type dir name to cd
setopt auto_pushd            # make cd behaves like pushd
setopt pushd_ignore_dups     # don't pushd duplicates
setopt pushd_minus           # exchange the meanings of `+` and `-` in pushd
setopt interactive_comments  # comments in interactive shells
setopt multios               # multiple redirections
setopt ksh_option_print      # make setopt output all options
setopt extended_glob         # extended globbing
setopt no_bare_glob_qual     # disable `PATTERN(QUALIFIERS)`, extended_glob has `PATTERN(#qQUALIFIERS)`
WORDCHARS='*?_-.[]~=&;!#$%^(){}<>'  # remove '/'

# zsh history
setopt hist_ignore_all_dups  # no duplicates
setopt hist_save_no_dups     # don't save duplicates
setopt hist_ignore_space     # no commands starting with space
setopt hist_reduce_blanks    # remove all unneccesary spaces
setopt share_history         # share history between sessions
HISTFILE=~zdot/.zsh_history
HISTSIZE=50000
SAVEHIST=10000
```

这些`setopt`主要功能都写在注释里了。`$WORDCHARS`也是之前介绍过的。`$HISTFILE`、`$HISTSIZE`、`$SAVEHIST`这三个分别指定存指令历史记录的文件、文件大小限制、条目数限制。

```zsh
# zsh completion
compdef _galiases -first-
_galiases() {
    if [[ $PREFIX == :* ]]; then
        local des
        for k v ("${(@kv)galiases}") des+=("${k//:/\\:}:alias -g '$v'")
        _describe 'alias' des
    fi
}
zstyle ':completion:*:git-checkout:*' sort false
zstyle ':completion:*:git-rebase:*' sort false
zstyle ':completion:*:git-revert:*' sort false
zstyle ':completion:*:git-reset:*' sort false
zstyle ':completion:*:git-diff:*' sort false
```

这里我给自己的 global alias 写了一个补全。因为补全类型比较特殊不知道怎么拆成文件，就留在这了。Global alias 是 Zsh 特有的功能，通过`alias -g xxxx=yyy`设置，在指令的任何地方遇到单独的`xxx`都会被替换为`yyy`，而不只是指令的开头。通常设置成全大写来避免意外替换，我则是采用冒号开头。其实在 fast-syntax-highlighting 里 global alias 会被突出显示，基本不会意外替换的。这个补全的效果如下：

![galias-comp](/galias-comp.svg)

然后剩下的就是一些补全的配置。我关掉了很多 git 指令的补全排序，因为它们补全的是 commit ，默认就是按照时间顺序给出的，按字符串排序后反而乱了。我正在考虑要不要默认关闭掉所有补全的排序，并不只有 git 会给出有内在顺序的补全选项。

由于我自己对 Zsh 的补全系统也很不了解，这块就不展开讲语言知识了……

#### fzf

```zsh
export FZF_DEFAULT_OPTS='--ansi --height=60% --reverse --cycle --bind=tab:accept'
```

主要想提一下`--bind=tab:accept`，用 Tab 键来选择，使得补全体验与默认情况以及代码编辑器里的更接近。

Fzf 真的是一个特别好用的命令行基础设施。模糊搜索这个功能可以与许多其他的功能组合在一起，创造出更强大的功能，[这里](https://github.com/junegunn/fzf/blob/master/ADVANCED.md#ripgrep-integration)就有一些例子。

#### fast-syntax-highlighting

```zsh
unset 'FAST_HIGHLIGHT[chroma-man]'  # chroma-man will stuck history browsing
```

一个 bug 的暂时解决方案。因为原 repo 没了，我现在也不知道修了没有，如果有遇到可以像我这样设置。原作者推荐`FAST_HIGHLIGHT[chroma-man]=`，这样写若是 fast-syntax-highlighting 没加载就会报错，因为对一个不存在的变量取了索引，而用`unset`就不会，不过其实也没啥用。

#### zsh-autosuggestions

```zsh
ZSH_AUTOSUGGEST_MANUAL_REBIND='1'
```

默认情况下 zsh-autosuggestions 为了防止它绑定的按键被其他插件覆盖，会挂一个 precmd hook ，每条指令都给你重新绑定一次键位，就比较蠢，而且会拖慢速度。因为我对自己的键位绑定非常清楚所以就改为手动了。

#### zsh-history-substring-search

```zsh
HISTORY_SUBSTRING_SEARCH_FUZZY='1'
```

默认把`ab c`当成一个整体来搜索，开了这个以后当成`*ab*c*`来搜索。

#### man-pages

```zsh
export MANPAGER='sh -c "col -bx | bat -pl man --theme=Monokai\ Extended"'
export MANROFFOPT='-c'
```

配置 man-pages 使用 [bat](https://github.com/sharkdp/bat) 作为 pager ，bat 在这里可以提供语法高亮。我的 bat 默认使用的 theme 是 OneHalfDark ，这里特意指定了用 Monokai Extended ，这是我试过一遍后感觉对 man-pages 高亮效果最好的。效果如下：

![bat-man-page](/bat-man-page.svg)

Bat 的仓库里还有许多别的用法示例。

### 别名

```zsh
alias l='exa -lah --group-directories-first --git --time-style=long-iso'
alias lt='l -TI .git'
alias clc='clipcopy'
alias clp='clippaste'
alias clco='tee >(clipcopy)'  # clicpcopy + stdout
alias sc='sudo systemctl'
alias scu='systemctl --user'
alias sudo='sudo '
alias cgp='cgproxy '
alias pc='proxychains -q '
alias open='xdg-open'
alias with-proxy=' \
    http_proxy=$MY_PROXY \
    HTTP_PROXY=$MY_PROXY \
    https_proxy=$MY_PROXY \
    HTTPS_PROXY=$MY_PROXY '

alias -g :n='/dev/null'
alias -g :bg='&>/dev/null &'
alias -g :bg!='&>/dev/null &!'  # &!: background + disown
```

正如我前面所说，我并不喜欢 subcommand 别名。我现在补全的体验很舒适，我会重度依赖补全。在不用补全的时候，写`ab`确实比`aaa bbb`快，但在用补全的时候，原先我可能只需要按`a<TAB>b<TAB>`就能打出来，有了 alias 后我按`a<TAB>`将会出现一大堆候选项，反而慢了。或者可能原先打`a<TAB>`是 X 个候选项，有了 alias 后就是 XY 个候选项了。而且 alias 的补全是没解释文本的，而 subcommand 的补全往往是有的。这也是为啥我不喜欢`apt-*`和`nix-*`，虽然它们不是 alias 。

那么多长得差不多的 alias 对记忆力也是挑战。我曾经用过很长一段时间的 [zsh-you-should-use](https://github.com/MichaelAquilina/zsh-you-should-use) 来帮助我记忆 alias ，最后发现不用记 alias 才是最舒服的。所以我在 alias 的数量方面一直都很克制。

注意到很多 alias 后面有一个空格，这是 Zsh 的一个优秀设计。Zsh 会首先看第一个参数是不是 alias ，如果是的话就替换。如果这个 alias 最后有一个空格，那么会尝试展开下一个参数，然后看它有没有空格。这使得我们可以在 alias 前面使用 precommand ，如`sudo`、`cgproxy`、`proxychains`等。所有的 precommand 都应该 alias 一遍并加上末尾空格。Precommand + alias 这么常见的需求不懂为什么 Fish 迟迟不解决。

这些 alias 里面除了 global alias 外我觉得最有意思的就是`with-proxy`。一些软件（主要是用 go 写的那些）无法被 proxychains 代理，可以试一下环境变量。有个语法是`VAR=XXX command`，把`$VAR`传进去作为这个指令的环境变量，比较方便，还限制了作用范围。这个 alias 就是用这个语法简化了传递一堆 proxy 变量的操作，用起来和其他 precommand 一模一样。我曾经把这个设计提交给了 Sukka 的 [zsh-proxy](https://github.com/SukkaW/zsh-proxy) ，得到一句 LGTM 之后就被晾着了……

## `key-bindings.zsh`

Zsh 默认的按键绑定是非常难用的，啥都没有。如果你用 OMZ 的 lib/key-bindings.zsh 的话它会帮你绑定相当多的常用按键，非常贴心。不过对我来说它的设置基本都被后面的一些插件覆盖了（比如 zsh-edit 和 zsh-autopair ），所以就没用。由于插件提供了很多我需要的按键绑定，我自己绑定的键位其实不多（相比那些从头开始自己绑按键的 Zsh 用户）。

如果你不知道你的按键被绑了啥，可以执行`bindkey`查看全部按键绑定，或者`bindkey <key>`来查看某个按键的绑定，有时你可以意外地发现某个插件给你绑了奇怪的东西。

```zsh
bindkey -r '^['  # [Esc] (Default: vi-cmd-mode)

bindkey '^Z' undo         # [Ctrl-Z]
bindkey '^Y' redo         # [Ctrl-Y]
bindkey '^Q' push-line    # [Ctrl-Q]
bindkey ' '  magic-space  # [Space] Do history expansion

# Widgets are from zsh-history-substring-search
bindkey '^[[A' history-substring-search-up    # [UpArrow]
bindkey '^[[B' history-substring-search-down  # [DownArrow]
```

- 解绑 Esc ，默认的 vi-cmd-mode 我甚至都不知道怎么退出（臭名昭著的勒索软件.jpg）。

- Undo 和 redo ，很多人可能都不知道 Zsh 还提供了这种功能。

- Push-line 的作用是保存你当前 buffer 的内容然后清空，等你执行完一条指令后再把保存的内容恢复到你的 buffer ，这对于输入到一半要执行其他指令时非常有用。

- Magic-space 的作用见前文提到的 Aloxaf 的文章。

```zsh
# Trim trailing newline from pasted text
bracketed-paste() {
    zle .$WIDGET && LBUFFER=${LBUFFER%$'\n'}
}
zle -N bracketed-paste
```

从 https://unix.stackexchange.com/questions/693118 改来的代码。如果你三连击 code block 任意一行，你就会全选这一行，包括后面的换行。如果此时复制粘贴这一行到 Zsh 并执行，那么这一个多余的换行也会在 history 里被保留下来。这里通过替换默认的`bracketed-paste`组件来在粘贴后自动修改要粘贴的内容以解决这个问题。链接里有更详细的描述。

这里用到了 Zsh 的`$'xxx'`语法，这个语法会对里面的转义字符进行转义，不像 Bash 还得依靠`echo -en`之类的东西。

Zsh 每个内置组件都有一个`.`开头的别名，用来在组件被替换的时候访问原组件，因此这里写`zle .$WIDGET`。

```zsh
# [Ctrl+L] clear screen while maintaining scrollback
fixed-clear-screen() {
    # FIXME: works incorrectly in tmux
    local prompt_height=$(echo -n ${(%%)PS1} | wc -l)
    local lines=$((LINES - prompt_height))
    printf "$terminfo[cud1]%.0s" {1..$lines}  # cursor down
    printf "$terminfo[cuu1]%.0s" {1..$lines}  # cursor up
    zle reset-prompt
}
zle -N fixed-clear-screen
bindkey '^L' fixed-clear-screen
```

从 https://superuser.com/questions/1389834 改来的代码。Terminal 通常会保留你之前的输入输出，往上滚动的时候可以看到这些历史信息，称为 scrollback 。系统自带（？）的 clear 程序不仅会清屏，也会清除 scrollback ，而很多时候我们只是希望当前输入行被顶到上面去而已。这里用了一种移植性不太好但效果还不错的办法：发出特殊的控制指令让 terminal 把指针往下移一屏把当前输入行顶上去，再把指针往上移回去，最后调用`reset-prompt`恢复指针的正常横坐标。

链接最高赞末尾有一个很好的设计，就是绑定到 Enter ，然后检测当前 buffer ，如果为空就清屏，不为空就提交指令。但它前面的部分有点冗长，我这里利用 Zsh 的特性进行了大幅简化。Zsh 的`printf`如果参数是一个数组就会对每个数组元素应用前面的格式串。我们可以利用这个特性在格式串里塞一个零宽度的格式`%.0s`然后把它应用到一个 N 个元素的数组里，从而输出同样的字符串 N 遍。这个特性还有一个很常见的用法是`printf '%s\n' $path`，一行一个元素地查看`$PATH`。

```zsh
# [Ctrl-R] Search history by fzf-tab
fzf-history-search() {
    local selected=$(
        fc -rl 1 |
        ftb-tmux-popup -n '2..' --tiebreak=index --prompt='cmd> ' ${BUFFER:+-q$BUFFER}
    )
    if [[ $selected != '' ]] {
        zle vi-fetch-history -n $selected
    }
    zle reset-prompt
}
zle -N fzf-history-search
bindkey '^R' fzf-history-search
```

从 [Aloxaf 的 dotfiles](https://github.com/Aloxaf/dotfiles/blob/0619025cb2/zsh/.config/zsh/snippets/key-bindings.zsh#L80-L102) 改来的。其实 fzf 的 repo 里就提供了好几个[按键绑定](https://github.com/junegunn/fzf/blob/master/shell/key-bindings.zsh)，让你用 fzf 搜索历史、跳转路径等等。这里自己重写一个主要是为了用上 fzf-tab 提供的 [ftb-tmux-popup](https://github.com/Aloxaf/fzf-tab/wiki/Configuration#fzf-command) ，它在 tmux 里视觉效果非常好。（快点前一个链接！）

```zsh
# [Ctrl-N] Navigate by xplr
bindkey -s '^N' '^Q cd -- ${$(xplr):-.} \n'
```

这是一个很精巧的按键绑定。[Xplr](https://github.com/sayanarijit/xplr) 是一个命令行文件浏览器，如果你在里面选择了一个文件/文件夹，它会退出并输出对应的路径。基于这点我们可以用`cd -- $(xplr)`来使用 xplr 快速导航。但如果我们中途决定直接退出的话，xplr 会返回空字符串，这时`cd`就会报错。因此我们包装一下，这种情况下返回`.`，现在就变成了`cd -- ${$(xplr):-.}`。为了方便，我们用`bindkey -s`把它作为一条指令（而不是 widget）绑定到按键上。先`^Q`进行 push-line ，然后加一个空格使得这条指令不进入历史记录（别忘记我前面设置了`hist_ignore_space`），最后回车提交指令。我也把这行精巧的代码分享到了 xplr 的 [discussion](https://github.com/sayanarijit/xplr/discussions/254) 。它的仓库里也还有许多别的用法示例。

## 速度

有了 instant prompt ，启动速度对我已经不那么重要了。不过还是让我们看看，在我配置了这么多东西，用了这么多插件的情况下，究竟能达到什么速度。我的 CPU 是 3700X ，以下是我的 Zsh 以 interative 模式启动（会加载`.zshrc`）到执行完第一条指令退出的时间：

```console
$ hyperfine 'zsh -ic exit'
Benchmark 1: zsh -ic exit
  Time (mean ± σ):      61.6 ms ±   2.4 ms    [User: 41.8 ms, System: 21.8 ms]
  Range (min … max):    59.7 ms …  75.4 ms    38 runs
```
