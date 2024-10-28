How-to use the integrated GPU for display and the NVIDIA discrete GPU for machine learning under Ubuntu

Based on https://askubuntu.com/questions/1061551/how-to-configure-igpu-for-xserver-and-nvidia-gpu-for-cuda-work/1099963#1099963

If you have both integrated (Intel, ASPEED, etc.) and discrete (NVIDIA) GPUs in your ML lab machine, you might want to use the integrated GPU for display and the NVIDIA GPU purely for ML to conserve all the NVIDIA GPU memory for your ML workloads.

By default, even if you connect your monitor only to the integrated GPU, X.org will use both GPUs. You can check if X.org is using the NVIDIA GPU with this command:
```
$ nvidia-smi
Mon Oct 28 10:55:21 2024       
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.183.06             Driver Version: 535.183.06   CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  NVIDIA GeForce GTX 1080        Off | 00000000:01:00.0 Off |                  N/A |
| 24%   37C    P2              42W / 180W |    115MiB /  8192MiB |      0%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+
                                                                                         
+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
|    0   N/A  N/A     29350      G   /usr/lib/xorg/Xorg                            4MiB |
|    0   N/A  N/A     29421    C+G   ...libexec/gnome-remote-desktop-daemon      105MiB |
+---------------------------------------------------------------------------------------+
```

The last 2 lines show that both X.org and Gnome remote desktop daemon are using the NVIDIA GPU. The following shows you how to disable X.org from using your NVIDIA GPU.

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

If you run nvidia-smi, it still says an X.org process is using the NVIDIA GPU. That's because X.org references the NVIDIA driver in /usr/share/X11/xorg.conf.d/10-nvidia.conf 

So, we will rename this file to have a different extension than .conf. Sometimes, X.org may recreate this file when you log out and log back in. So, we will create a black file and set its attribute to immutable, so no one other than root can edit it.

```
$ sudo mv /usr/share/X11/xorg.conf.d/10-nvidia.conf /usr/share/X11/xorg.conf.d/10-nvidia.conf.original
$ sudo touch /usr/share/X11/xorg.conf.d/10-nvidia.conf
$ sudo chattr +i /usr/share/X11/xorg.conf.d/10-nvidia.conf
```

Now, log out of X Windows and log back in, and run nvidia-smi again.

```
$ nvidia-smi
Mon Oct 28 11:00:45 2024       
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.183.06             Driver Version: 535.183.06   CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  NVIDIA GeForce GTX 1080        Off | 00000000:01:00.0 Off |                  N/A |
| 24%   37C    P8               7W / 180W |    110MiB /  8192MiB |      0%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+
                                                                                         
+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
|    0   N/A  N/A     25601    C+G   ...libexec/gnome-remote-desktop-daemon      105MiB |
+---------------------------------------------------------------------------------------+

```

X.org is no longer using the discrete GPU. Howver, the Gnome remote desktop daemon is still using it, consuming 105MiB of GPU RAM. How can we disable this? 

When I check which GPU is primary, I see that it's the Intel iGPU. 

```
$ switcherooctl list
Device: 0
  Name:        Intel® HD Graphics P4600/P4700
  Default:     yes
  Environment: DRI_PRIME=pci-0000_00_02_0

Device: 1
  Name:        NVIDIA Corporation GP104 [GeForce GTX 1080]
  Default:     no
  Environment: __GLX_VENDOR_LIBRARY_NAME=nvidia __NV_PRIME_RENDER_OFFLOAD=1 __VK_LAYER_NV_optimus=NVIDIA_only
```

The easy option is to remove the Gnome remote desktop daemon with the following:
```
$ systemctl --user disable --now gnome-remote-desktop.service
$
$ nvidia-smi
Mon Oct 28 11:11:53 2024       
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.183.06             Driver Version: 535.183.06   CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  NVIDIA GeForce GTX 1080        Off | 00000000:01:00.0 Off |                  N/A |
| 24%   34C    P8              13W / 180W |      0MiB /  8192MiB |      0%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+
                                                                                         
+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
|  No running processes found                                                           |
+---------------------------------------------------------------------------------------+

```

But in case we need Gnome remote desktop daemon, can we stop it from using the NVIDIA GPU?



