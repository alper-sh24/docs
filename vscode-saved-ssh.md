# Saving VSCode ssh connection

## Initial connection

The authenticity of host '10.10.12.210 (10.10.12.210)' can't be established.
ED25519 key fingerprint is SHA256:7OBDxdxxxxluDyTblJbY3Z0OxxxvgJpMZB0Y.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.10.12.210' (ED25519) to the list of known hosts.

## Creating Identity

The steps provided at StackOverflow: https://stackoverflow.com/a/69970152

generate a public and a private key with something like OpenSSL

```bash
ssh-keygen -q -b 2048 -P "" -f /Users/<localuser>/.ssh/keys/<myhost>_rsa -t rsa
```

This should make two files:

```
<myhost>_rsa        (private key)
<myhost>_rsa.pub    (public key)
```

The private key (`<myhost>_rsa`) can stay in the local .ssh folder

The public key (`<myhost>_rsa.pub`) needs to be copied to the server (`<myhost>`)

### On the Server

There is a file on the server which has a list of public keys inside it: **<remoteuserhome>/.ssh/authorized_keys**

If it exists already, you need to add the contents of <myhost>\_rsa.pub to the end of the file.

If it does not exist you can use the <myhost>\_rsa.pub and rename it to authorized_keys with permissions of 600.

If everything goes according to plan you should now be able to go into terminal and type

ssh <remoteuser>@<myhost>

and you should be in without a password. The same will now apply in Visual Studio Code.

## Saving Identity

File: **.ssh/config**
Example content:

```conf
Host 10.10.12.179
  HostName 10.10.12.179
  User administrator
  Port 22
  PreferredAuthentications publickey
  IdentityFile "~/.ssh/forvscode"
```
