# tertius.vim

Utilities to streamline a **Git feature-branch workflow** with a little LLM help.
Tertius opens small, throwaway scratch buffers where you can draft a **user story**, **commit message**, **pull-request description**, **code review summary**, or a **todo list**, then calls your LLM to generate or refine the text based on real repository context (commits, diffs). It also includes a helper to **bootstrap a feature branch** from a drafted user story.

> Requires `git` and `curl`, plus an OpenAI-compatible API (key via `OPENAI_API_KEY`).

---

## Features

* **LLM-assisted texts**:
  * Commit message generator (analyzes branch commits & diff).
  * User story composer.
  * Pull-request description builder.
  * Code-review outline on the feature branch.
  * Developer todo list (xit format).

* **Feature-branch bootstrapper**:
  * From a drafted user story buffer, Tertius can:

    * `git add .`
    * `git stash`
    * `git fetch --all`
    * checkout default branch (configurable)
    * create/switch to `-B <sanitized-branch-name>`
    * make an (allow-empty) initial commit from the buffer contents
  * Branch name is derived from the **first line** of your user-story buffer, lowercased and slugified (`a–z, 0–9, -`).

---

## Installation

Use any Vim plugin manager; the plugin is a single Vimscript file.

### vim-plug

```vim
call plug#begin('~/.vim/plugged')
Plug 'placek/tertius-vim'
call plug#end()
```

### packer.nvim (Neovim, Lua)

```lua
use { 'placek/tertius-vim' }
```

### lazy.nvim

```lua
{ 'placek/tertius-vim' }
```

### Pathogen

```bash
cd ~/.vim/bundle
git clone https://github.com/placek/tertius-vim.git
```

---

## Configuration

All knobs live in `g:tertius_config` (defaults shown):

```vim
let g:tertius_config = {
\ 'gitExec': 'git',
\ 'curlExec': 'curl',
\ 'defaultBranch': 'origin/master',
\ 'llmBaseUrl': 'https://api.openai.com/v1',
\ 'llmModel': 'gpt-4o',
\ 'userStoryIdPattern': '\[\([^\]]\+\)\]', " capture [TICKET-123] etc.
\}
```

Environment variables:

* `OPENAI_API_KEY` – **required**.
* `OPENAI_BASE_URL` – optional override for `g:tertius_config.llmBaseUrl`.
* `OPENAI_MODEL` – optional override for `g:tertius_config.llmModel`.

> If `git`/`curl` aren’t found, the plugin will echo an error.

Git default branch is used when bootstrapping a feature branch from a user story. Such a branch can be configured with:

```sh
git config --global core.default origin/main
```

If this option is not set, Tertius defaults to `g:tertius_config.defaultBranch`.


The `userStoryIdPattern` is used to extract the user story ID from the first line of the user story buffer, e.g., `[PROJ-42]` or `[TICKET-123]`.

---

## Commands & Functions

Tertius exposes Vim **functions** (callable via `:call`) and ships minimal mappings/autocmds you can replace with your own. Functions read the **current buffer** as input (unless noted).

### Core generator

* `:call Tertius('commit_message', getline(1, '$'))`
  Low-level entrypoint used by helpers below. Sends a system prompt (from config) and the provided content to the LLM, optionally invoking built-in “tools” to:

  * list commits on the current branch since it diverged from the default branch
  * fetch a specific commit’s message and diff

### High-level helpers

* `:call TertiusCommitMessage()`
  Generate a **commit message** for the staged changes + optional text in your `gitcommit` buffer.

* `:call TertiusOpenUserStoryWindow()`
  Open a scratch buffer (`/tmp/tertius_user_story`) to draft a **user story**.

* `:call TertiusUserStory()`
  Ask the LLM to **compose/improve** the user story using your current buffer content.

* `:call TertiusOpenPullRequestWindow()`
  Open a scratch buffer for a **pull-request description**.

* `:call TertiusPullRequest()`
  Generate the **PR description** from recent commit messages/context.

* `:call TertiusOpenCodeReviewWindow()`
  Open a scratch buffer and immediately populate a **code-review** outline for the current feature branch.

* `:call TertiusTodoList()`
  Generate a developer **todo list** in xit format from the current buffer context.

### Feature-branch bootstrap (auto)

When you close (`BufUnload`) the **user story** buffer (`/tmp/tertius_user_story`), Tertius:

1. Slugifies the first line into a branch name, e.g.
   “Add OAuth login \[PROJ-42]” → `add-oauth-login-proj-42`
2. Runs: `git add .`, `git stash`, `git fetch --all`,
   `git checkout <defaultBranch>`, `git checkout -B <branch>`,
   and makes an **allow-empty** commit with the buffer text as the message.

This gives you a ready feature branch with initial context committed.

---

## Default filetype autocmds

Tertius defines a few buffer-local conveniences:

* On `FileType gitcommit`: create a normal-mode mapping that calls `TertiusCommitMessage()`.
* On `FileType tertius_user_story`: create a normal-mode mapping that calls `TertiusUserStory()`.
* On `BufUnload /tmp/tertius_user_story`: run feature-branch bootstrap (above).

> **Note:** The stock mappings are intentionally minimal so you can bind your own keys. See below.

---

## Recommended key mappings

The plugin ships simple `nnoremap` lines; you can (and probably should) add your own in `vimrc/init.vim`. Examples:

```vim
" User story flow
nnoremap <leader>us :call TertiusOpenUserStoryWindow()<CR>
nnoremap <leader>uS :call TertiusUserStory()<CR>

" Pull request
nnoremap <leader>pr :call TertiusOpenPullRequestWindow()<CR>
nnoremap <leader>pR :call TertiusPullRequest()<CR>

" Code review
nnoremap <leader>cr :call TertiusOpenCodeReviewWindow()<CR>

" Commit message (in a gitcommit buffer)
autocmd FileType gitcommit nnoremap <buffer> <leader>cm :call TertiusCommitMessage()<CR>

" Todo list
nnoremap <leader>td :call TertiusTodoList()<CR>
```

---

## How it talks to your LLM

Tertius performs one or more `/chat/completions` calls with:

* the relevant **system prompt** (from `g:tertius_config.prompts.*`)
* your buffer text as **user content**
* optional **function/tool calls** so the model can:

  * list commits on the current branch since its merge-base with the default branch
  * fetch a specific commit’s message/diff

Responses are written back into the current buffer.

---

## Tips & Caveats

* Set `OPENAI_API_KEY` (and optionally `OPENAI_BASE_URL`/`OPENAI_MODEL`) in your shell before launching Vim/Neovim.
* The **branch bootstrap** uses your user-story’s **first line** for the branch name. Keep it slug-friendly.
* If you use a default branch other than `origin/master`, set:

  ```vim
  let g:tertius_config.defaultBranch = 'origin/main'
  ```
* The commit-message helper works best when you’ve staged changes and/or have a diff in the buffer (e.g. via a `gitcommit` buffer).

---

## License

MIT

---

## Acknowledgements / Source

* Implementation details and defaults come from the plugin source: `plugin/tertius.vim`. ([GitHub][1])

---

[1]: https://raw.githubusercontent.com/placek/tertius-vim/refs/heads/master/plugin/tertius.vim "raw.githubusercontent.com"
