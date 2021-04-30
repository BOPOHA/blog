---
title: "fedora: make gnome title bar smaller"
date: 2016-08-29T11:11:08+00:00
url: /2016/08/fedora-make-gnome-title-bar-smaller/
categories:
  - fedora
tags:
  - fedora
  - gnome
  - smaller
  - titlebar

---
<a href="http://blog.samalik.com/make-your-gnome-title-bar-smaller-fedora-24-update/" target="_blank">Source.</a>  
[bash]  
echo &#8216;  
window.ssd headerbar.titlebar {  
padding-top: 4px;  
padding-bottom: 4px;  
min-height: 0;  
}

window.ssd headerbar.titlebar button.titlebutton {  
padding: 0px;  
min-height: 0;  
min-width: 0;  
}  
&#8216; > ~/.config/gtk-3.0/gtk.css  
[/bash]