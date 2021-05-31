---
title: "How to disable keyring password storage for chrome"
date: 2016-04-29T00:38:55+00:00
url: /2016/04/how-to-disable-keyring-password-storage-for-chrome/
categories:
  - Linux

---
```shell
orig_desktop=/usr/share/applications/google-chrome.desktop
new_desktop="$HOME/.local/share/applications/google-chrome.desktop" 
cat "${orig_desktop}" | sed "s|google-chrome-stable|google-chrome-stable --password-store=basic --ssl-version-min=tls1.2|" > "${new_desktop}"
```
