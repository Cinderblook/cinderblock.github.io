# Overview

I stumbled upon a collection of affordable Dell Optiplex 7060 Micros, equipped with exceptional specs for a homelab cluster ESXi environment. However, I faced a challenge in having only one Ethernet interface on each host, when I needed dedicated ISCSI storage on a separate interface. I found a solution using custom drivers for specific USB to Ethernet adapters, supported unofficially by VMware staff.

## Choosing the Adapters
Make sure to select a USB to Ethernet adapter that's compatible. See a list of [officially unsupported but working adapters on VMware's site](https://flings.vmware.com/usb-network-native-driver-for-esxi#requirements), or choose the one I used [from Amazon](https://www.amazon.com/dp/B00YUU3KC6?psc=1&ref=ppx_yo2ov_dt_b_product_details).
   - The adapter I chose, is a TP-Link E300, with a Realtek RTL8153 chip, that supports 10/100/1000MB speeds.

## Physical and SSH Access
You'll need physical access to install the adapters and connect them to the network (either directly to the ISCSI interface or through a dedicated VLAN). On the ESXi host(s), go to https://ip.address:443 to enable SSH if needed.

## Installing the VMware Driver
- Download the driver from [VMware](https://flings.vmware.com/usb-network-native-driver-for-esxi#summary). Ensure the correct version based on the ESXi hosts.
    - Do a quick check on the host with `vmware -v`
- Transfer it to the host(s) and install:

    ```bash
    scp /file/location <user>@<ip-esxi-host>:/location/to/put/file
    ssh <user>@<ip>
    esxcli software component apply -d /path/to/the component zip
    reboot
    ```

## Configuring USB to Ethernet Usage
- After rebooting, create a new NIC on the ESXi host or in vSphere.
- Confirm proper configuration within the GUI, then run within the SSH console:

    ```bash
    esxcli system module parameters set -p "usbBusFullScanOnBootEnabled=1" -m vmkusb_nic_fling
    ```

- Edit the local.sh file:

    ```bash
    vi /etc/rc.local.d/local.sh
    ```
    
    Add this snippet:
    
    ```bash
    vusb0_status=$(esxcli network nic get -n vusb0 | grep 'Link Status' | awk '{print $NF}')
    count=0
    while [[ $count -lt 20 && "${vusb0_status}" != "Up" ]]
    do
        sleep 10
        count=$(( $count + 1 ))
        vusb0_status=$(esxcli network nic get -n vusb0 | grep 'Link Status' | awk '{print $NF}')
    done

    esxcfg-vswitch -R
    ```

- Add the provided code, and if you need support for multiple adapters, see more details [here](https://flings.vmware.com/usb-network-native-driver-for-esxi#instructions).

## Conclusion
Verify the network connectivity, and you'll now have a dedicated management port and an additional ISCSI NAS port for HA failover. Additionally, if you are like me, an affordable solution created to creating additional network interfaces on a Dell Optiplex 7060 Micro!
