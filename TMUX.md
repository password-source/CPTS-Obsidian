

```
printf '%s\n' 'set -g mouse on' 'unbind -n C-z' 'unbind -n C-x' 'unbind -n C-n' 'unbind -n C-S-+' 'unbind -n C-S--' 'bind -n C-z split-window -h' 'bind -n C-x split-window -v' 'bind -n C-n new-window' 'bind -n C-S-+ if-shell -F '\''#{window_zoomed_flag}'\'' '\''display-message "Already zoomed"'\'' '\''resize-pane -Z'\''' 'bind -n C-S-- if-shell -F '\''#{window_zoomed_flag}'\'' '\''resize-pane -Z'\'' '\''display-message "Not zoomed"'\''' > ~/.tmux.conf && tmux source-file ~/.tmux.conf
```


|Shortcut|Action|Equivalent tmux command|
|---|---|---|
|`Ctrl + Z`|↔️ Split pane **left / right**|`split-window -h`|
|`Ctrl + X`|↕️ Split pane **top / bottom**|`split-window -v`|
|`Ctrl + N`|🆕 Create a new tab/window|`new-window`|
|`Ctrl + Shift + +`|🔍 Zoom **in** on current pane|`resize-pane -Z`|
|`Ctrl + Shift + -`|🔎 Zoom **out**|`resize-pane -Z`|
|🖱️ Mouse|🖱️ Select, resize, and interact with panes|`set -g mouse on`|