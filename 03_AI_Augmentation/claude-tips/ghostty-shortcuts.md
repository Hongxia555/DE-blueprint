# Ghostty Shortcuts

Popular keyboard shortcuts for the Ghostty terminal emulator.
macOS defaults shown — on Linux replace `Cmd` with `Ctrl+Shift`.

## Tabs & Windows

| Action | Shortcut |
|---|---|
| New tab | `Cmd+T` |
| Close tab | `Cmd+W` |
| Next tab | `Cmd+Shift+]` |
| Previous tab | `Cmd+Shift+[` |
| New window | `Cmd+N` |

## Splits

| Action | Shortcut |
|---|---|
| Split right | `Cmd+D` |
| Split down | `Cmd+Shift+D` |
| Navigate splits | `Cmd+Option+Arrow` |
| Close split | `Cmd+W` |

## Scrollback

| Action | Shortcut |
|---|---|
| Scroll up | `Cmd+Shift+Up` / `Shift+PgUp` |
| Scroll down | `Cmd+Shift+Down` / `Shift+PgDn` |
| Scroll to top | `Cmd+Home` |
| Scroll to bottom | `Cmd+End` |

## Font Size

| Action | Shortcut |
|---|---|
| Increase font size | `Cmd++` |
| Decrease font size | `Cmd+-` |
| Reset font size | `Cmd+0` |

## Misc

| Action | Shortcut |
|---|---|
| Copy | `Cmd+C` |
| Paste | `Cmd+V` |
| Clear screen | `Cmd+K` |
| Command palette | `Cmd+Shift+P` |
| Quick terminal (toggle) | `` Cmd+` `` (if configured) |

## Customization

Bindings can be overridden in `~/.config/ghostty/config`:

```
keybind = cmd+t=new_tab
keybind = cmd+d=new_split:right
```
