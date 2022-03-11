---
title: "Find Who Is Logged in by Ssh Key Fingerprint"
date: 2020-10-12T11:42:40+01:00
categories:
  - Linux
  - ssh

---


# Old debian based OS

```shell
$ grep 'Accepted publickey' /var/log/auth.log
...
Oct 12 07:09:05 <hostname> sshd[54321]: Accepted publickey for <username> from <ip> port 12345 ssh2: ED25519 00:00:00:00:00:00:00:00:00:00:00:00:00:00
...
```

```shell
sudo -i -u username sh -c  'cat ~/.ssh/authorized_keys | sed "s/^/localhost /" > /tmp/authorized_keys'
ssh-keygen -l -f /tmp/authorized_keys | grep 00:00:00:00:00:00:00:00:00:00:00:00:00:00
256 00:00:00:00:00:00:00:00:00:00:00:00:00:00  user@example.com (ED25519)
```


# Modern RHEL/Debian based OS

```shell
$ journalctl -u sshd | grep 'Accepted publickey'
Oct 12 01:02:04 <hostname> sshd[54321]: Accepted publickey for <username> from <ip> port 12345 ssh2: ECDSA SHA256:ABCDEFGabcdefgABCDEFGabcdefgABCDEFGabcdefg
Oct 12 01:07:16 <hostname> sshd[54322]: Accepted publickey for <username> from <ip> port 12346 ssh2: RSA SHA256:abcdefgABCDEFGabcdefgABCDEFGabcdefgABCDEFG
Oct 12 07:24:13 <hostname> sshd[54323]: Accepted publickey for <username> from <ip> port 12347 ssh2: RSA SHA256:DEFGabcdefgABCDEFGabcdefgABCDEFGabcdefgABC

```

```shell
sudo -i -u username sh -c 'ssh-keygen -l -f ~/.ssh/authorized_keys'
2048 SHA256:abcdefgABCDEFGabcdefgABCDEFGabcdefgABCDEFG user1@example.com (RSA)
521 SHA256:ABCDEFGabcdefgABCDEFGabcdefgABCDEFGabcdefg user2@example.com (ECDSA)
4096 SHA256:DEFGabcdefgABCDEFGabcdefgABCDEFGabcdefgABC user3@example.com (RSA)
```

