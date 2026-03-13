# marimo-nvim Design Document

## Context

marimo is a reactive Python notebook that stores notebooks as pure `.py` files
with `@app.cell` decorated functions. While marimo provides a browser-based
editor, there is no native Neovim editing experience. The browser editor remains
the best way to view cell outputs (charts, tables, markdown rendering), but the
code editing experience should feel native to Neovim.

This plugin bridges that gap: edit marimo notebooks in Neovim with full LSP
support, while marimo's browser UI handles rendering via `--watch` mode.

## Goals

- Edit marimo notebooks in Neovim with a native feel
- Hide boilerplate (`@app.cell`, `def _():`, `return`) — show only cell bodies
- Full LSP support (completion, diagnostics, hover, go-to-definition)
- Cell-level operations (add, delete, move, navigate, toggle markdown/python)
- Bidirectional sync with marimo via file watching
- Launch and manage the marimo watch server from Neovim

## Non-Goals

- Render cell output inside Neovim (browser handles this)
- WebSocket communication with marimo (protocol is internal/undocumented/unstable)
- Custom "cell mode" in MVP (future feature, see appendix)

## Architecture Overview

```
+-------------------+       +-------------------+       +------------------+
|                   |  1:1  |                   | write |                  |
|  Virtual Buffer   | line  |  Hidden Buffer    |  -->  |  File on Disk    |
|  (user sees)      | map   |  (real .py file)  |       |  (notebook.py)   |
|                   |       |  LSP attached here |       |                  |
+-------------------+       +-------------------+       +------------------+
        |                                                       |
        | folds hide                                   marimo check --fix
        | boilerplate                                    (on save)
        |                                                       |
        v                                                       v
  +------------+                                      +------------------+
  |  User edits|                                      | marimo edit      |
  |  cell body |                                      | --watch picks up |
  |  lines     |                                      | file changes     |
  +------------+                                      +------------------+
```

### Two-Buffer Model

The plugin maintains two buffers per notebook:

1. **Hidden buffer**: Contains the real marimo `.py` file content exactly as it
   exists on disk. LSP (pyright, pylsp, etc.) attaches to this buffer. This
   buffer is never displayed to the user.

2. **Virtual buffer**: Displayed to the user. Contains the same number of lines
   as the hidden buffer (1:1 line mapping), but:
   - Cell body lines are **dedented** (4-space indent removed)
   - Boilerplate lines (`@app.cell`, `def _():`, `return ...`, inter-cell
     blanks, header, footer) are **placeholder lines** hidden inside folds
   - Folds render as visual separators via custom `foldtext`

### Why Two Buffers?

The virtual buffer strips boilerplate and dedents code, so its text content
doesn't match the real file. LSP servers (pyright, etc.) need the real Python
file to understand types, imports, and cross-cell variable references. By
keeping the real file in a hidden buffer with LSP attached, we get full
language intelligence without a translation layer for the LSP protocol itself.

## Marimo File Format

A marimo notebook on disk looks like this:

```python
import marimo                        # header

__generated_with = "0.20.4"          # header
app = marimo.App()                   # header


@app.cell                            # cell overhead
def _():                             # cell overhead
    x = 1                            # cell 1 body
    y = 2                            # cell 1 body
    return x, y                      # cell overhead


@app.cell                            # cell overhead
def _(x, y):                         # cell overhead
    z = x + y                        # cell 2 body
    return                           # cell overhead


if __name__ == "__main__":           # footer
    app.run()                        # footer
```

Key format properties:
- Cells are `@app.cell` decorated functions
- Cell function name is `_` for anonymous cells, or a descriptive name
- Function parameters declare dependencies on other cells' variables
- `return x, y` declares which variables the cell exports
- 2 blank lines between top-level definitions (PEP 8, enforced by marimo)
- `marimo check --fix` auto-generates correct `return` and `def _(params)`
- Cell order in the file does NOT affect execution order (DAG-based)
- Markdown cells use `mo.md(r"""...""")` inside a regular `@app.cell`

## 1:1 Line Mapping via Folds

This is the core architectural insight that makes LSP integration simple.

### The Problem

