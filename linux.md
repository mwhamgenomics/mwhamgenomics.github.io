---
layout: page
title: Linux 
permalink: /linux
---

As a Linux user, I have and probably always will have a lot to learn about the systems I use - definitely
something worth posting about, but it didn't seem right to put this in a normal post since I'll be
updating this over time. Here are some compiled notes on some of the main things I use working with Linux.

Last updated: 9th Mar 19

## Bash
Bash is my current shell of choice. I'm not hardcore enough to use zsh yet.

| Keystroke              | Action                                                                             |
|------------------------|------------------------------------------------------------------------------------|
| `ctrl-k`               | truncate the line at the cursor                                                    |
| `ctrl-r`               | enter incremental history search. `ctrl` + `r`/`s` searches backwards and forwards |
| `alt` + `right`/`left` | jump the cursor over words                                                         |
| `ctrl` + `a`/`e`       | jump to the start or end of the line                                               |
{:.table}

In macOS, `ctrl-s` might not work. This seems to have been turned off as a safety measure against hitting `ctrl-s`
outside of the search (which enters XOFF mode, cutting off stdin from the terminal - the remedy being `ctrl-q`).
To allow `ctrl-s` to work, run `stty -ixon -ixoff`.


## Vi
The more you put into Vi/Vim, the more you get out. If you just use its basic features with no customisation,
there's no reason to use it over nano or gedit, but its more advanced features make it a viable dev environment
for headless Linux.

