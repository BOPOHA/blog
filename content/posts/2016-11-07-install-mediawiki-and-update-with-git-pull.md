---
title: "install MediaWiki and update with git pull"
date: 2016-11-07T19:40:58+00:00
url: /2016/11/install-mediawiki-and-update-with-git-pull/
categories:
  - Linux

---
Install:  
[code]  
cd ~/public_html  
git clone https://gerrit.wikimedia.org/r/mediawiki/core.git .  
git clone https://gerrit.wikimedia.org/r/mediawiki/vendor.git  
cd ~/public_html/skins/  
git clone https://gerrit.wikimedia.org/r/mediawiki/skins/CologneBlue  
git clone https://gerrit.wikimedia.org/r/mediawiki/skins/Modern  
git clone https://gerrit.wikimedia.org/r/mediawiki/skins/MonoBook￼￼  
git clone https://gerrit.wikimedia.org/r/mediawiki/skins/VectorV2  
git clone https://gerrit.wikimedia.org/r/mediawiki/skins/Vector  
[/code]

Update:  
[code]  
for i in \`find ~ -type d -name \.git\`; do cd \`dirname $i\` ; git pull ; done  
[/code]

Example:  
[code]  
-sh-4.2$ find . -type d -name \.git  
./public_html/skins/CologneBlue/.git  
./public_html/skins/Modern/.git  
./public_html/skins/VectorV2/.git  
./public_html/skins/Vector/.git  
./public_html/skins/MonoBook/.git  
./public_html/vendor/.git  
./public_html/.git  
[/code]  
Upgrade DB:  
[code]/opt/rh/rh-php72/root/usr/bin/php maintenance/update.php[/code]