The virtual buffer hides boilerplate, so it has fewer *visible* lines than the
real file. LSP operates on buffer line numbers. If line numbers don't match,
every LSP request/response needs position translation — a complex, bug-prone
mapping layer.

### The Solution

The virtual buffer has the **same number of buffer lines** as the real file.
Boilerplate lines become placeholder lines (empty or marker text) that are
**folded** into closed folds. Folds are purely visual — they compress multiple
buffer lines into one display line without changing buffer line numbers.

```
Buffer    Hidden buffer (real)         Virtual buffer (display)
line
 1        import marimo                ┈ marimo - http://127.0.0.1:2718 ┈┈
 2        (blank)                      ┈ cell 1 ┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈
 3        __generated_with = "0.20.4"  (folded into line above)
 4        app = marimo.App()           (folded into line above)
 5        (blank)                      (folded into line above)
 6        (blank)                      (folded into line above)
 7        @app.cell                    (folded into line above)
 8        def _():                     (folded into line above)
 9            x = 1                    x = 1
10            y = 2                    y = 2
11            return x, y             ┈ cell 2 ┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈
12        (blank)                      (folded into line above)
13        (blank)                      (folded into line above)
14        @app.cell                    (folded into line above)
15        def _(x, y):                 (folded into line above)
16            z = x + y                z = x + y
17            return                  ┈ cell 3 (md) ┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈
18        (blank)                      (folded into line above)
19        (blank)                      (folded into line above)
20        @app.cell                    (folded into line above)
21        def _(x, y):                 (folded into line above)
22            mo.md(r"""               (folded into line above)
23            # This is a title        # This is a title
24            """)                     (folded into line above)
25            return                  ┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈
26        (blank)                      (folded into line above)
27        (blank)                      (folded into line above)
28        if __name__ == "__main__":   (folded into line above)
29            app.run()                (folded into line above)
```

Result: buffer line 9 in the virtual buffer corresponds to buffer line 9 in
the hidden buffer. LSP completion at virtual `(9, col)` maps to real
`(9, col + 4)`. The only translation needed is a **column offset of ±4** for
cell body lines (to account for the function body indentation).

### LSP Position Mapping

| Direction | Line | Column |
|-----------|------|--------|
| Virtual → Hidden (requests: completion, hover, goto-def) | Identity | `col + 4` for cell body lines, `col` otherwise |
| Hidden → Virtual (responses: diagnostics, locations) | Identity | `col - 4` for cell body lines, `col` otherwise |

This is dramatically simpler than a full line+column mapping table.

### Fold Configuration

```lua
-- Buffer-local window options
vim.wo.foldmethod = "manual"          -- folds created programmatically
vim.wo.foldtext = "v:lua.require('marimo').foldtext()"
vim.wo.foldopen = ""                  -- no command auto-opens folds
vim.wo.foldclose = "all"              -- re-close if somehow opened
vim.wo.foldenable = true
vim.wo.foldminlines = 0               -- allow folding even 1-line regions
```

Fold regions are created with `vim.api.nvim_buf_set_extmark` or
`vim.fn.foldclosed`/`vim.cmd("N,Mfold")` for each separator region.

Remap `zo`, `zO`, `zr`, `zR` to noop in the virtual buffer to prevent the
user from manually opening folds.

### Fold Visual Rendering

Custom `foldtext` function renders different content based on fold position:

- **Header fold** (first fold): `┈ marimo ─ <URL or "not running"> ┈┈┈`
- **Inter-cell folds**: `┈ cell N [name] (md if markdown cell)┈┈┈┈┈┈┈┈`
- **Footer fold** (last fold): `┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈`

The header fold displays the marimo server URL once `:MarimoStart` is run,
or "not running" otherwise.

### Variable Separator Height

The number of lines folded between cells varies because:
- Return statements can be 0 lines (no exports) or 1+ lines
- Blank lines between cells are usually 2 (PEP 8) but may vary

This is a non-issue with folds: whether a separator region is 4 or 6 lines,
it always displays as 1 fold line. The visual result is uniform.

### Edit Protection

Separator lines (inside folds) must be protected from accidental edits.

