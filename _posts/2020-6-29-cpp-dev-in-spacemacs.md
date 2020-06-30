---
layout: post
title: C++ in spacemacs
---

Two new emacs packages, ['lsp-mode'](https://github.com/emacs-lsp/lsp-mode) and ['dap-mode'](https://github.com/emacs-lsp/dap-mode), have 
brought the power of Microsoft's [Language Server Protocol](ttps://github.com/Microsoft/language-server-protocol/) and [Debug Adapter Protocol](https://microsoft.github.io/debug-adapter-protocol/)
to emacs.

This is a guide to getting `lsp-mode` and `dap-mode` working in spacemacs for C++.

Spacemacs' [C++ layer](https://develop.spacemacs.org/layers/+lang/c-c++/README.html) supports multiple backends for `lsp-mode`. We'll be using 
[clangd](https://clangd.llvm.org/), a language server built on clang.

The distro I use in this post is Ubuntu 20.04.

*Disclaimer*

I wrote this *after* figuring out how to configure everything, so it's highly likely I've forgotten something or missed a step. Please let me know if this doesn't work for you.

## Dependencies

### clang tools

We will be installing our clang tools from the Ubuntu nightly repos provided by llvm.

Add the gpg key and repository config to apt:

    $ wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key | sudo apt-key add -
    $ sudo add-apt-repository "deb http://apt.llvm.org/focal/ llvm-toolchain-focal main"
    $ sudo apt update

Install the clang tools

    $ sudo apt install -y clang-format clang-tidy clang clangd libclang-dev liblldb-11-dev lldb
    
### node

`dap-mode` leverages the vscode debug adapters which are written in javascript. As such we need to install node

    $ sudo apt install nodejs

## `lsp-mode`

`lsp-mode` will run `clangd` and communicate with it in order to get information about your C++ code.

You need to tell `clangd` where to find [`compile_commands.json`](https://clang.llvm.org/docs/JSONCompilationDatabase.html), the 
compilation database which describes how your code is compiled, where to find includes etc.

You probably want to do this on a per-project basis, so that different projects with different settings don't interfere with eachother.

In order to do this you need to add configuration for `lsp-clients-clangd-args` to your project's `.dir-locals.el`

    ((c++-mode . ((lsp-clients-clangd-args . ("--compile-commands-dir=build"
                                              "--pch-storage=memory"
                                              "--background-index"
                                              "-j=4"
                                              ))
                  )))
                  
Here you can see that I've added a number of configuration arguments for clangd, the most important being `--compile-commands-dir`, here specified 
as a subdirectory `build`, which is relative to the source tree's root dir.

The other options I've specified are performance related:

    --pch-storage=<value> - Storing PCHs in memory increases memory usages, but may improve performance
    --background-index    - Index project code in the background and persist index on disk.
    -j=<uint>             - Number of async workers used by clangd. Background index also uses this many workers.

## `dap-mode`

In order to communicate with gdb and/or lldb, we need to install the [vscode debug adapter for gdb and lldb](https://github.com/WebFreak001/code-debug).

    M-x dap-gdb-lldb-setup

## spacemacs layers

Now we need to configure the `lsp`, `dap` and `c-c++` layers in our spacemacs config.

    dotspacemacs-configuration-layers
    '(
      ....
      (lsp :variables
           lsp-restart 'auto-restart) ; if the server exits, just restart it without prompting
      (dap :variables
           dap-enable-ui-controls nil ; don't display the mouse buttons 
           dap-auto-configure-features '(sessions locals breakpoints expressions tooltip)) ; use the auto-configure layout, but no mouse buttons
      (c-c++ :variables
             c-c++-backend 'lsp-clangd)
      ....
      )
    
## Debugging an app

You can add debug configurations for individual apps to your project's `.dir-locals-el` file

Here is an example:

    ((c++-mode . ((dap-debug-template-configurations . (
                                                        ("my_app"
                                                        :type "gdb"
                                                        :target "/home/user/src/build/app/my_app"
                                                        )
                                                        )))))
                                                        
With that added to your `.dir-locals.el` you can reload your project so that debug-template is added to `dap-mode`, and then start debugging:

    `SPC d b a`: adds a breakpoint at the current source line
    `SPC d d d`: opens a list of debug-templates, select `my_app` and it should start
                                                        
