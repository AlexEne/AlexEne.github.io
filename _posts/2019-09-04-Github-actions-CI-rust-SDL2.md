---
layout: post
title: Github Actions CI with Rust and SDL2
---

Github has added a way to have CI with [github actions](https://github.blog/2019-08-08-github-actions-now-supports-ci-cd/) and I decided to try it after I saw [how well it worked]((https://github.com/yak32/glw_json/blob/master/.github/workflows/main.yml)) for my colleague, Iakov.  
I am doing a game in Rust and I need to test it on Windows, Linux and MacOS.  
However, while windows and Linux tests are easy to do with the help of WSL on my own machine, I don't own a Mac.  

Besides supporting all three platforms I was interested in, Github also [offers](https://github.com/features/actions) a good range of pay as you go prices with a cost per minute of: 0.008$ for Linux, 0.016$ for Windows and 0.08$ for MacOS as well as **2000 minutes of a free tier**.  
This is great news for hobby projects like mine that happen to be on a private github repository.  

Below you can find the action that I'm currently using.  
It installs SDL2 on Linux and MacOS and assumes the DLLs for Windows are in the git repo.  
If you think that pushing some DLLs in a git repo is not a clean solution, there's the alternative of using something like [this PowerShell script](https://github.com/AlexEne/rust-particles/blob/master/appveyor_get_sdl_dll.ps1) to get the SDL 2 DLLs.  

It comes with the stable Rust 1.37 version on Linux and the Windows distribution.
I had to manually install it on MacOS.  
Using `rustup` you can also get `nightly/beta` versions if needed.

More information on how to set up github actions CI for your own projects:
- [Virtual environments available](https://help.github.com/en/articles/virtual-environments-for-github-actions)
- [Workflow yaml syntax](https://help.github.com/en/articles/workflow-syntax-for-github-actions)

## The action

```yaml
name: Rust

on: [push]

jobs:
  test_ubuntu:

    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v1
    - name: install_dependencies
      run: |
        sudo add-apt-repository -y "deb http://archive.ubuntu.com/ubuntu `lsb_release -sc` main universe restricted multiverse"
        sudo apt-get update -y -qq
        sudo apt-get install libsdl2-dev
    - name: Build
      run: |
        cargo build
    - name: Run tests
      run: cargo test 
      
  test_on_macOS:

    runs-on: macOS-latest
    
    steps:
    - uses: actions/checkout@v1
    - name: install_dependencies
      run: | 
        brew install SDL2
        brew install rustup
        rustup-init -y --default-toolchain stable
    - name: build
      run: |
        export PATH="$HOME/.cargo/bin:$PATH"
        cargo build
    - name: test
      run: |
        export PATH="$HOME/.cargo/bin:$PATH"
        cargo test
      
  test_on_Windows:
    runs-on: windows-2016
    
    steps:
    - uses: actions/checkout@v1
    - name: build
      run: cargo build
    - name: test
      run: cargo test
```

## Some details

On `ubuntu-latest` I had to do some tricks to install `libsdl2-dev`.  
Just doing `sudo apt-get install libsdl2-dev` doesn't work right now as it has a missing package.

On Windows I use `windows-2016` it has Visual Studio 2017. For now, that's the newest version supported by rust-hawktracer.  
After I will update my build script for rust-hawktracer to handle Visual Studio 2019, `windows-latest` will work too.