**Navigation**: Closed folds are treated as single lines by all motion
commands (`j`, `k`, `w`, `b`, `/`, `gg`, mouse click, `:42`). The cursor
naturally skips over folded regions. No custom keybindings needed.

**Edit prevention**: Two mechanisms combined:

1. **`CursorMoved` autocmd** with direction-aware snapping:
   ```lua
   nvim_create_autocmd({"CursorMoved", "CursorMovedI"}, {
       buffer = vbuf,
       callback = function()
           local line = nvim_win_get_cursor(0)[1]
           if vim.fn.foldclosed(line) ~= -1 then
               vim.bo[vbuf].modifiable = false
               -- snap to nearest cell body based on movement direction
               local target = find_nearest_cell_body(line, direction)
               vim.api.nvim_win_set_cursor(0, {target, 0})
           else
               vim.bo[vbuf].modifiable = true
           end
       end
   })
   ```

2. **`nomodifiable` toggle**: Buffer is `nomodifiable` when cursor is on a
   fold line, `modifiable` when on a cell body line. This prevents destructive
   operations (`dd`, `dj`, `cc`) from crossing into separator regions.

Direction awareness is achieved by tracking the previous cursor line:
- `current > prev` → moving down → snap to first line after fold
- `current < prev` → moving up → snap to last line before fold

## File Detection

Two-stage detection on `BufReadPost *.py`:

### Stage 1: Fast Gate (string match)

Check if the first line of the buffer is `import marimo`. This is a simple
string comparison on an already-loaded buffer line — effectively instant.

If the first line doesn't match, the file is not a marimo notebook. Stop.

### Stage 2: Treesitter Confirmation

Parse the buffer with treesitter (Python parser required) and confirm the
presence of `@app.cell` decorated function definitions. This handles edge
cases where a file imports marimo but isn't a notebook.

Treesitter query (approximate):
```scheme
(decorated_definition
  (decorator (attribute
    object: (identifier) @obj
    attribute: (identifier) @attr))
  (#eq? @obj "app")
  (#eq? @attr "cell"))
```

If confirmed, activate marimo-nvim for this buffer.

### Requirements

- Neovim >= 0.10 (for `vim.system()`, modern `vim.health`)
- treesitter Python parser installed (`nvim-treesitter` with `python` grammar)

## Edit Sync and Write-Back

### Virtual → Hidden Buffer Sync (on every edit)

When the user edits cell body lines in the virtual buffer:

1. `nvim_buf_attach` `on_lines` callback fires
2. Identify which cell was modified (from line number → cell mapping)
3. Translate the change: add 4-space indent to each modified line
4. Apply the indented change to the corresponding lines in the hidden buffer
5. Debounce: coalesce rapid edits, sync within 100-500ms

### Hidden Buffer → Disk (on save)

When the user runs `:w` on the virtual buffer:

1. Write the hidden buffer content to disk (the real `.py` file)
2. Run `marimo check --fix <file>` as a post-save hook
3. Re-read the fixed file into the hidden buffer
4. Re-parse cells (return statements and function signatures may have changed)
5. Re-render the virtual buffer: update cell body lines (dedented), recreate
   folds for separator regions

`marimo check --fix` handles:
- Generating correct `return x, y` from variable analysis
- Fixing function parameters `def _(x, y):` for dependencies
- Normalizing formatting and `__generated_with` version
- Removing exports for variables no downstream cell uses

### Marimo Watch Picks Up Changes

After the file is written and fixed on disk, `marimo edit --watch` detects
the change and syncs it to the browser for rendering. The user sees updated
output in the browser.

**Important**: The marimo `--watch` mode warns "Enabling watch mode may
interfere with auto-save." The plugin should only write to disk on explicit
`:w`, never on `TextChanged` or similar auto-save events.

## Marimo Server Management

### :MarimoStart

Launches the marimo watch server for the current notebook:

```
marimo edit --watch --headless --no-token <file.py>
```

- `--watch`: monitor file for external changes
- `--headless`: don't auto-open browser
- `--no-token`: no authentication (local dev)

The process is spawned via `vim.system()`. Stdout is captured and parsed for
the `URL: http://127.0.0.1:XXXX` line. The URL is stored and displayed in the
header fold's `foldtext`.

