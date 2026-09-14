

```
printf '%s\n' 'set -g mouse on' 'unbind -n C-z' 'unbind -n C-x' 'unbind -n C-n' 'bind -n C-z split-window -h' 'bind -n C-x split-window -v' 'bind -n C-n new-window' > ~/.tmux.conf && tmux source-file ~/.tmux.conf
```

| Shortcut   | Action                         |
| ---------- | ------------------------------ |
| `Ctrl + Z` | ↔️ Split left/right            |
| `Ctrl + X` | ↕️ Split top/bottom            |
| `Ctrl + N` | 🆕 New tmux window             |
| `Ctrl + =` | 🔎 Increase terminal text size |
| `Ctrl + -` | 🔍 Decrease terminal text size |
| 🖱️ Mouse  | 🖱️ tmux mouse support         |