### Windows
From [Vi's docs](), a window is a view on a buffer, which is a bit of text loaded into memory. You can split
your view up into windows on startup with `-o` or `-O` to split horizontally or vertically, or after Vi has
started:

| Keystroke               | Action                                                              |
|-------------------------|---------------------------------------------------------------------|
| `:new/:vnew`            | split the window horizontally or vertically with a new empty buffer |
| `:split/:vsplit [file]` | same, except with the current buffer or `[file]`                    |
{:.table}

Once multiple windows are open:

| Keystroke             | Action                                               |
|-----------------------|------------------------------------------------------|
| `ctrl-w <direction>`  | navigate the cursor between windows                  |
| `ctrl-w K/J/H/L`      | move the current window to the top/bottom/left/right |
| `ctrl-w +/-`          | in/decrement the current window height               |
| `ctrl-w </>`          | in/decrement the current window width                |
| `[vertical] resize n` | change the current window size to `n`                |
| `:only`               | close all other windows                              |
| `:ls`                 | list all buffers                                     |
{:.table}

### Tabs
A tab is a collection of windows.

- `:tabnew` to open a new tab with an empty buffer
- `:tabedit <file>` to edit a file in a new tab
- `gt` and `gT` to move between tabs
- `vi -p <file1> <file2> ...` to open multiple files, each in their own tab

### Diff mode
I didn't know Vi could do this at first, but it's pretty cool.

- `vi -d <file1> <file2> ...` to open the given files in diff mode, where differences between buffers are highlighted
- `:diffsplit <file>` to open a new file, comparing with the current buffer
- `:vertical diffsplit <file>` to do the above but with a vertical split
- `:set scrollbind` locks the scrolling between diff-ed buffers

### Multiline editing
Vi doesn't support multiple cursors, but does have features for multiline editing:

- Select the lines to edit with Visual Block (`ctrl-v`). From here, you can:
    - insert text with `I`, then entering some text (it looks like you're only editing one line), then `esc` to exit insert mode and edit all lines.
    - append the end of the line with `A`, then proceeding as above for inserting text
    - delete text the same way as in other modes

### Searching
You might already be familiar with Vi's search function (`/` or `?`), but here are some things you can do once
it's found a match:

| Keystroke | Action                |
|-----------|-----------------------|
| `n`       | search forwards       |
| `N`       | search backwards      |
| `ggn`     | go to the first match |
| `GN`      | go to the last match  |
{:.table}

You can also append the search term with `\c` to make the search case insensitive, or `\C` for case sensitive.
There are also a couple of search shortcuts:

| Keystroke | Action                                                                 |
|-----------|------------------------------------------------------------------------|
| `*`/`#`   | search forwards/backwards for occurrences of the word the cursor is on |
| `g*`/`g#` | same, except search for any occurrence, not just the exact word        |
{:.table}

autoindent

### Solarized
Solarized is a Vi colour palette available at https://github.com/altercation/vim-colors-solarized, and it's
much nicer than the default colours.

### Sessions
- `mksession <file>` saves the current Vi session (windows, tabs, etc.) to a session file
- `vi -S <file>` loads a session file

### Copy/paste
The system-wide clipboard won't work properly if you've got line numbers on or have a window or tmux pane off to
the right hand side. Vi's visual selection, however, does work.

With some text selected (with visual, visual line, or visual block), `d` deletes the text and `y` ('yank')
copies it. Then `p` or `P` pastes it after or before the cursor.

### Marks
Marks are what Vi calls bookmarks. Each mark is identified by a single char - if it's a capital letter, it's
global across all files.

| Keystroke                 | Action                                                |
|---------------------------|-------------------------------------------------------|
| `m<char>`                 | create a mark                                         |
| `'<char>` or `` `<char>`` | jump to a mark                                        |
| `:marks`                  | list all marks                                        |
| `:delmarks <chars>`       | delete marks - can also multi-delete with, e.g, 'a-d' |
{:.table}

### Other random stuff

| Keystroke            | Action                                                                                        |
|----------------------|-----------------------------------------------------------------------------------------------|
| `:changes`           | view all of the file's current changes                                                        |
| `J`                  | join lines - like deleting the `\n` at the end of the line, except the cursor can be anywhere |
| `set <settingname>?` | show the value of a setting                                                                   |
| `set <settingname>&` | reset a setting to its default                                                                |
{:.table}

### netrw
Netrw is the default file system explorer for Vi.

| Keystroke          | Action                           |
|--------------------|----------------------------------|
| `:Explore`         | open netrw in the current window |
| `:Sexplore` (heh.) | split and open netrw             |
| `:Vexplore`        | vertically split and open netrw  |
{:.table}

In netrw:

- `return` enters a directory or opens a file in the current window
  - the way it opens files is controlled by the variable `g:netrw_browse_split`
  - 1 opens in a new split
  - 2 opens in a vsplit
  - 3 opens in a new tab
  - 4 opens in the previous window

- `I` turns off the header
- `i` cycles through view types:
  - 0: one file per line
  - 1: `ls -l`-style view
  - 2: `ls`-style view
  - 3: tree view
  - set `g:netrw_liststyle` in the vimrc to set a default view type
  - set `g:netrw_winsize` to set a default (percentage) window size for netrw


## tmux
Standing for Terminal MUltipleXer, tmux is kind of like GNU screen with some extra stuff, including window
splitting and scripted sessions. A tmux session contains multiple windows, which in turn contains multiple
panes - in this case a tmux window is like a Vi tab, and a tmux pane is a Vi window.

Once a tmux session is open:

| Keystroke               | Action                             |
|-------------------------|------------------------------------|
| `ctrl-b :`              | enter the tmux command line        |
| `ctrl-b` + `(` or `)`   | switch between sessions            |
| `ctrl-b d`              | detach the client                  |
| `ctrl-b c`              | create a new window                |
| `ctrl-b &`              | close the current window           |
| `ctrl-b` + `n` or `p`   | go to previous/next window         |
| `ctrl-b` + `'` or `0-9` | go to a window                     |
| `ctrl-b <arrow>`        | navigate between panes             |
| `ctrl-b ctrl-o`         | rotate all panes around the window |
| `ctrl-b ctrl-<arrow>`   | resize the pane in increments of 1 |
| `ctrl-b alt-<arrow>`    | resize the pane in increments of 5 |
{:.table}

One thing I currently dislike about tmux is the number of keystroke-based commands, which I often find
unintuitive. For certain features, I prefer string commands input via `ctrl-b :`:

| Command                                  | Action                                                                           |
|------------------------------------------|----------------------------------------------------------------------------------|
| `rename-session <name>`                  | rename the session                                                               |
| `split-window`/`splitw`                  | split the window into two panes - `-v` or `-h` for vertical/horizontal splits    |
| `rotate-window`                          | rotate all panes around the window                                               |
| `resize-pane (-UDLR) <value>`            | resize the pane up/down/left/right by the given value                            |
| `resize-pane (-xy) <value>`              | resize the pane to the given x/y value                                           |
| `break-pane`                             | split off the current pane into a new window                                     |               
| `capture-pane -S <fromline> -E <toline>` | capture the contents of the pane into a buffer                                   |
| `list-buffers`                           | list all paste buffers                                                           |
| `delete-buffer -b <buffer>`              | delete a buffer                                                                  |
| `paste-buffer`                           | enter the most recent buffer (or any buffer name with `-b` into the current pane |
| `save-buffer <file>`                     | write the most recent buffer (or any buffer name with `-b` to `<file>`           |

  

When I tried using Vi in tmux, I found that tmux was eating my colour scheme. This was fixed by adding
`set -g default-terminal "screen-256color"` to my `tmux.conf`. I also ran into some problems with resizing windows
on macOS because `ctrl-<arrow>` is a system command to switch between spaces. I got around this by:

- unbinding the `ctrl-<arrow>` commands
- re-binding each `alt-<arrow>` command to what `ctrl-<arrow>` was, since they're pretty similar anyway
- modifying the `alt-<arrow>` keyboard actions in Terminal settings:
  - an `alt-<arrow>` sends a sequence of `\033\033[` plus A, B, D and C for up, down, left and right respectively  

(Or you can just use string commands.)
  
There's also a solarized-style tmux colour scheme available at https://github.com/seebi/tmux-colors-solarized.


TODO: screen, less, https://shapeshed.com/vim-netrw