Port selection is delegated to marimo (defaults to 2718, auto-increments on
conflict). No admin/root privileges required.

### :MarimoStop

Kills the marimo server process via the stored process handle. nvim should take
care of stopping the server when nvim is closed

### :MarimoOpen

Opens the marimo URL in the system browser (convenience command).

### Server Lifecycle

- Server process handle stored as module-level variable
- `vim.schedule()` used in stdout/stderr callbacks for safe API access
- On `BufUnload` / `VimLeavePre`: auto-stop the server if running
- Health check: `:checkhealth marimo` verifies `marimo` is installed and
  reachable via `uvx` or direct `marimo` command

## Commands and Keybindings

### Commands

| Command | Description |
|---------|-------------|
| `:MarimoStart` | Launch marimo watch server for current file |
| `:MarimoStop` | Stop the marimo watch server |
| `:MarimoOpen` | Open marimo URL in system browser |
| `:MarimoNextCell` | Move cursor to the top of the next cell body |
| `:MarimoPrevCell` | Move cursor to the top of the previous cell body |
| `:MarimoAddCellBelow` | Insert new empty cell below current cell and move cursor into new cell |
| `:MarimoAddCellAbove` | Insert new empty cell above current cell and move cursor into new cell |
| `:MarimoDeleteCell` | Delete current cell (with confirmation prompt) |
| `:MarimoMoveUp` | Move current cell up (swap with previous cell) |
| `:MarimoMoveDown` | Move current cell down (swap with next cell) |
| `:MarimoToggleMarkdown` | Toggle current cell between Python and markdown |

### Default Keybindings (buffer-local, set on activation)

```lua
-- Navigation
vim.keymap.set("n", "]c", "<cmd>MarimoNextCell<CR>",    { buffer = true, desc = "Next marimo cell" })
vim.keymap.set("n", "[c", "<cmd>MarimoPrevCell<CR>",    { buffer = true, desc = "Previous marimo cell" })
vim.keymap.set("n", "<M-c>", "<cmd>MarimoNextCell<CR>",    { buffer = true, desc = "Next marimo cell" })
vim.keymap.set("n", "<M-C>", "<cmd>MarimoPrevCell<CR>",    { buffer = true, desc = "Previous marimo cell" })

-- Cell operations
vim.keymap.set("n", "<leader>ma", "<cmd>MarimoAddCellBelow<CR>",    { buffer = true, desc = "Add cell below" })
vim.keymap.set("n", "<leader>mA", "<cmd>MarimoAddCellAbove<CR>",    { buffer = true, desc = "Add cell above" })
vim.keymap.set("n", "<leader>md", "<cmd>MarimoDeleteCell<CR>",      { buffer = true, desc = "Delete cell" })
vim.keymap.set("n", "<leader>mj", "<cmd>MarimoMoveDown<CR>",        { buffer = true, desc = "Move cell down" })
vim.keymap.set("n", "<leader>mk", "<cmd>MarimoMoveUp<CR>",          { buffer = true, desc = "Move cell up" })
vim.keymap.set("n", "<leader>mm", "<cmd>MarimoToggleMarkdown<CR>",  { buffer = true, desc = "Toggle markdown" })

-- Server
vim.keymap.set("n", "<leader>ms", "<cmd>MarimoStart<CR>", { buffer = true, desc = "Start marimo server" })
vim.keymap.set("n", "<leader>mS", "<cmd>MarimoStop<CR>",  { buffer = true, desc = "Stop marimo server" })
vim.keymap.set("n", "<leader>mo", "<cmd>MarimoOpen<CR>",  { buffer = true, desc = "Open in browser" })
```

All keybindings are configurable via `setup()` opts. Users can disable
defaults and define their own.

## Cell Operations — Implementation Details

### Cell Data Structure

Each cell is tracked with:

```lua
---@class MarimoCell
---@field index number          -- cell order (1-based)
---@field name string|nil       -- function name if not "_"
---@field type "python"|"markdown"
---@field body_start number     -- first body line in hidden buffer
---@field body_end number       -- last body line in hidden buffer
---@field region_start number   -- first line of entire cell block (decorator)
---@field region_end number     -- last line of entire cell block (return)
---@field fold_before_start number  -- first fold line before this cell's body
---@field fold_before_end number    -- last fold line before this cell's body
```

