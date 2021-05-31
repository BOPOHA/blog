---
title: "ZE551ML: flash firmware"
date: 2016-04-29T01:39:26+00:00
url: /2016/04/ze551ml-flash-firmware/
categories:
  - Phablets

---
Asus ZenFone 2

  1. Unlock boot loader.  
    Unlock Device App you can find at  `https://www.asus.com` [here][1] in Utilities:  
    [Direct link][2].
  2. Download lastest firmware...
    you can find it at https://www.asus.com [here][1] in Firmware (select "for WW SKU only*")  
    Also, direct link can be curled:  

    URL='https://www.asus.com/us/support/Download/39/1/0/13/Lk0Sg1Ulh0Ph49Hl/32/'  
    curl -s $URL| grep -o "\"http://dlcdnet.asus.com/pub/ASUS/ZenFone/ZE551ML/UL-Z00A-WW.*.zip\""  
    "http://dlcdnet.asus.com/pub/ASUS/ZenFone/ZE551ML/UL-Z00A-WW-2.20.40.174-user.zip"  

  3. TWRP  
    u can found here https://dl.twrp.me/Z00A/ ([twrp-3.0.0-0-Z00A.img][3] 15.3M )  

  4. adb sideload

 [1]: https://www.asus.com/us/Phone/ZenFone_2_ZE551ML/HelpDesk_Download/
 [2]: http://dlcdnet.asus.com/pub/ASUS/ZenFone/ZE551ML/UnlockApp_ze551ml_20150723.apk
 [3]: https://dl.twrp.me/Z00A/twrp-3.0.0-0-Z00A.img