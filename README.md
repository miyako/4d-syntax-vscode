# 4d-syntax-vscode
A Visual Studio Code syntax extension for 4D.

based on [4D.tmbundle](https://github.com/miyako/4D.tmbundle)

## remarks

the `support` scope typically has no colour assigned so `variable` is used instead.  

the negative look-behind assertion `(?<!\\w)` does not match leading space, so `(\\s*)` is used instead.
