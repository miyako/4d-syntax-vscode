![version](https://img.shields.io/badge/version-20%2B-EB8E5F)
![platform](https://img.shields.io/static/v1?label=platform&message=mac-intel%20|%20mac-arm&color=blue)
[![license](https://img.shields.io/github/license/miyako/4d-syntax-vscode)](LICENSE)
![downloads](https://img.shields.io/github/downloads/miyako/4d-syntax-vscode/total)

# 4d-syntax-vscode
A Visual Studio Code syntax extension for 4D.

based on [4D.tmbundle](https://github.com/miyako/4D.tmbundle)

normally you should use the LSP aware [4D-Analyzer](https://marketplace.visualstudio.com/items?itemName=4D.4d-analyzer).

<img src="https://github.com/user-attachments/assets/1235826e-6a98-49fb-8719-1a2f995cdd60" width=400 height=auto />

## remarks

the `support` scope typically has no colour assigned so `variable` is used instead.  

the negative look-behind assertion `(?<!\\w)` does not match leading space, so `(\\s*)` is used instead.
