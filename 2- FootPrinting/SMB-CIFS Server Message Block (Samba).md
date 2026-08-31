CIFS operates over TCP port `445`

| **SMB Version** | **Supported**                       | **Features**                                                           |
| --------------- | ----------------------------------- | ---------------------------------------------------------------------- |
| CIFS            | Windows NT 4.0                      | Communication via NetBIOS interface                                    |
| SMB 1.0         | Windows 2000                        | Direct connection via TCP                                              |
| SMB 2.0         | Windows Vista, Windows Server 2008  | Performance upgrades, improved message signing, caching feature        |
| SMB 2.1         | Windows 7, Windows Server 2008 R2   | Locking mechanisms                                                     |
| SMB 3.0         | Windows 8, Windows Server 2012      | Multichannel connections, end-to-end encryption, remote storage access |
| SMB 3.0.2       | Windows 8.1, Windows Server 2012 R2 |                                                                        |
| SMB 3.1.1       | Windows 10, Windows Server 2016     | Integrity checking, AES-128 encryption                                 |
```shell
smbclient -N -L //10.129.14.128
```
`null session` (`-N`), which is `anonymous` access without the input of existing users or valid passwords.


```
smbclient \\\\$IP$\\$FOLDER$
```

```
smbclient -U $bob$ \\\\$IP$\\$FOLDER$
```

```shell
smbclient //10.129.136.220/devshare -U alex
```

# smbmap to enumerate the target service 

```shell
smbmap -H $TARGET
```
# enum4linux-ng
```shell
git clone https://github.com/cddmp/enum4linux-ng.git
```

```shell 
cd enum4linux-ng
```

```shell
pip3 install -r requirements.txt
```

```shell
./enum4linux-ng.py 10.129.14.128 -A
```

