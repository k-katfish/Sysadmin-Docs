# Sysadmin-Docs

How to do all the things nobody will teach you about System Administration and other nonsense. In other words, the beginner Sysadmin's guide to fakeing-it-till-you-make-it.

All of this documentation will be written for Rocky Linux/RHEL Linux/CentOS7&8. Basically enterprise-grade linux. For other distros... this isn't the place to look.

## Some philosophy things

1. If it's in production it must be monitored
2. Do you use Single Sign On? Should you be using SSO? (yes you should and you know it)
3. WTF is this timezone thing? Everything should always be in UTC time. Always. ALLLLLLLWWAAAAAAYYYYYYSSSSS. Unless you know for a fact that every single site and service you're administering is in the same timezone (which it almost never is) you should be using UTC time.

## What is Enterprise Linux?

RedHat sells Linux. Don't as me how, idk how you can sell something that's free, but RedHat found a way. They also sell some really good documentation. But it's all paid. If you've got the money: use RedHat Enterprise Linux (RHEL). If not, you can use the free Enterprise Linux alternative Rocky Linux. The documentation for the two are entirely interchangable and anything you can do on RHEL you can do on Rocky and visa-versa.

## Recommended Reading

1. The Unix and Linux System Adminstration Handbook, 5th Edition by Evi Nemeth, Garth Snyder, Trent Hein, Ben Whaley, Dan Mackin, et al.
    It's the one with the lovely cartoons all over the front. Literally cannot recommend this book highly enough. It's an absolute masterpice. This book won't ever find its way onto your bookshelf, because it'll be living on your desk right next to your mouse.
