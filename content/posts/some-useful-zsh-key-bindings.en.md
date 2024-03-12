---
title: "Some useful Zsh key-bindings"
date: 2024-03-12
tags: [zsh]
---

## Built-in

### `undo` & `redo`

No need to explain. They work as their name suggest. Many Zsh users remain unaware of these two widgets since they are not bound in `viins` keymap (which is the default for most users). I bind them to `[Ctrl+Z]` & `[Ctrl+Y]`, respectively.

### `magic-space`

It behaves like normal space except that it can perform [history expansion](https://zsh.sourceforge.io/Doc/Release/Expansion.html#History-Expansion). That is to say, it turns `!-1` to `<last command> ` in your buffer. Usually bound to `[Space]`.

### `push-line`

> Push the current buffer onto the buffer stack and clear the buffer. Next time the editor starts up, the buffer will be popped off the top of the buffer stack and loaded into the editing buffer.

It allows you to leave halfway to execute one other command and then resume. Usually bound to `[Ctrl+Q]`.

### `push-line-or-edit`

> At the top-level (PS1) prompt, equivalent to push-line. At a secondary (PS2) prompt, move the entire current multiline construct into the editor buffer. The latter is equivalent to push-input followed by get-line.

A more powerful version of `push-line`. Usually bound to `[Ctrl+Q]`.

If you want to learn more, there are a few good places to go:

- https://zsh.sourceforge.io/Doc/Release/Zsh-Line-Editor.html
- https://github.com/zsh-users/zsh/tree/master/Functions/Zle

## Mine

### Word movement

```zsh
# Ref: https://github.com/marlonrichert/zsh-edit
qc-word-widgets() {
    if [[ $WIDGET == *-shellword ]] {
        local words=(${(Z:n:)BUFFER}) lwords=(${(Z:n:)LBUFFER})
        if [[ $WIDGET == *-backward-* ]] {
            local tail=$lwords[-1]
            local move=-${(N)LBUFFER%$tail*}
        } else {
            local head=${${words[$#lwords]#$lwords[-1]}:-$words[$#lwords+1]}
            local move=+${(N)RBUFFER#*$head}
        }
    } else {
        local subword='([[:WORD:]]##~*[^[:upper:]]*[[:upper:]]*~*[[:alnum:]]*[^[:alnum:]]*)'
        local word="(${subword}|[^[:WORD:][:space:]]##|[[:space:]]##)"
        if [[ $WIDGET == *-backward-* ]] {
            local move=-${(N)LBUFFER%%${~word}(?|)}
        } else {
            local move=+${(N)RBUFFER##(?|)${~word}}
        }
    }
    if [[ $WIDGET == *-kill-* ]] {
        (( MARK = CURSOR + move ))
        zle -f kill
        zle .kill-region
    } else {
        (( CURSOR += move ))
    }
}
for w in qc-{back,for}ward-{,kill-}{sub,shell}word; zle -N $w qc-word-widgets
bindkey '^[[1;5D' qc-backward-subword         # [Ctrl+Left]
bindkey '^[[1;5C' qc-forward-subword          # [Ctrl+Right]
bindkey '^[[1;3D' qc-backward-shellword       # [Alt+Left]
bindkey '^[[1;3C' qc-forward-shellword        # [Alt+Right]
bindkey '^H'      qc-backward-kill-subword    # [Ctrl+Backspace] (in Konsole)
bindkey '^W'      qc-backward-kill-subword    # [Ctrl+Backspace] (in VSCode)
bindkey '^[[3;5~' qc-forward-kill-subword     # [Ctrl+Delete]
bindkey '^[^?'    qc-backward-kill-shellword  # [Alt+Backspace]
bindkey '^[[3;3~' qc-forward-kill-shellword   # [Alt+Delete]
```

[zsh-edit](https://github.com/marlonrichert/zsh-edit)'s README explains the effect very well:

This is how built-in word widgets (`{for,back}ward-word`, `{,backward-}-kill-word`) move:

```zsh
# Zsh with default WORDCHARS='*?_-.[]~=/&;!#$%^(){}<>' moves/deletes way too much:
#              >       >             >             >          >
% ENV_VAR=value command --option-flag camelCaseWord ~/dir/*.ext
# <             <       <             <             <

# Zsh with  WORDCHARS='' is bit better, but skips punctuation clusters & doesn't find subWords:
#    >   >     >         >      >    >               >     >
% ENV_VAR=value command --option-flag camelCaseWord ~/dir/*.ext
# <   <   <     <         <      <    <               <     <
```

This is how zsh-edit subword widgets move:

```zsh
# Zsh Edit with WORDCHARS=''
#   >   >     >       >  >     >    >     >   >   >  >  >      >
% ENV_VAR=value command --option-flag camelCaseWord ~/dir/?*.ext
# <   <   <     <       < <      <    <    <   <    < <      <

# Zsh Edit with WORDCHARS='~*?'
#  > >   >  >   >
% cd ~/dir/?*.ext
# <  < <   <  <
```

This function also includes shell-word widgets to let you move with greater strides. They are more intuitive than Zsh's [`select-word-style`](https://github.com/zsh-users/zsh/blob/master/Functions/Zle/select-word-style) in some edge cases.

### Accept line

**Update 2024-03-12**: I find that Roman Perepelitsa has made an excellent plugin called [zsh-no-ps2](https://github.com/romkatv/zsh-no-ps2) for this purpose. It's more thoughtful and rigorous than my widget.

```zsh
# [Enter] Insert `\n` when accept-line would result in a parse error or PS2
# Ref: https://github.com/romkatv/zsh4humans/blob/v5/fn/z4h-accept-line
qc-accept-line() {
    if [[ $(functions[_qc-test]=$BUFFER 2>&1) == '' ]] {
        zle .accept-line
    } else {
        LBUFFER+=$'\n'
    }
}
zle -N qc-accept-line
bindkey '^M' qc-accept-line
```

On occasion, we may unintentionally hit a key -- like `'` -- before hitting enter. And that will result in a secondary prompt (PS2), which is poorly supported, and your first line will be recorded in history anyway.

In contrast, my widget gives you a newline instead in that scenario, so you can just delete back to correct your command. Moreover, my widget makes writing multi-line commands easier.

### Trim paste

```zsh
# Trim trailing whitespace from pasted text
# Ref: https://unix.stackexchange.com/questions/693118
qc-trim-paste() {
    zle .bracketed-paste
    LBUFFER=${LBUFFER%%[[:space:]]#}
}
zle -N bracketed-paste qc-trim-paste
```

Sometimes, we want to copy some command from the browser, say:

```zsh
echo 'hello world'
```

We tripple-click it to select the whole line and then copy. However, tripple-clicking also selects the newline character. This could be annoying when pasting to the terminal. Have a try!

Thankfully, we have `bracketed-paste` handling all pasted text. We can hook pasting by overwriting this widget. The same trick is used by [bracketed-paste-magic](https://github.com/zsh-users/zsh/blob/master/Functions/Zle/bracketed-paste-magic) and [bracketed-paste-url-magic](https://github.com/zsh-users/zsh/blob/master/Functions/Zle/bracketed-paste-url-magic).

### Rationalize dot

```zsh
# Change `...` to `../..`
# Ref: https://grml.org/zsh/zsh-lovers.html#_completion
qc-rationalize-dot() {
    if [[ $LBUFFER == *.. ]] {
        LBUFFER+='/..'
    } else {
        LBUFFER+='.'
    }
}
zle -N qc-rationalize-dot
bindkey '.' qc-rationalize-dot
bindkey '^[.' self-insert-unmeta  # [Alt+.] insert dot
```

It's a better alternative to traditional `...`/`....`/`.....` aliases. It's more readable and supports unlimited levels.

### Clear screen

```zsh
# [Ctrl+L] Clear screen but keep scrollback
# Ref: https://superuser.com/questions/1389834
qc-clear-screen() {
    local prompt_height=$(echo -n ${(%%)PS1} | wc -l)
    local lines=$((LINES - prompt_height))
    printf "$terminfo[cud1]%.0s" {1..$lines}  # cursor down
    printf "$terminfo[cuu1]%.0s" {1..$lines}  # cursor up
    zle .reset-prompt
}
zle -N qc-clear-screen
bindkey '^L' qc-clear-screen
```

Zsh has a built-in `clear-screen` widget bound to `[Ctrl+L]` by default. But it also erases your terminal emulator's whole history in addition to the screen. My widget can keep the scrollback.

And it's more flexible. You can customize prompt position by adjusting the number of lines. For example, `local lines=$((LINES/2))` moves the prompt to the middle.

### Fuck

```zsh
# [Esc Esc] Correct previous command
# Ref: https://github.com/ohmyzsh/ohmyzsh/blob/master/plugins/thefuck
qc-fuck() {
    local fuck=$(THEFUCK_REQUIRE_CONFIRMATION=false thefuck $(fc -ln -1) 2>/dev/null)
    if [[ $fuck != '' ]] {
        compadd -Q $fuck
    } else {
        compadd -x '%F{red}-- no fucks given --%f'
    }
}
zle -C qc-fuck complete-word qc-fuck
bindkey '\e\e' qc-fuck
```

It's a powerful replacement of oh-my-zsh's [sudo plugin](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/sudo). It calls `thefuck` to correct previous comamnd. There are many cases that sudo plugin cannot handle, for example:

```zsh
echo 1 > /sys/devices/system/cpu/cpufreq/boost
```

You can't simply prefix it with `sudo` as `echo` is not a program but a shell built-in command. Now, [thefuck](https://github.com/nvbn/thefuck) to the rescue. It can smartly prompt `sudo sh -c "echo 1 > /sys/devices/system/cpu/cpufreq/boost"`.

It's better than using `fuck` as well. Because you can edit the command before executing it.

## More

Here are some key-bindings that aren't up my street but are still useful.

### fzf

[fzf](https://github.com/junegunn/fzf) is a fuzzy finder. It can be integrated to many commands to provide a handy selection menu. fzf offers some completions and key-bindings under the `shell` folder. For Zsh, that includes:

- `[Ctrl+T]` - Paste the selected file path(s) into the command line
- `[Alt+C]` - `cd` into the selected directory
- `[Ctrl+R]` - Paste the selected command from history into the command line

I also recommend you to take a look at [fzf-tab](https://github.com/Aloxaf/fzf-tab), a fabulous completion plugin.

### xplr

[xplr](https://github.com/sayanarijit/xplr) is a TUI file explorer. Its [document](https://xplr.dev/en/awesome-hacks) offers a bunch of hacks. I used to enjoy this key-binding:

```zsh
# [Ctrl+N] Navigate by xplr
# This is not a widget since properly resetting prompt is hard
# See https://github.com/romkatv/powerlevel10k/issues/72
bindkey -s '^N' '^Q cd -- $(xplr --print-pwd-as-result) \n'
```

I've spent enough time on this blog. There are more awesome projects but I'm too lazy to introduce all of them. Some are listed below.

- [mouse.zsh](https://github.com/matschaffer/oh-my-zsh-custom/blob/master/mouse.zsh)
- [zce.zsh](https://github.com/hchbaw/zce.zsh)
- [mcfly](https://github.com/cantino/mcfly)
- [atuin](https://github.com/atuinsh/atuin)
