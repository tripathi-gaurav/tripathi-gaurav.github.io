---
layout: post
title:  "Running MacOS on Debian/Ubuntu/Pop_OS!"
date:   2018-06-03 23:27:07 -0400
categories: technology general
---
If you want to experiment with a MacOS system and use GNU/Linux on your machines, then it's actually quite simple to setup a virtual instance of MacOS on [VirtualBox][virtualbox].
Instead of looking for a Hackintosh image, I tried using a [Vagrant][vagrant-download] image and was able to boot into MacOS within minutes.

## 1. Prerequisites

0. Install VirtualBox from VirtualBox's [download page][virtualbox-download]   
  Install additional dependencies:
  `sudo apt-get install virtualbox-guest-utils virtualbox-guest-x11 virtualbox-guest-dkms`

1. Setup Vagrant by downloading the appropriate Vagrant DEB package from Vagrant's [download page][vagrant-download]
Jekyll also offers powerful support for code snippets:


## 2. Meat

1. Setup Vagrant image. In a shell, do the following:

{% highlight shell %}
mkdir vagrant
cd vagrant
mkdir macos
cd macos
vagrant init jhcook/macos-sierra
{% endhighlight %}

2. Start the image! <br/>
    `vagrant up`

  3. Open VirtualBox and select `macos_default_xxxxx` VM and choose `Show` and it will bring up the GUI. <br/>
  Start the VM from VirtualBox in future. <br/>
  Default username and password is `vagrant` <br/>

[virtualbox]: https://www.virtualbox.org/
[virtualbox-download]:  https://www.virtualbox.org/wiki/Linux_Downloads
[vagrant]: https://www.vagrantup.com/
[vagrant-download]: https://www.vagrantup.com/downloads.html
