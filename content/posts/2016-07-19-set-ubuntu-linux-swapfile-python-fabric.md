---
title: Set Ubuntu Linux Swapfile Using Python Fabric
author: Ben E. Boyter
type: post
date: 2016-07-19T22:53:52+00:00
url: /2016/07/set-ubuntu-linux-swapfile-python-fabric/
nkweb_code_in_head:
  - default
nkweb_Use_Custom_js:
  - default
nkweb_Use_Custom_Values:
  - default
nkweb_Use_Custom:
  - 'false'
categories:
  - Python Fabric

---
Annoyingly most cloud providers have an irritating habit of not adding any swap memory to any instance you spin up. Probably because if they added swap to the instance the disk size would appear to be smaller then it is or if they had a dedicated swap partition they would have to bear the cost or again use some of your disk space. 

Thankfully adding swap to your Ubuntu linux instance is fairly easy. The following task when run will check if a swapfile already exists on the host and if not create one, mount it and set it to be remounted when the instance is rebooted. It takes in a parameter which specifies the size of the swap in gigabytes.

    
    def setup_swapfile(size=1):
        if fabric.contrib.files.exists('/swapfile') == False:
            sudo('''fallocate -l %sG /swapfile''' % (size))
            sudo('''chmod 600 /swapfile''')
            sudo('''mkswap /swapfile''')
            sudo('''swapon /swapfile''')
            sudo('''echo "/swapfile   none    swap    sw    0   0" >> /etc/fstab''')
    

BTW I am writing a book about how to [Automate your life using Python Fabric][1] click the link and register your interest for when it is released.

 [1]: https://leanpub.com/pythonfabricautomation