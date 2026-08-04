Here is your clean, source-free tmux cheat sheet:

## 🗂️ Session Management

Run these directly from your standard terminal shell:

- `tmux`: Start a new unnamed session.
- `tmux new -s <name>`: Start a session with a specific name.
- `tmux ls`: List all active running sessions.
- `tmux attach`: Reattach to your most recent session.
- `tmux attach -t <name>`: Reattach to a specific named session.
- `tmux kill-session -t <name>`: Destroy a specific session.

Interactive shortcuts (inside tmux):

- `Ctrl+b` then `d`: Detach from the session (leaves it running).
- `Ctrl+b` then `s`: Open an interactive list to switch sessions.

---

## 🪟 Windows (Tabs)

- `Ctrl+b` then `c`: Create a brand new window.
- `Ctrl+b` then `,`: Rename the current window.
- `Ctrl+b` then `n`: Move to the next window.
- `Ctrl+b` then `p`: Move to the previous window.
- `Ctrl+b` then `0–9`: Jump directly to a window by its number.
- `Ctrl+b` then `w`: Choose a window from a visual tree menu.
- `Ctrl+b` then `&`: Kill (close) the current window.

---

## 🧱 Panes (Splits)

- `Ctrl+b` then `%`: Split the current pane vertically (left/right).
- `Ctrl+b` then `"`: Split the current pane horizontally (top/bottom).
- `Ctrl+b` then `Arrow Key`: Move cursor focus to that pane.
- `Ctrl+b` then `z`: Zoom/Maximize the active pane (press again to minimize).
- `Ctrl+b` then `x`: Close the active pane safely.
- `Ctrl+b` then `Space`: Cycle through built-in grid layouts.
- `Hold Ctrl+b` + `Arrow Keys`: Dynamically resize the active pane.

---

## 📜 Scroll & Copy Mode

- `Ctrl+b` then `[`: Enter copy mode (allows terminal history scrolling).
- `q`: Exit copy mode and return to live terminal entry.
- `Space`: Begin selecting text block.
- `Enter`: Save selected text to buffer and exit copy mode.
- `Ctrl+b` then `]`: Paste the contents from the buffer.

---

## ⚙️ Utilities

- `Ctrl+b` then `:`: Open the internal tmux command prompt line.
- `Ctrl+b` then `?`: View a reference page of all key bindings.
- `tmux source-file ~/.tmux.conf`: Reload your configuration file.

Would you like help setting up mouse scrolling or creating a custom bash alias to automatically open your favorite window layout?