

```
printf '%s\n' 'set -g mouse on' 'unbind -n C-z' 'unbind -n C-x' 'unbind -n C-n' 'unbind -n C-S-+' 'unbind -n C-S--' 'bind -n C-z split-window -h' 'bind -n C-x split-window -v' 'bind -n C-n new-window' 'bind -n C-S-+ if-shell -F '\''#{window_zoomed_flag}'\'' '\''display-message "Already zoomed"'\'' '\''resize-pane -Z'\''' 'bind -n C-S-- if-shell -F '\''#{window_zoomed_flag}'\'' '\''resize-pane -Z'\'' '\''display-message "Not zoomed"'\''' > ~/.tmux.conf && tmux source-file ~/.tmux.conf
```

| Shortcut   | Action                         |
| ---------- | ------------------------------ |
| `Ctrl + Z` | ↔️ Split left/right            |
| `Ctrl + X` | ↕️ Split top/bottom            |
| `Ctrl + N` | 🆕 New tmux window             |
| `Ctrl + =` | 🔎 Increase terminal text size |
| `Ctrl + -` | 🔍 Decrease terminal text size |
| 🖱️ Mouse  | 🖱️ tmux mouse support         |