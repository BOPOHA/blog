---
title: "Compress Images && Progressive JPEGs"
date: 2016-04-28T22:23:41+00:00
draft: false
categories:
  - Linux

---
```shell
yum install ImageMagick -y

for fn in `find /home/ -name "*.jpg"`; do  
    if [ ! -f "$fn.non_progressive" ]; then  
      cp $fn $fn.non_progressive ;
    fi
    convert -strip -interlace Plane -quality 75% $fn.non_progressive $fn
done

find /home/ -name "*.non_progressive" -delete
```
