# Configure 802.1X Client Auth Mechanism for Routers at IIIT Hyderabad

**Note:** While this tutorial focuses on configuring Wi-Fi routers to work with LAN ports at IIIT Hyderabad, the same procedure can be applied to any enterprise network using PEAP and MSCHAPV2 authentication.

## Pre-Requisites:

1. **Check Firmware Compatibility:**
    - Check and note your Wi-Fi Router's model. For example: ```TP-Link TL-WR850N v2```. Ensure to note down any version numbers too.
   - Go to [OpenWRT's supported devices page](https://wiki.openwrt.org/toh/start) and check for your routers name in the list to confirm if your router is compatible with OpenWRT.

2. **Environment**:

    Try out these steps from any system (Linux/ Windows) with a terminal that supports scp, ssh and vim. 

    **Note**: You can use any alternatives for the mentioned packages like nano, sftp etc.

3. **Download Firmware:**

    - Open the corresponding device page for your router from the list mentioned above.
   - Download the appropriate firmware image for your device from OpenWRT. There are two types here:
        1. **Install Image**: "Install" with "factory image" is meant for the first installation, so that the flashing is done with the router's OEM flashing routine.
        2. **Upgrade Image**: This image is meant for a router already running OpenWrt, so that the new image is flashed from the existing OpenWrt.
    - It is advisable to download both these images along with the original firmware image (Stock OEM image)
    - Also note down the ```current release``` version of the downloaded packages. For e.g. ```22.03.5``` etc


## Method:



1. **Flash Firmware:**
   
    - Ideally, if you would like to install OpenWRT from the default router's firmware, then go ahead with the Install Image
    - Navigate to your router's local IP (Figure this out as different models have different addresses. Usually they are ```192.168.1.1```, ```192.168.0.1``` and so on) and select firmware upgrade and upload the install image file with ```.bin``` extension.
   - Wait for this firmware to be loaded onto your router.
   
   **Note**: This way of flashing might not work for few routers due to various reasons like firmware change not being allowed if the router is supplied by an internet vendor like ACT. In this case, follow the steps mentioned in ```Recovery``` and ```Flash/Recovery``` on the OpenWRT's device page for your router to load OpenWRT into the router.

3. **Connect to the Router:**
   - Connect your computer to the router using an Ethernet cable.
   - Open a web browser and visit: `192.168.1.1`.

4. **Set Up Password:**
   - Log in to the router.
   - Set an admin password for your router. Remember this password as it will be used to ssh later.

5. **Network Configuration:**
   - Go to **Network > Interfaces** in the router settings.
   - Delete "WAN6" profile.

6. **SSH to the Router:**
   - Use an SSH client (can be done from terminal) to connect to your router with the following details:
     - Username: `root`
     - IP Address: `192.168.1.1`
     
     The command on terminal would look like: ```ssh root@192.168.1.1```

     When asked for a password, enter the admin password created earlier. You would now get access to the router's system through the terminal.
   - Create a file named `/etc/config/wpa.conf` with the following content (The command on terminal would look like: ```vi /etc/config/wpa.conf``` ):
     ```plaintext
     ctrl_interface=/var/run/wpa_supplicant
     ctrl_interface_group=root
     ap_scan=0
     network={
         key_mgmt=IEEE8021X
         eap=PEAP
         identity="your_username_here"
         password="your_password_here"
         phase1="peaplabel=0"
         phase2="auth=MSCHAPV2"
     }
     ```
     Note: Replace "your_username_here" and "your_password_here" with your IIIT email ID and 802.1x password within the quotes.

7. **Create Startup Script:**
   - Type ```ifconfig``` to list down all the network interfaces
   - Lookup for interface which has ip address of type ```10.x.xx.xxx```, for e.g.: ```10.2.40.121``` and note it down. The interface names could be something like ```eth1```, ```eth0```, ```eth0.2``` etc depending on your router.
   - Create another file named `/etc/init.d/wpa` with the following content ((The command on terminal would look like: ```vi /etc/init.d/wpa``` )):
     ```plaintext
     #!/bin/sh /etc/rc.common
     # Example script
     # Copyright (C) 2007 OpenWrt.org
     START=99
     start() {
         echo start
         date --set="2016-04-13 00:00:00" 
         wpa_supplicant -D wired -i eth1 -c /etc/config/wpa.conf &
         udhcpc -i eth1 &
     }
     ```
     **Note**: 
     - Replace the word ```eth1``` on the last 2nd and 3rd lines in the script with the interface name noted down in the earlier step
     - Adjust the date if needed (Mostly not required).


8. **Make Startup Script Executable:**
   - Give executable permissions to the script by running this command on the terminal:
     ```
     chmod a+x /etc/init.d/wpa
     ```
   - Enable the script to run on startup:
     ```
     /etc/init.d/wpa enable
     ```

9. **Identify ISA and Download Package:**
   - Run the command to find your routers's ISA(system architecture) and note it down (for e.g.: ```mipsel_24kc```):
     ```
     opkg print-architecture
     ```
    - Open the [OpenWRT's download page](https://archive.openwrt.org/releases/)  to download a package to complete our process
   - Select the release number of your downloaded image of OpenWRT (Refer to [Pre-requsities](#pre-requisites) last point)
   - Select ```packages``` folder option
   - Select the architecture name that was earlier found    corresponding to your router's architecture found.
   - Select the ```base``` folder option
   - Search for packages with name just ```wpad```. There can be many with names ```wpad-mini```, ```wpad-wolfssl``` etc, but just look for ```wpad_$version_$arch.ipk```. If there are multiple files, that indicates different versions, download the latest one.

10. **Copy the package to the router**
    - Open another terminal
    - Copy the downloaded file to the router with the following command:
    ```
    scp ~/Downloads/wpad_$version_$arch.ipk root@192.168.1.1:/tmp/
    ```
    Feel free to change the first argument(```~/Downloads/wpad_$version_$arch.ipk```) to the actual location of the downloaded package file.
    - To verify if the file has been transferred, in the first terminal opened earlier, type ```ls /tmp/``` and check if the file exists.

10. **Remove and Install Packages:**
    - Remove the `wpad-mini` package:
      ```
      opkg remove wpad-mini
      ```
    - Install the downloaded `.ipk` package file:
      ```
      opkg install /tmp/wpad_$version_$arch.ipk
      ```
    - Incase if you encounter an error that there is a conflicting package, remove that package and try installing again from the previous step

11. **Edit DHCP Configuration:**
    - Edit the file `/etc/config/dhcp` (terminal command would look like ```vi /etc/config/dhcp```).
    - Look for this line and change the value to '0':
      ```
      option rebind_protection '1'
      ```

12. **Reboot Your Device:**
    - Reboot your router.

13. **Enable Wireless:**
    - On a browser, goto ```192.168.1.1```
    - In the router settings, go to the Wireless section.
    - Enable the wireless feature if it's not already enabled.
    - Set a proper SSID and password for your wireless network.
    - The network security can be anything, with WPA2-PSK being suggested.

## Debugging:

Incase if you are not able to access the internet after the above steps, it can be due to multiple reasons.

1. Memory on your router is not sufficient to compile the packages. In that case, follow the steps in [Section 2](https://self-help.iiit.ac.in/wiki/index.php/Configure_802.1X_Client_Auth_Mechanism_for_Routers) located at the bottom of the Self-Help page.
2. Wrong user credentials or interface is provided. To check this, do the following steps:

    - SSH into the router as mentioned in Step 5 earlier.
    - In the terminal, type the following command: 
    ```
    wpa_supplicant -D wired -i eth1 -c /etc/config/wpa.conf
    ``` 
    , where eth1 should be replaced with the suitable interface found in Step 6 earlier.
    
    - The above command tries to authenticate with the creds provided. Check for the server response on the terminal screen. For a successful authentication, you will receive a positive response message. If the interface is wrong, you will not receive any acknowledgements from the server. If the interface is right and the IIIT credentials are wrongly provided, you will receive an acknowledgement that reads IIIT's name. However, your connection will not succeed.
    - A successful acknowledgement message would read like this:
      ```
      Successfully initialized wpa_supplicant
        wan: Associated with 01:XX:c2:00:00:XX
        wan: CTRL-EVENT-SUBNET-STATUS-UPDATE status=0
        wan: CTRL-EVENT-EAP-STARTED EAP authentication started
        wan: CTRL-EVENT-EAP-PROPOSED-METHOD vendor=0 method=25
        wan: CTRL-EVENT-EAP-METHOD EAP vendor 0 method 25 (PEAP) selected
        wan: CTRL-EVENT-EAP-PEER-CERT depth=0 subject='C=IN, ST=Telangana, O=IIIT Hyderabad, CN=IIIT-H Radius Server Certificate/emailAddress=sysadmins@****.iiit.ac.in' hash=****
        wan: CTRL-EVENT-EAP-PEER-CERT depth=1 subject='C=IN, ST=Telangana, L=Gachibowli, O=IIIT Hyderabad, CN=IIIT-H Radius Server CA/emailAddress=sysadmins@****.iiit.ac.in' hash=****
        EAP-MSCHAPV2: Authentication succeeded
        EAP-TLV: TLV Result - Success - EAP-TLV/Phase2 Completed
        wan: CTRL-EVENT-EAP-SUCCESS EAP authentication completed successfully
        wan: CTRL-EVENT-CONNECTED - Connection to 01:XX:c2:00:00:XX completed [id=0 id_str=]
    ```

## Credits:

This information is from multiple online resources, including [Self-Help Portal](self-help.iiit.ac.in). I would also like to thank my buddy [Harsha Vardhan](https://www.linkedin.com/in/harshavardhannarla/) for helping me out.
