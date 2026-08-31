```
keelwa@htb[/htb]$ python -c 'import pty; pty.spawn("/bin/bash")'
```

After we run this command, we will hit `ctrl+z` to background our shell and get back on our local terminal, and input the following `stty` command:
```
keelwa@htb[/htb]$ stty raw -echo 
keelwa@htb[/htb]$ fg 
```
[Enter] 
[Enter]
```
www-data@remotehost$

www-data@remotehost$ export TERM=xterm-256color

www-data@remotehost$ stty rows 67 columns 318
```