### :MarimoAddCellBelow / :MarimoAddCellAbove

1. Determine the current cell from cursor position
2. In the hidden buffer, insert a new cell block at the appropriate position:
   ```python
   \n\n@app.cell\ndef _():\n    pass\n
   ```
3. In the virtual buffer, insert corresponding lines:
   - Placeholder lines for the new separator/overhead (fold region)
   - A single body line with `pass` (editable), or if possible keep the cell empty and show ghost text `empty cell`
4. Create fold for the new separator region
5. Adjust all subsequent cell tracking data (line offsets shift)
6. Write to disk and run `marimo check --fix`

### :MarimoDeleteCell

1. Identify current cell from cursor position
2. Prompt for confirmation: `Delete cell N? (y/n)`
3. In the hidden buffer, remove the entire cell region (decorator through
   return, plus trailing blank lines)
4. In the virtual buffer, remove corresponding lines (body + separator)
5. Remove the fold for this cell's separator
6. Adjust all subsequent cell tracking data
7. Write to disk and run `marimo check --fix`

### :MarimoMoveUp / :MarimoMoveDown

1. Identify current cell and target cell (above or below)
2. In the hidden buffer, swap the two entire cell blocks (region_start through
   region_end, including surrounding blank lines)
3. In the virtual buffer, swap the corresponding line ranges
4. Recreate folds (positions change after swap)
5. Move cursor to the new position of the moved cell
6. Write to disk and run `marimo check --fix`

Note: cell order in the file doesn't affect marimo's execution order (it's
DAG-based), so moving cells is purely organizational/visual.

### :MarimoToggleMarkdown

Toggles the current cell between Python and markdown. Content is preserved,
only the wrapping changes and adapting folding range to hide the mo.md wrapper.

**Python → Markdown**:
```python
# Before (cell body in virtual buffer):
This is a title

# After (cell body in virtual buffer) identical:
This is a title

# Before (cell in hidden buffer):
def _():
    This is a title

# After (cell in hidden buffer):
def _():
    mo.md(r"""
    This is a title
    """)
```


**Markdown → Python**:
```python
# After (cell body in virtual buffer) identical:
This is a title

# Before (cell body in virtual buffer):
This is a title

# Before (cell in hidden buffer):
def _():
  mo.md(r"""
    This is a title
    """)

# After (cell in hidden buffer):
def _():
    This is a title

```

The cell's `type` field is toggled, if the type is update foldtext with type.
Write to disk and run `marimo check --fix`.

### :MarimoNextCell / :MarimoPrevCell

Move cursor to the first body line of the next/previous cell. Since folds
already make `j`/`k` skip separators, these commands provide larger jumps:

- `:MarimoNextCell`: cursor → first line of next cell body
- `:MarimoPrevCell`: cursor → first line of previous cell body

## LSP Integration Details

### Attachment

When the hidden buffer is created, LSP clients are attached to it. Since the
hidden buffer contains a valid `.py` file, LSP attachment is automatic via
Neovim's built-in LSP mechanisms (filetype detection, lspconfig, etc.).

The virtual buffer does NOT have LSP attached directly. Instead, the plugin
intercepts LSP-related actions in the virtual buffer and proxies them. Syntax
highlighting should be consistent with hidden buffer.

### Completion

When the user triggers completion in the virtual buffer:

1. Capture cursor position `(vline, vcol)` in virtual buffer
2. Translate to hidden buffer position: `(vline, vcol + 4)` for cell body
3. Request completion from LSP at the hidden buffer position
4. Translate completion results back: adjust any position fields by `col - 4`
5. Present results to the user via `nvim-cmp`, built-in completion, etc.

### Diagnostics

LSP publishes diagnostics for the hidden buffer. The plugin:

1. Listens for `vim.diagnostic` updates on the hidden buffer
2. Filters diagnostics to only those on cell body lines
3. Translates positions: `col - 4` for cell body lines
4. Publishes translated diagnostics on the virtual buffer via
   `vim.diagnostic.set()`

