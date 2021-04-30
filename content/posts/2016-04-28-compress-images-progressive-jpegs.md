---
title: "Compress Images && Progressive JPEGs"
date: 2016-04-28T22:23:41+00:00
draft: false
url: /?p=1
categories:
  - Linux

---
[bash]  
yum install ImageMagick -y

for fn in \`find /home/ -name &#8220;*.jpg&#8221;\`; do  
if [ ! -f $FILE ]; then  
cp $fn $fn.non_progressive ;  
convert -strip -interlace Plane -quality 75% $fn.non_progressive $fn ;  
elif  
convert -strip -interlace Plane -quality 75% $fn.non_progressive $fn ;  
fi  
done

find /home/ -name &#8220;*.non_progressive&#8221; -delete  
[/bash]
