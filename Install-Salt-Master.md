## Install Salt Master
```
wget -O - https://repo.saltstack.com/py3/ubuntu/18.04/amd64/latest/SALTSTACK-GPG-KEY.pub | sudo apt-key add -
printf 'deb http://repo.saltstack.com/py3/ubuntu/18.04/amd64/latest bionic main\n' | sudo tee -a /etc/apt/sources.list.d/saltstack.list
sudo apt update
sudo apt install salt-master python3-pygit2
```

## Github SSH Key
This may be temporary.  When we have our own local Git server we will replace
this.

```
ssh-keygen -t ed25519 -C "SaltStack Master GitHub"
```
