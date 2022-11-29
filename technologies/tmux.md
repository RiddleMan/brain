# Tmux - Tips & Tricks

## Troubleshooting
### Tmux server doesn't start
Try to cleanup stored sessions by _tmux-resurrect_ by executing:
```sh
rm -Rf /tmp/tmux-* && rm -Rf ~/.tmux/resurrect/*
```