Diagnostics on boilerplate lines (decorator, def, return) are suppressed —
the user can't see or edit those lines.

### Hover and Go-to-Definition

Same pattern as completion:

1. Translate cursor position from virtual → hidden (`col + 4`)
2. Make LSP request on hidden buffer
3. If the response contains locations in the same file, translate back
   (`col - 4`)
4. If the response points to a different file, no translation needed

### Excluded from MVP

- **Rename/Refactor**: Requires applying edits across the hidden buffer and
  reflecting them in the virtual buffer. Complex interaction with fold
  regions. Deferred.
- **Code Actions**: Same complexity as rename. Deferred.

## Plugin Structure

```
marimo-nvim/
├── plugin/marimo.lua              -- Load guard, BufReadPost autocmd
├── lua/marimo/
│   ├── init.lua                   -- setup(), config, module table
│   ├── detect.lua                 -- Two-stage marimo file detection
│   ├── buffer.lua                 -- Hidden/virtual buffer creation and sync
│   ├── cells.lua                  -- Cell parsing, data structures, operations
│   ├── folds.lua                  -- Fold creation, foldtext rendering
│   ├── lsp.lua                    -- LSP proxy (completion, diagnostics, hover, goto-def)
│   ├── server.lua                 -- Marimo server lifecycle (start/stop)
│   ├── commands.lua               -- Command and keybinding registration
│   └── health.lua                 -- :checkhealth support
├── doc/marimo.txt                 -- Vimdoc
├── tests/                         -- Tests (plenary.busted or mini.test)
├── .gitignore
└── LICENSE
```

### Module Responsibilities

**`detect.lua`**: Exports a function that takes a buffer number and returns
`true` if the buffer contains a marimo notebook. Stage 1: string match on
line 1. Stage 2: treesitter query for `@app.cell`.

**`buffer.lua`**: Creates the hidden buffer (clone of real file), creates the
virtual buffer (dedented cell bodies + placeholder lines), sets up
`nvim_buf_attach` for edit sync, handles the write-back flow (save → check
--fix → re-read → re-render).

**`cells.lua`**: Parses the hidden buffer with treesitter to extract cell
boundaries. Maintains the cell list. Implements add/delete/move/toggle
operations. Provides lookup: given a buffer line, which cell is it in?

**`folds.lua`**: Creates fold regions for all separator zones. Provides the
`foldtext` function. Handles fold recreation after cell operations.

**`lsp.lua`**: Intercepts LSP requests/responses and translates positions
between virtual and hidden buffers. Manages diagnostic mirroring.

**`server.lua`**: Spawns `marimo edit --watch --headless --no-token`, parses
URL from stdout, stores process handle, provides start/stop/status.

**`commands.lua`**: Registers all user commands and default keybindings.
Commands call into other modules lazily.

## Configuration

```lua
require("marimo").setup({
    -- Detection
    auto_detect = true,  -- auto-activate on BufReadPost *.py

    -- Server
    marimo_command = "uvx marimo",  -- or "marimo" if installed globally
    server_args = {},               -- additional args for marimo edit

    -- Keybindings
    keybindings = {
        enabled = true,             -- set false to define your own
        next_cell = "]c",
        prev_cell = "[c",
        add_below = "<leader>ma",
        add_above = "<leader>mA",
        delete_cell = "<leader>md",
        move_down = "<leader>mj",
        move_up = "<leader>mk",
        toggle_markdown = "<leader>mm",
        start_server = "<leader>ms",
        stop_server = "<leader>mS",
        open_browser = "<leader>mo",
    },

    -- Appearance
    separator_char = "┈",
    show_cell_index = true,
    show_cell_name = true,
})
```

## MVP Scope

### In MVP

1. Detection: two-stage on `BufReadPost *.py`
2. Virtual buffer rendering with folds
3. Edit sync: virtual → hidden buffer (debounced, < 500ms)
4. Write-back: save → `marimo check --fix` → re-render
5. LSP: completion, diagnostics, hover, go-to-definition
6. Cell navigation: `:MarimoNextCell`, `:MarimoPrevCell`
7. Cell operations: add, delete, move, toggle markdown
8. Server management: `:MarimoStart`, `:MarimoStop`, `:MarimoOpen`
9. Health check: `:checkhealth marimo`
10. Default keybindings (configurable)

