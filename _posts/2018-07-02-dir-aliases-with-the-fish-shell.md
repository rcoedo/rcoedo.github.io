---
layout: post
resources: dir-aliases-with-the-fish-shell
title: Dir Aliases with the Fish Shell
date: 2018-07-02
footer_content: This article was [originally written for the Trabe medium publication](https://medium.com/trabe/continuation-local-storage-for-easy-context-passing-in-node-js-2461c2120284), a collection of excellent articles written by [the awesome people from trabe.io](https://trabe.io/).
---

My memory is terrible. I usually have a hard time remembering long commands and their parameters. For this reason, I constantly define many aliases in my shell environment, which I also end up forgetting.

To unclutter my environment, I wrote [a small Fish plugin](https://github.com/rcoedo/dir-abbr-fish) that allows me to write [fish abbreviations](https://fishshell.com/docs/current/commands.html#abbr) scoped by directory. To those unfamiliar with fish abbreviations, they are basically like aliases, but you get the command substituted in your prompt after you type it.

## What does this look like

To declare your abbreviations, you must write a `.abbr` file. A `.abbr` file looks like this:

```sh
y=yarn
init=rm -rf node_modules/; and yarn install
run=docker-compose -f config/docker-compose.yml up
clean=rm -rf lib/; and rm -rf node_modules/
```

Once you have your `.abbr` file, it will be loaded by fish automatically whenever you enter that directory. The first time fish encounters a `.abbr` file, you will be prompted to avoid accidentally loading abbreviations that could be harmful. When you leave that directory, fish will restore your abbreviations.

![dir-abbr-fish example](/resources/images/dir-aliases-with-the-fish-shell/example.gif)

You can also list your current directory abbreviations with `dir-abbr-list` and force a reload with `dir-abbr-reload.`

## Installing dir-abbr-fish

The plugin is available as a [fisher](https://github.com/jorgebucaran/fisher) plugin. Just type `fisher install rcoedo/dir-abbr-fish.`

Enjoy!
