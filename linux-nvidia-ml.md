How-to use the integrated GPU for display and the NVIDIA discrete GPU for machine learning under Ubuntu
Based on https://askubuntu.com/questions/1061551/how-to-configure-igpu-for-xserver-and-nvidia-gpu-for-cuda-work/1099963#1099963

In Ubuntu 24.04 and recent versions, xorg.conf file doesn't exist by default. It is not needed in these distros. 
However, custom settings can be stored in this file and X.org will use them.

You can find the list of GPUs and their PCI address with this command.
```
$ lscpi | grep VGA
00:02.0 VGA compatible controller: Intel Corporation Xeon E3-1200 v3 Processor Integrated Graphics Controller (rev 06)
01:00.0 VGA compatible controller: NVIDIA Corporation GP104 [GeForce GTX 1080] (rev a1)
```

Create or append /etc/X11/xorg.conf.d/xorg.conf file with the contents. PCI address (00:02.0) above is written as 'PCI:0:2:0' in this file. 

If the iGPU is Intel:

```
Section "Device"
    Identifier      "intel"
    Driver          "intel"
    BusId           "PCI:0:2:0"
EndSection

Section "Screen"
    Identifier      "intel"
    Device          "intel"
EndSection
```

If the iGPU is ASPEED:

```
Section "Device"
    Identifier      "ast"
    Driver          "ast"
    BusId           "PCI:0:2:0"
EndSection

Section "Screen"
    Identifier      "ast"
    Device          "ast"
EndSection
```
