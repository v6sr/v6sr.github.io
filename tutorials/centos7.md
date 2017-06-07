---
layout: default
---

# [](#header-1)Introduction

In order to enable IPv6 Segment Routing (SR) on CentOS 7, you must install the latest mainline kernel. The default kernel does not support SR at this time. The good news is that its not difficult to install the mainline kernel. This instructions should work with RHEL 7 as well. 

## [](#header-2) Step One - Install ELRepo

Execute the following commands to install and enable the ELRepo.

```
sudo rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
sudo rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm 
```

Edit /etc/yum.repos.d/elrepo.repo and change the enabled key to '1' under [[elrepo-kernel]].

## [](#header-2) Step Two - Install the latest Mainline Kernel

To install the Mainline kernel, first we update with yum and then install the "ML" kernel packages. We will also install the kernel tools/headers because they might necessary for other
tutorials.

```
sudo yum update
sudo yum remove kernel-headers kernel-tools kernel-tools-libs kernel-tools-libs-devel
sudo yum install kernel-ml kernel-ml-headers kernel-ml-tools kernel-ml-tools-libs kernel-ml-tools-libs-devel
```

## [](#header-2) Step Three - Configure Grub

Once the kernel is installed, we have to configure Grub to boot the new mainline kernel. 

First, we list all the kernels installed on the machine.
```
sudo grep vmlinuz /boot/grub2/grub.cfg
```

The output will look something like this:

```
linux16 /vmlinuz-4.11.3-1.el7.elrepo.x86_64 root=/dev/mapper/cl-root ro crashkernel=auto rd.lvm.lv=cl/root rd.lvm.lv=cl/swap rhgb quiet
linux16 /vmlinuz-3.10.0-514.21.1.el7.x86_64 root=/dev/mapper/cl-root ro crashkernel=auto rd.lvm.lv=cl/root rd.lvm.lv=cl/swap rhgb quiet
```

We want the latest kernel (in this case 4.11.3-1). The kernels in grub.cfg are indexed starting at 0, so we'll select kernel 0 as the default and reboot. 

```
sudo grub2-set-default 0
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
sudo reboot
```

## [](#header-2)Step Four - Verify

The last step is to confirm that the correct kernel was booted. 

```
uname -r
```

The output should be the kernel version that was selected in Step Three. 

```
4.11.3-1.el7.elrepo.x86_64
```

That is all, now you are ready to send and route IPv6 SR packets. 

[Return to the Main page](../)
