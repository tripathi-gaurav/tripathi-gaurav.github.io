---
layout: post
title:  "Android Studio: /dev/kvm permission denied"
date:   2018-07-16 23:27:07 -0400
categories: technology general
---
## Simple resolution to the Android Studio error: /dev/kvm permission denied

{% highlight shell %}
1098  sudo apt install qemu-kvm
1099  grep kvm /etc/group
1100  ls -al /dev/kvm
1101  sudo adduser username kvm
sudo chown username /dev/kvm
{% endhighlight %}
