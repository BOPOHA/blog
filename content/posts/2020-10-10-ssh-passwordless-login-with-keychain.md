---
title: "SSH Passwordless Login With Keychain"
date: 2020-10-10T17:22:40+01:00
categories:
  - Linux
  - ssh
---

# Install Keychain

```shell
dnf install keychain -y
# apt-get install keychain -y
# pkg install keychain -y
```

# Generate a new ssh key if not exist

```shell
ssh-keygen -t ecdsa-sha2-nistp256 -C $USERNAME -f ~/.ssh/$USERNAME
```
Assign the pass phrase when prompted.

# How to use Keychain

Update your $HOME/.bash_profile file or simular initialization file for zsh, fish, etc.
```shell
vi $HOME/.bash_profile
```
Append the following code:
```shell
alias sshkey_secured="keychain $USERNAME --nogui --quiet --timeout 60 ; source ~/.keychain/$HOSTNAME-sh"
source ~/.keychain/$HOSTNAME-sh
```

Run once in the same terminal to update environment:
```shell
source $HOME/.bash_profile
```
Start ssh agent with keychain:

```shell
sshkey_secured
```
Enter pass phrase when prompted.
The ssh-agent's key will be cleared automatically after 60 minutes.
