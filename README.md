# r8169

ASPM [1] is disabled for Realtek network chips by default when running linux. This leads to a vast increase in power consumption on certain systems. This patch enables ASPM on some chips to prolong battery life and decrease cpu temperature.

# Details

Due to the reported regressions that have been reported for the ASPM patch [2] it had been reverted several years ago. Nevertheless similar modifications on the driver work perfectly on my notebook by now. This repository contains the modificated driver thus ASPM can be enabled by a modprobe option. If there are more people that report it's running stable a pull request will be sent to linux upstream.
On my notebook this module decreased the system power consumption from 13W to 8W, as the cpu was prohibited from entering low package sleep states (PC6/PC7) before.

To check if you are affected of this problem, too you can run
```
watch -n1 sudo cpupower monitor
```

For a haswell system on idle output should look something like that:

|CPU | C3   | C6   | PC3  | PC6  | C7   | PC2  | PC7  | C0   | Cx   | Freq | POLL | C1-H | C1E- | C3-H | C6-H | C7s-
|----|------|------|------|------|-------|-----|------|------|----|---------|-----|------|------|------|------|----|
|   0|  0,28|  0,04|  6,84| 50,04| 94,53| 31,23|  0,00|  3,23| 96,77|  2469|  0,00|  0,73|  0,31|  0,31|  0,04| 95,21
|   4|  0,28|  0,04|  6,84| 50,04| 94,53| 31,23|  0,00|  0,70| 99,30|  2386|  0,00|  0,00|  0,00|  0,00|  0,00| 99,67
|   1|  0,21|  0,04|  6,84| 50,04| 97,47| 31,23|  0,00|  1,51| 98,49|  2773|  0,00|  0,00|  0,02|  0,19|  0,10| 98,53
|   5|  0,21|  0,04|  6,84| 50,04| 97,47| 31,23|  0,00|  0,72| 99,28|  2363|  0,00|  0,00|  0,02|  0,11|  0,00| 99,52
|   2|  0,16|  0,01|  6,84| 50,04| 97,35| 31,23|  0,00|  1,52| 98,48|  2493|  0,00|  0,00|  0,00|  0,21|  0,00| 98,59
|   6|  0,16|  0,01|  6,84| 50,04| 97,36| 31,23|  0,00|  0,57| 99,43|  2334|  0,00|  0,00|  0,00|  0,10|  0,00| 99,70
|   3|  0,24|  0,00|  6,84| 50,04| 95,19| 31,23|  0,00|  1,54| 98,46|  2435|  0,00|  0,00|  0,00|  0,10|  0,00| 98,64
|   7|  0,24|  0,00|  6,84| 50,04| 95,19| 31,23|  0,00|  2,26| 97,74|  2465|  0,00|  0,00|  0,01|  0,21|  0,00| 97,77 
   
States with a higher numbers are deeper thus the cores should be in C7s the package (the states that start with a 'P') should be in PC6 or higher for more than 50%. PC7 is only used when display is turned off.

As long as a single device or prevents the cpu from sleeping the result looks similar to that:

|CPU | C3   | C6   | PC3  | PC6  | C7   | PC2  | PC7  | C0   | Cx   | Freq | POLL | C1-H | C1E- | C3-H | C6-H | C7s-
|----|------|------|------|------|-------|-----|------|------|----|---------|-----|------|------|------|------|----|
   0|  0,11|  0,10| 76,37|  0,00| 95,81| 17,02|  0,00|  1,90| 98,10|  2484|  0,00|  1,27|  0,30|  0,11|  0,10| 96,25
   4|  0,11|  0,10| 76,37|  0,00| 95,81| 17,02|  0,00|  0,58| 99,42|  2707|  0,00|  0,00|  0,00|  0,04|  0,00| 99,65
   1|  0,03|  0,08| 76,37|  0,00| 98,18| 17,02|  0,00|  0,83| 99,17|  2598|  0,00|  0,00|  0,10|  0,00|  0,08| 99,27
   5|  0,03|  0,08| 76,37|  0,00| 98,18| 17,02|  0,00|  0,49| 99,51|  2716|  0,00|  0,00|  0,01|  0,03|  0,00| 99,73
   2|  0,06|  0,12| 76,37|  0,00| 97,86| 17,02|  0,00|  1,25| 98,75|  2497|  0,00|  0,00|  0,00|  0,07|  0,06| 98,78
   6|  0,06|  0,12| 76,37|  0,00| 97,86| 17,02|  0,00|  0,59| 99,41|  2661|  0,00|  0,00|  0,00|  0,00|  0,06| 99,60
   3|  0,00|  0,07| 76,37|  0,00| 98,88| 17,02|  0,00|  0,41| 99,59|  2777|  0,00|  0,00|  0,00|  0,00|  0,07| 99,80
   7|  0,00|  0,07| 76,37|  0,00| 98,88| 17,02|  0,00|  0,35| 99,65|  2748|  0,00|  0,00|  0,00|  0,00|  0,00| 99,92 
   
PC6 and higher isn't used at all even when all cores are at a deep sleep state.

To narrow the problems that prevent the cpu from sleeping you can run
```
sudo lspci -vvv | fgrep "ASPM Disabled" 
```
To check if there are pcie devices that don't use ASPM.
If it's only the realtek network card please use this driver to check if the problem is solved then.

# Hints

You can't enable ASPM. The only thing you can do is NOT DISABLE it. That means, if the wrong driver, or the driver with wrong options was loaded once a COLD reboot has to be done to reset the pci device and to be able to use ASPM again. Only loading the other module won't work as less as only a warm reboot ` sudo reboot`.

# Why don't we use r8168

This driver, provided by Realtek itself, worked for me to prevent the ASPM bug. Nevertheless there had been system freezes quite regularly (every 30 seconds for about 2 seconds) thus this module was unusable for me.

# Installation
Please remove/save your old r8169 module before installing this module. Its in 
```
/var/lib/modules/$(uname -r)/kernel/drivers/net/ethernet/realtek/r8169.ko.gz
```
for me.

```
make
sudo make install
sudo depmod -a
sudo modprobe r8169
 ```
 

[1] https://en.wikipedia.org/wiki/Active_State_Power_Management
[2] https://lkml.org/lkml/2012/11/1/216
