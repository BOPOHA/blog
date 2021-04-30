---
title: "How to disable keyring password storage for chrome"
date: 2016-04-29T00:38:55+00:00
url: /2016/04/how-to-disable-keyring-password-storage-for-chrome/
categories:
  - Linux

---
[bash]  
new_desktop=/usr/share/applications/custom-google-chrome.desktop  
cp -a /usr/share/applications/google-chrome.desktop $new_desktop  
sed -i &#8220;s|google-chrome-stable|google-chrome-stable &#8211;password-store=basic &#8211;ssl-version-min=tls1.2|&#8221; $new_desktop  
sed -i &#8220;s|Google Chrome|Custom Google Chrome|&#8221; $new_desktop  
[/bash]