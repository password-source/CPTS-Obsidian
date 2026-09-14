# we can enumerate nfs with nmap scripts

```shell
sudo nmap --script nfs* $TARGET -sV -p$PORTS(NFS,rpcbind)
```
#### this command will use all the nfs scripts for the target host 
# Show Available NFS Shares

```shell
showmount -e 10.129.14.128
```

# Mounting NFS Share

```shell
mkdir target-NFS
```

```shell
sudo mount -t nfs 10.129.14.128:/ ./target-NFS/ -o nolock
```

```shell
cd target-NFS
```

```shell
tree .
```

# List Contents with Usernames & Group Names

```shell
ls -l mnt/nfs/
```
# Unmounting

```shell
cd ..
```

```shell
sudo umount ./target-NFS
```