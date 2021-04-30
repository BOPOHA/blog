---
title: "WP-monitoring"
date: 2016-04-29T00:43:21+00:00
url: /2016/04/wp-monitoring/
categories:
  - Linux
  - virtualmin

---
[bash]  
git clone https://github.com/rsmmr/git-notifier  
cd public_html  
git init  
echo &#8216;*.jpg  
*.jpeg  
*.gif  
*.png  
*.ico  
*.css  
*.zip  
*.tgz  
*.gz  
*.rar  
*.bz2  
*.doc  
*.xls  
*.exe  
*.pdf  
*.ppt  
*.txt  
*.tar  
*.mid  
*.midi  
*.wav  
*.bmp  
*.rtf  
*.js  
*.wmv  
*.wma  
*.mp3  
*.mpg  
*.avi  
*.mpeg  
*.mp4  
*.divx  
*.ttf  
*.woff2  
*.woff  
*.swf  
*.scss  
*.svg  
*.mo  
*.po  
*.pot  
*.eot  
*.xap  
*.json  
*.xml  
*.md  
*.non_progressive  
.git-notifier.dat  
.git-notifier.dat.bak  
git-notifier.log  
wp-content/cache/  
&#8216; > .gitignore  
git config hooks.mailinglist your@mail  
ln -s $HOME/git-notifier/git-notifier .git/hooks/post-commit

crontab -l > tmp.cron  
echo &#8216;0 \* \* \* \* cd $HOME/public_html ; git add . && git commit -am &#8220;$(date +\%Y-\%m-\%d@\%H:\%M)&#8221; > /dev/null&#8217; >> tmp.cron  
crontab tmp.cron  
rm -rf tmp.cron  
[/bash]