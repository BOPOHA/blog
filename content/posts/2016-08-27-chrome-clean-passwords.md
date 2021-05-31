---
title: "google-chrome: clean passwords"
date: 2016-11-30T00:00:00+00:00
draft: false

---
Run google-chrome with --password-store=basic ([example][1])  
```shell 
dnf install sqliteman  
cp ~/.config/google-chrome/Default/Web\ Data{,.bak}  
sqliteman ~/.config/google-chrome/Default/Web\ Data  
```

 [1]: https://blog.vorona.me/2016/04/how-to-disable-keyring-password-storage-for-chrome/
