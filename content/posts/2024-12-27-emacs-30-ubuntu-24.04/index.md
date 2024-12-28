+++
title = "Installing Emacs 30"
author = ["David"]
date = 2024-12-27
tags = ["Emacs"]
url = "emacs-30-ubuntu-24-04"
draft = false
summary = "Installing Emacs 30 from source on Ubuntu 24.04"
+++

## Installing Emacs 30 on Ubuntu 24.04 {#installing-emacs-30-on-ubuntu-24-dot-04}

At the time of writing Emacs 30 is not yet released, but it might be worth upgrading to this version since it has some nice new features. Some highlights are given here; [Emacs wiki Emacs 30 highlights](https://www.emacswiki.org/emacs/EmacsThirtyHighlights), especially useful is that it comes with `elixir-ts-mode` and `heex-ts-mode`, you just need to add the necessary language grammers.


### Install Emacs build deps {#install-emacs-build-deps}

In `Software & Updates` in the first tab `Ubuntu Software` ensure `Source code` is selected.

```bash
sudo apt update
sudo apt build-dep -y emacs
```


### Install Tree-sitter {#install-tree-sitter}

```bash
cd /opt/
sudo mkdir tree-sitter
sudo chown $USER:$USER tree-sitter/
git clone https://github.com/tree-sitter/tree-sitter
cd tree-sitter/
make all
sudo make install

sudo ldconfig
```


### Install Emacs {#install-emacs}

```bash

cd /opt/
sudo mkdir emacs
sudo chown $USER:$USER emacs/
git clone git@github.com:emacs-mirror/emacs.git --single-branch --branch emacs-30
cd emacs/

./autogen.sh
./configure --with-tree-sitter --with-pgtk

# find out the number of processors to use in the next command
nproc

make bootstrap -j4

# quick test before install
src/emacs -Q

sudo make install
```

Note the option `--with-pgtk` builds emacs for "Pure GTK" which is needed for Wayland, now the default on Ubuntu (instead of the X Window System).


### Set up Emacs init {#set-up-emacs-init}

The most interesting part in the `init.el` file is as follows, this assumes you have an Elixir LSP server running at `/opt/elixir-ls/release/`

```elisp

(require 'treesit)
(require 'heex-ts-mode)
(require 'elixir-ts-mode)

;; elixir-mode, elixir-ts-mode, heex-ts-mode
;; are set up in the eglot-server-programs variable to look for
;; language_server.sh
;; so just add that to the path in ~/.profile using...
;; PATH="/opt/elixir-ls/release:$PATH"
(require 'eglot)
(add-hook 'elixir-ts-mode-hook 'eglot-ensure)

(require 'yaml-ts-mode)

```


### Install language grammars for Tree-sitter {#install-language-grammars-for-tree-sitter}

To add the language grammars interactively use:

```bash
M-x treesit-install-language-grammar
```

Then add the language name, URL and accept all defaults.

For example I immediately set up:

| language | URL                                                        |
|----------|------------------------------------------------------------|
| elixir   | <https://github.com/elixir-lang/tree-sitter-elixir.git>    |
| heex     | <https://github.com/phoenixframework/tree-sitter-heex.git> |
| yaml     | <https://github.com/tree-sitter-grammars/tree-sitter-yaml> |

And Enjoy!
