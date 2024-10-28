How-to use the integrated GPU for display and the NVIDIA discrete GPU for machine learning under Ubuntu

Based on https://askubuntu.com/questions/1061551/how-to-configure-igpu-for-xserver-and-nvidia-gpu-for-cuda-work/1099963#1099963

If you have both integrated (Intel, ASPEED, etc.) and discrete (NVIDIA) GPUs in your ML lab machine, you might want to use the integrated GPU for display and the NVIDIA GPU purely for ML to conserve all the NVIDIA GPU memory for your ML workloads.

By default, even if you connect your monitor only to the integrated GPU, X.org will use both GPUs. You can check if X.org is using the NVIDIA GPU with this command:
```
$ nvidia-smi
Mon Oct 28 10:33:45 2024       
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

If you run nvidia-smi, it still says an X.org process is using the NVIDIA GPU. 
