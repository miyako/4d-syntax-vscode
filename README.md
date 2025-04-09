# 4d-syntax-vscode
A Visual Studio Code syntax extension for 4D.

based on [4D.tmbundle](https://github.com/miyako/4D.tmbundle)

normally you should use the LSP aware [4D-Analzer](https://marketplace.visualstudio.com/items?itemName=4D.4d-analyzer).

<img src="https://github.com/user-attachments/assets/1235826e-6a98-49fb-8719-1a2f995cdd60" width=400 height=auto />

## remarks

the `support` scope typically has no colour assigned so `variable` is used instead.  

the negative look-behind assertion `(?<!\\w)` does not match leading space, so `(\\s*)` is used instead.
