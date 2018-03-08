---
layout: post
title:  "My work environment setup"
date:   2018-03-07 14:00:00
tags:   tools
language: EN
---

I am a Software Engineer and I like when the tools I use help me through my day of work, not when they make it complicated. This goes through the choice of tools that I use, and how I configure them. In this post, I thought I would share what are "my essentials" when I use my Mac...

# Visual Studio Code

I was a great fan of Sublime Text... until [Visual Studio Code](https://code.visualstudio.com/) came out. VS Code is great text editor, and it does much more (markdown preview, including images; Javascript debugger, etc.). Lots of plugins are available, updates are frequent (unlike Sublime...), and it is fast (unlike Atom). And no, it has nothing to do with the Visual Studio IDE.

![Visual Studio Code](/images/my-env-vscode.png)

# IntelliJ IDEA (Community Edition)

I am a Kotlin/Scala/Java developer, and [IntelliJ IDEA](https://www.jetbrains.com/idea/) is just the best IDE on the market for these languages. Many people use the Ultimate Edition (paid version) but I use the Community Edition. All I need is in there: support for the programming languages I use (Kotlin/Scala/Java), support for the build tools I use (Gradle/Maven/SBT).

![IntelliJ IDEA](/images/my-env-intellij.png)

# SourceTree

[SourceTree](https://www.sourcetreeapp.com/) is a GUI for Git. What I absolutely love with SourceTree is the ability to stage _hunks_ of code (this can even be a single line). This allows you to precisely select what you are committing, and what you are leaving behind. With SourceTree, you will never commit something unwanted.

![SourceTree](/images/my-env-sourcetree.png)

# iTerm2

[iTerm2](https://www.iterm2.com/) is a great replacement for Terminal. It allows you to split your windows, it has better text editing features, and lots of options.

![iTerm](/images/my-env-iterm.png)

# Oh My Zsh

[Oh My Zsh](https://github.com/robbyrussell/oh-my-zsh) is a tool to manage your Zsh configuration. It has plugins, such as a Git plugin to display information about the current Git repository in your prompt.

My prompt is largely customized to display the following information:

- the time, so that I can go back and see when I typed a command, and also to have an estimate of how long a command took to execute
- the user and hostname, so that I know where this console is running, which is useful when I log in on EC2 instances
- the full path to the current directory, so that I know where commands are executed, and so that I can easily copy/paste the path
- Git information, to know what branch I am in, and to know if there are non-committed changes (displays a red '*').

The colors allow me to easily spot the information of the prompt, and also to see where commands where typed (it is hard to distinguish commands from results if your terminal displays all the text in one color).

![My prompt](/images/my-env-prompt.png)

# Homebrew

[Homebrew](https://brew.sh/) is a package installer. Almost everything I install on my Mac comes through Homebrew. It just works.

Want to install Maven? Just run `brew install maven`!

I also recommend using [Homebrew Cask](https://github.com/caskroom/homebrew-cask) to be able to install applications that have installers.

# f.lux

[f.lux](https://justgetflux.com/) is a small utility that changes the color scheme of your screen throughout the day so that it never looks too bright (and especially too blue at night).

Yes, my screen turns yellow-ish at night and I love it!

![f.lux](/images/my-env-flux.png)

# HyperSwitch

I used Microsoft Windows for a very long time, and _Alt+Tab_ allows you to cycle between your windows. _All_ your windows. On a Mac, on the contrary, you switch between applications with _Command+Tab_, and another shortcut to switch between windows of the same application. Too complicated for me!

[HyperSwitch](https://bahoom.com/hyperswitch) allows you to remap _Alt+Tab_ (or another shortcut) to cycle between your windows. It does other things but this is all I am using it for.

# ShiftIt

[ShiftIt](https://github.com/fikovnik/ShiftIt) is a tool that allows you to manage the size and position of your windows through keyboard shortcuts. It is simple and works perfectly. I use it to put 2 windows side-by-side, or to maximize a window.

![ShiftIt](/images/my-env-shiftit.png)
