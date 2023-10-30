---
title: Configuring IP aliasing
excerpt: Find out how to add Additional IP addresses to your VPS configuration
updated: 2023-10-25
---

> [!primary]
>
> Since October 6th, 2022 our service "Failover IP" is named [Additional IP](https://www.ovhcloud.com/asia/network/additional-ip/). This renaming has no effect on its technical features.
>

## Objective

IP aliasing refers to a special network configuration for certain OVHcloud services. Additional IPs allow you to associate multiple IP addresses with a single network interface.

**This guide explains how to add Additional IP addresses to your network configuration.**

> [!warning]
>OVHcloud is providing you with services for which you are responsible, with regard to their configuration and management. You are therefore responsible for ensuring they function correctly.
>
>This guide is designed to assist you in common tasks as much as possible. Nevertheless, we recommend that you contact a [specialist service provider](https://partner.ovhcloud.com/asia/directory/) and/or discuss the issue with our community on https://community.ovh.com/en/ if you have difficulties or doubts concerning the administration, usage or implementation of services on a server.
>

## Requirements

- A [Virtual Private Server](https://www.ovhcloud.com/asia/vps/) in your OVHcloud account
- An [Additional IP address](https://www.ovhcloud.com/asia/bare-metal/ip/) or an Additional IP block
- Administrative access (root) via SSH or GUI to your server
- Basic networking and administration knowledge

## Instructions

The following sections contain the configurations for the most commonly used distributions/operating systems. The first step is always to log in to your server via SSH or a GUI login session (RDP for a Windows VPS). The examples below presume you are logged in as a user with elevated permissions (Administrator/sudo).

> [!primary]
>
Concerning different distribution releases, please note that the proper procedure to configure your network interface as well as the file names may have been subject to change. We recommend to consult the manuals and knowledge resources of the respective OS versions if you experience any issues.
> 

**Please take note of the following terminology that will be used in code examples and instructions of the guide sections below:**

|Term|Description|Examples|
|---|---|---|
|ADDITIONAL_IP|An Additional IP address assigned to your service|169.254.10.254|
|NETWORK_INTERFACE|The name of the network interface|*eth0*, *ens3*|
|ID|ID of the IP alias, starting with *0* (depending on the number of additional IPs there are to configure)|*0*, *1*|

As an example, we will use 123.123.123.123/32 as the IP block, with eth0 and ens3 as the network interfaces. We will also use “nano” as the editing tool.

### Debian 10/11

#### Step 1: Disable automatic network configuration

The main configuration file is located in `/etc/network/interfaces.d/`. In this example it is called "ifcfg-eth0".

Here, the Additional IPs are configured directly in the main configuration file. This is done by creating "virtual interfaces or ethernet aliases" (example, eth0:0, eth0:1 etc...).

Open the following file path with a text editor:

```bash
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```
Enter the following line, then save and exit the editor.

```bash
network: {config: disabled}
```
Creating this configuration file will prevent changes to your network configuration from being made automatically.

#### Step 2: Edit the network configuration file

You can verify your network interface name with this command:

```bash
ip a
```

Open the network configuration file for editing with the following command:

```bash
sudo nano /etc/network/interfaces.d/50-cloud-init
```
Then add the following lines:

```bash
auto NETWORK_INTERFACE:ID
iface NETWORK_INTERFACE:ID inet static
address ADDITIONAL_IP
netmask 255.255.255.255
```

For example, if we want to configure IP block 169.254.10.254/32 with network interface eth0, we have the following:

```bash
auto eth0:0
iface eth0:0 inet static
address 169.254.10.254
netmask 255.255.255.255
```

#### Step 3: Restart the interface

Apply the changes with the following command:

```bash
sudo systemctl restart networking
```

### Ubuntu (20.04 - 23.04) & Debian 12

The configuration file for your Additional IP addresses is located in `/etc/netplan/`. In this example it is called "50-cloud-init.yaml". Before making changes, verify the actual file name in this folder. Each Additional IP address will need its own line within the file.

#### Step 1: Disable automatic network configuration

Open the following file path with a text editor:

```bash
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
```
Enter the following line, then save and exit the editor.

```bash
network: {config: disabled}
```
Creating this configuration file will prevent changes to your network configuration from being made automatically.

#### Step 2: Edit the configuration file

You can verify your network interface name with this command:

```bash
ip a
```

Open the network configuration file for editing with the following command:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

As Netplan does not support virtual interfaces or ethernet aliases (for example ens3:0, ens3:1), all Additional IPs are configured on a single network interface.

Please note that with the recent distributions, it is possible that the IPV6 is automatically configured on your service. Do not modify the existing lines in the configuration file, simply replace ADDITIONAL_IP with your own values.

```yaml
network:
    version: 2
    ethernets:
        NETWORKINTERFACE:
            accept-ra: false
            addresses:
            - 2607:xxxx:xxx:xxxx::xxc/56
            - ADDITIONAL_IP/32
            dhcp4: true
            match:
                macaddress: fa:xx:xx:xx:xx:50
            mtu: 1500
            nameservers:
                addresses:
                - 213.186.33.99
                search: []
            routes:
            -   to: ::/0
                via: 2607:xxxx:xxx:xxxx::1
            set-name: ens3
```

> [!warning]
>
> It is important to respect the alignment of each element in this file as represented in the example above. Do not use the tab key to create your spacing.
>

Save and close the file.

#### Step 3: Apply the new network configuration

You can test your configuration using this command:

```bash
sudo netplan try
```

If it is correct, apply it using the following command:

```bash
sudo netplan apply
```

Repeat this procedure for each Additional IP address.

#### Assign an Additional IP temporarily

It is possible to asign an Additional IP temporarily, however, note that the configuration will be lost when the server is rebooted. This process also allows you to label the IP with a virtual interface, even if the IP is configured on the main network interface. Simply replace `ADDITIONAL_IP`, `NETWORK_INTERFACE` and `NETWORK_INTERFACE:ID` with your own values.

```bash
sudo ip address add ADDITIONAL_IP/32 dev NETWORK_INTERFACE label NETWORK_INTERFACE:ID
```

For example, if we want to temporarily assign IP block `169.254.10.254/32` to our server with network interface **ens3**:

```bash
sudo ip address add 169.254.10.254/32 dev ens3 label ens3:0
```

If we run `ip a`, we can see the IP is configured on the main interface with virtual interface **ens3:0**:

```bash
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether fa:16:3e:a7:13:7d brd ff:ff:ff:ff:ff:ff
    inet 158.xx.xx.xx/32 scope global dynamic ens3
       valid_lft 86179sec preferred_lft 86179sec
    inet 169.254.10.254/32 scope global ens3:0
       valid_lft forever preferred_lft forever
```

### Windows Server 2016

#### Step 1: Verify the network configuration

Right-click on the `Start Menu`{.action} button and open `Run`{.action}.

Type `cmd` and click `OK`{.action} to open the command line application.

![cmdprompt](images/vps_win07.png){.thumbnail}

In order to retrieve the current IP configuration, enter `ipconfig` at the command prompt.

![check main IP configuration](images/image1-1.png){.thumbnail}

#### Step 2: Change the IPv4 Properties

Now you need to change the IP properties to a static configuration.

Open the adapter settings in the Windows control panel and then open the `Properties`{.action} of `Internet Protocol Version 4 (TCP/IPv4)`{.action}.

![change the ip configuration](images/image2.png){.thumbnail}

In the IPv4 Properties window, select `Use the following IP address`{.action}. Enter the IP address which you have retrieved in the first step, then click on `Advanced`{.action}.

#### Step 3: Add the Additional IP in the "Advanced TCP/IP Settings"

In the new window, click on `Add...`{.action} under "IP addresses". Enter your Additional IP address and the subnet mask (255.255.255.255).

![advance configuration section](images/image4-4.png){.thumbnail}

Confirm by clicking on `Add`{.action}.

![Additional IP configuration](images/image5-5.png){.thumbnail}

#### Step 4: Restart the network interface

Back in the control panel (`Network Connections`{.action}), right-click on your network interface and then select `Disable`{.action}.

![disabling network](images/image6.png){.thumbnail}

To restart it, right-click on it again and then select `Enable`{.action}.

![enabling network](images/image7.png){.thumbnail}

#### Step 5: Check the new network configuration

Open the command prompt (cmd) and enter `ipconfig`. The configuration should now include the new Additional IP address.

![check current network configuration](images/image8-8.png){.thumbnail}

### cPanel (CentOS 7) / Red Hat derivatives

The main configuration file is located in `/etc/sysconfig/network-scripts/`. In this example it is called "ifcfg-eth0". Before making changes, verify the actual file name in this folder. 
Since the configuration of additional IPs is not made within the main configuration file, we have to create a new configuration file each assitional IP with "virtual interfaces or ethernet alias". 

To achieve this, we simply add a consecutive number to the interface name, starting with a value of 0 for the first alias. In our case, our first alias for eth0 is eth0:0.

#### Step 1: Edit the network configuration file

First, verify your network interface name with this command:

```bash
ip a
```

Once you have identified your network interface, create a network configuration file for editing. Replace `NETWORK_INTERFACE` with your interface name and `ID` with the first value:

```bash
sudo nano /etc/sysconfig/network-scripts/ifcfg-NETWORK_INTERFACE:ID
```

For example, with a network interface named "eth0", we will create the configuration file below:

```bash
sudo nano /etc/sysconfig/network-scripts/ifcfg-eth0:0
```

Then add these lines, replacing `NETWORK_INTERFACE:ID` and `ADDITIONAL_IP` with your own values.

```bash
DEVICE=NETWORK_INTERFACE:ID
BOOTPROTO=static
IPADDR=ADDITIONAL_IP
NETMASK=255.255.255.255
BROADCAST=ADDITIONAL_IP
ONBOOT=yes
```

Example:

```bash
DEVICE=eth0:0
BOOTPROTO=static
IPADDR=123.123.123.123
NETMASK=255.255.255.255
BROADCAST=123.123.123.123
ONBOOT=yes
```

#### Step 2: Restart the interface

Apply the changes with the following command:

```bash
sudo systemctl restart network.service
```

### Plesk

#### Step 1: Access the Plesk IP management section

In the Plesk control panel, choose `Tools & Settings`{.action} from the left-hand sidebar.

![acces to the ip addresses management](images/pleskip1.png){.thumbnail}

Click on `IP Addresses`{.action} under **Tools & Resources**.

#### Step 2: Add the additional IP information

In this section, click on the button `Add IP Address`{.action}.

![add ip information](images/pleskip2-2.png){.thumbnail}

Enter your Additional IP in the form `xxx.xxx.xxx.xxx/32` into the field "IP address and subnet mask", then click on `OK`{.action}.

![add ip information](images/pleskip3-3.png){.thumbnail}

#### Step 3: Check the current IP configuration

Back in the section "IP Addresses", verify that the Additional IP address was added correctly.

![current IP configuration](images/pleskip4-4.png){.thumbnail}

### Troubleshooting

First, restart your server from the command line or its GUI. If you are still unable to establish a connection from the public network to your alias IP and suspect a network problem, you need to reboot the server in [rescue mode](/pages/bare_metal_cloud/virtual_private_servers/rescue). Then you can set up the Additional IP address directly on the server.

Once you are connected to your server via SSH, enter the following command:

```bash
ifconfig ens3:0 ADDITIONAL_IP netmask 255.255.255.255 broadcast ADDITIONAL_IP up
```

To test the connection, simply ping your Additional IP from the outside. If it responds in rescue mode, that probably means that there is a configuration error. If, however, the IP is still not working, please inform our support teams by creating a support request in your [OVHcloud Control Panel](https://ca.ovh.com/auth/?action=gotomanager&from=https://www.ovh.com/asia/&ovhSubsidiary=asia) for further investigations.
 
## Go further

[Activating Rescue Mode on VPS](/pages/bare_metal_cloud/virtual_private_servers/rescue)

Join our community of users on <https://community.ovh.com/en/>.
