Things that I consider "unixy":

- pipes
- "do one thing and do it well"
- small, composable programs

But the output of `ps` and `ls` isn't easy to work with. Especially if you look
at another classic Unix built-in: `cut`. They're almost entirely incompatible.
Cut expects the field separator to be `Tab`, and even if you specify that the
separator is space `-d ' '` it still won't work with `ps` and `ls -l`.

Just take a look at:

```
ps -aux | cut -f 2
ls -l | cut -f 2
```

https://stackoverflow.com/questions/15643834/using-bash-ps-and-cut-together