### Post-MVP

- **Cell mode**: A dedicated modal editing mode for cells, entered via `<Esc>`
  from normal mode. See Appendix A.
- **LSP rename/refactor**: Position-aware edit application across buffers.
- **Cell status indicators**: Show execution state (stale, running, error) in
  fold text if marimo exposes this via file-based mechanism.
- **Multi-file support**: Handle marimo projects with multiple notebooks.
- **Undo across re-renders**: Preserve undo history when `marimo check --fix`
  triggers a re-render.

## Appendix A: Cell Mode (Future Feature)

A special modal editing mode where Neovim operates at the cell level rather
than the line/character level. Entered by pressing `<Esc>` from normal mode
in a marimo buffer. Since navigation and operation act on whole cells, some
sort of cell highlighting should be applied to show which cell is currently
'selected'.

### Behavior in Cell Mode

| Key | Action |
|-----|--------|
| `j` / `k` / `<up>` / `<down>`  | Move to next/previous cell |
| `dd` | Delete current cell |
| `yy` | Yank current cell |
| `p` | Paste cell below |
| `P` | Paste cell above |
| `o` | Add new cell below and enter insert mode |
| `O` | Add new cell above and enter insert mode |
| `<Alt-j>` | Move current cell down |
| `<Alt-k>` | Move current cell up |
| `i` / `<Enter>` | Exit cell mode, enter normal mode in cell body |
| `m` | Toggle markdown/python |

### Implementation Approach

Recommended: **submode.nvim** (requires Neovim >= 0.10)

- Creates a true submode with its own keymap set
- Visual indicator: statusline shows `CELL` mode
- Clean entry/exit with proper mode lifecycle
- All cell mode keybindings map to the same `:Marimo*` commands used
  in normal mode, so no duplicate logic

The commands (`:MarimoDeleteCell`, `:MarimoMoveUp`, etc.) are designed so
they can be wired to cell mode keybindings without any changes to the core
command logic. Cell mode is purely a keybinding layer on top of existing
commands.

## Appendix B: Verified Assumptions

These assumptions were tested during design:

1. **`marimo check --fix` fixes return statements**: Confirmed. Tested with
   empty returns (`return ()`), missing returns, and wrong returns. All were
   corrected to proper `return x, y` with correct function signatures.

2. **`marimo check --fix` fixes function parameters**: Confirmed. `def _()`
   is updated to `def _(x, y)` when the cell references variables from other
   cells.

3. **`__marimo__` directory is unreliable for detection**: Confirmed. Only
   created when notebooks use persistent caching, session export, or gallery
   features. Not created by `marimo edit`, `marimo run`, or `marimo check`.

4. **Port selection needs no admin**: Confirmed. Marimo defaults to port 2718
   and auto-increments on conflict. No root/admin needed for ports > 1024.

5. **`marimo edit --watch` works with file-only sync**: Confirmed. Changes
   written to disk are picked up by marimo's file watcher.

6. **Marimo cell order doesn't affect execution**: Confirmed via documentation.
   Execution order is determined by the dependency DAG, not file order.

## Appendix C: Risk Register

| Risk | Impact | Mitigation |
|------|--------|------------|
| `marimo check --fix` changes line count | Breaks 1:1 mapping until re-render | Re-parse and re-render entire virtual buffer after each fix (only on save) |
| User edits during re-render | Race condition between edit and re-render | Lock buffer as `nomodifiable` during re-render, unlock after |
| LSP diagnostics on boilerplate lines | Confusing diagnostics on invisible lines | Filter diagnostics to cell body lines only |
| treesitter parse failure | Detection or cell parsing fails | Fallback to regex-based cell detection; notify user |
| Marimo format changes in future versions | Cell parsing breaks | Pin expected format patterns; health check warns on version mismatch |
| Large notebooks (100+ cells) | Slow re-parse/re-render on save | Profile and optimize; consider incremental updates post-MVP |
| Fold z-commands leak through | User opens a fold, sees placeholder lines | `foldopen=""` + remap `zo`/`zO`/`zr`/`zR` to noop in buffer |
