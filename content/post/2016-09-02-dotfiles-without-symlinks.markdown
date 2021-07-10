---
author: "Luke Hinds"
date: "2016-08-17T16:29:56+01:00"
linktitle: Dotfiles without symlinks
title: Dotfiles without symlinks
weight: 10
tags: [
    "coding",
]
categories: [
    "Coding",
]
---

Hands down, this is the best way I have found so far, for managing dotfiles
without needing to symlink to a local repo somewhere.

Instead we use a `--bare` git repository and set up a simple alias.

## Setup
```bash
git init --bare $HOME/.dotfiles
alias dotfiles='/usr/bin/git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME'
echo "alias dotfiles='/usr/bin/git --git-dir=$HOME/.dotfiles/ --work-tree=$HOME'" >> $HOME/.zshrc
```

* note: I am a zshrc user, if your using bash, then echo the alias to .bashrc

## Github

Create a github repo, named 'dotfiles'

## Set origin for github repo
```bash
dotfiles remote add origin git@github.com:<username>/dotfiles.git
```

## Usage
```bash
dotfiles status
dotfiles add .config/i3/config
dotfiles commit -m 'Adding i3 Config'
dotfiles push
```

## Replication

Should you then want to set up your dotfiles on another machine:

```bash
git clone --separate-git-dir=$HOME/.dotfiles https://github.com/<username>/dotfiles.git dotfiles-tmp
rsync --recursive --verbose --exclude '.git' dotfiles-tmp/ $HOME/
rm --recursive dotfiles-tmp
```
