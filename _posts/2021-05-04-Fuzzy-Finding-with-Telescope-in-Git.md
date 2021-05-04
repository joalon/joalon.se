---
layout: post
title: "Fuzzy finding in a Git repository using Telescope"
author: "Joakim Lönnegren"
categories: blog
tags: [Neovim, Telescope, Git]
image:
  feature: telescope-git/feature.jpg
  teaser: telescope-git/teaser.jpg
---

Just wanted to share a small snippet for fuzzy finding/grepping in Neovim using Telescope. These two key bindings set the current working directory to the root of the git repo before starting the fuzzy find by using `git rev-parse --show-toplevel`.

```
call plug#begin(expand('~/.config/nvim/plugged'))

Plug 'nvim-lua/plenary.nvim'
Plug 'nvim-telescope/telescope.nvim'

call plug#end()

" Find files using Telescope command-line sugar.
nnoremap <leader>fr <cmd>lua require('telescope.builtin').find_files{ cwd = vim.fn.systemlist("git rev-parse --show-toplevel")[1] }<cr>
nnoremap <leader>gr <cmd>lua require('telescope.builtin').live_grep{ cwd = vim.fn.systemlist("git rev-parse --show-toplevel")[1] }<cr>
```

Pressing `<leader>fr` or `<leader>gr` will find or grep respectively in the whole git repository:

![example](/images/telescope-git/example.gif)
