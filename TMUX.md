Use this updated one-line command:

```
printf '%s\n' 'set -g mouse on' 'unbind -n C-z' 'unbind -n C-x' 'unbind -n C-n' 'unbind -n C-a' 'bind -n C-z split-window -h' 'bind -n C-x split-window -v' 'bind -n C-n new-window' 'bind -n C-a resize-pane -Z' 'bind -n C-Up select-pane -U' 'bind -n C-Down select-pane -D' 'bind -n C-Left select-pane -L' 'bind -n C-Right select-pane -R' > ~/.tmux.conf && tmux source-file ~/.tmux.conf
```

### Your complete shortcuts now

|Shortcut|Action|
|---|---|
|`Ctrl+Z`|↔️ Split left/right|
|`Ctrl+X`|↕️ Split top/bottom|
|`Ctrl+N`|🆕 New tmux window|
|`Ctrl+A`|🔎 Zoom/unzoom current pane|
|`Ctrl+↑`|⬆️ Move to pane above|
|`Ctrl+↓`|⬇️ Move to pane below|
|`Ctrl+←`|⬅️ Move to pane left|
|`Ctrl+→`|➡️ Move to pane right|
|🖱️ Mouse|🖱️ Select/resize panes|