# Switch Configuration

## Basic terminology

User mode (>)
Exec mode (#)
Global config mode ((config)#)
Interface config mode ((config-if)#) 
Line config ((config-line)#)

## Useful commands

#show interface, will display everything I am listing below.
	- brief
	- fastEthernet 1/0/1, will display status and statistics of a specific interface.
	- status, will show the status of all switch interfaces.
	- vlan 1, will show the status, statistics, and management IP of a specified vlan.

## Basic Switch Configuration

### SCREECH! What's my password?

I ran into a roadblock. A few months ago, I logged into this swtich and set the enable password. However, I forgot what I set the password as...

After reading through my CCNA book, I found that I set the password to "love" after copying what the book provided. I changed this and now store the enable and line passwords in a secure password vault.

### Password configuration

```BASH
jbone-switch#
jbone-switch#conf t
jbone-switch(config)#line con 0
jbone-switch(config-line)#login local
jbone-switch(config-line)#username ** secret **
jbone-switch(config-line)#ine vty 0 15
jbone-switch(config-line)#login local
jbone-switch(config-line)#username ** secret **
jbone-switch(config-line)#^Z
jbone-switch#
jbone-switch#conf t
jbone-switch(config)#enable secret **
```

I confirmed my changes by logging out of the console and logging back in. However, you can check the changes buy running `# show running-config *brief*`.

#### Setting exec-timeout

This is a command used for the virtual terminal lines which allow us to increase/decrease the inactivity timer for Telnet/SSH connections.

```BASH
jbone-desktop(conifg)#line vty 0 15
jbone-desktop(conifg-line)#exec-timeout 120 // sets 3 hours as the inactivity timer
```

### DISABLING IP-LOOKUP & Logging

This has to be one of the most important steps, because without it any mistyped command is interpretted as a DNS query. 

```BASH
jbone-switch(config)#no ip domain-lookup
```

The syslog messages can also be quite annoying as they interrupt you as you are typing commands. To disable them, you'll need to configure each line to display them after 'show' commands. 

``` BASH
jbone-desktop(conifg)#line con 0
jbone-desktop(conifg-line)#logging synchronous
jbone-desktop(conifg-line)#line vty 0 15
jbone-desktop(conifg-line)#logging synchronous
```

### Setting up a management IP

Assigning a management IP allows you to Telnet/SSH from within the LAN. Note that the IP must be associated with an active VLAN. If not, no traffic will be able to route to it. Even if a host is in a different VLAN, traffic can be forwarded to the default router and bounced to the corrrect address on the LAN.

Another important thing to remember is that when using a management IP a virtual interface (SVI) is created so that a physical port isn't assigned an address.

```BASH
#conf t
jbone-desktop(conifg)#interface vlan 1
jbone-desktop(conifg-if)#ip address 192.168.1.250 255.255.255.0
jbone-desktop(conifg-if)#no shutdown
jbone-desktop(conifg-if)#exit
jbone-desktop(conifg)#ip default-gateway 192.168.1.1
```

#### Issues I ran into

As I write this, I am RDPing into a host that is connected to the switch. The host is connected via a console cable, so no physical machines are connected to any of the switch interfaces. Becuase of this setup, my vlan 1 status shows as being 'up', but the line protocol is 'down'. 

After further digging, it seems like the line protocol is 'down' as a result of no/failed physical connection. I assume, that after plugging a device in to the switch the line protocol will show as 'up'.

```BASH
jbone-switch#show interface status
Port      Name               Status       Vlan       Duplex  Speed Type
Fa1/0/1                      notconnect   1            auto   auto 10/100BaseTX
...
Fa1/0/48                     notconnect   1            auto   auto 10/100BaseTX
Gi1/0/1                      notconnect   1            auto   auto Not Present
...
Gi1/0/4                      notconnect   1            auto   auto Not Present
jbone-switch#
jbone-switch#show interfaces vlan 1
Vlan1 is up, line protocol is down
  Hardware is EtherSVI, address is a40c.c39b.0240 (bia a40c.c39b.0240)
  Internet address is 192.168.1.250/24
```

## Interface Configuration

Interfaces refer to the physical ports on a switch and each port can be configured by interface subcommands entered in the `config-if#` prompt. Entering ranges of interfaces makes ocnfiguration easier, but only works if the interfaces are the same type and are numbered consecutively.

### Status codes & nonworking states

This section will hone in on codes in the line status and protocol status of the `show interfaces` exec command. Line status refers to the physical interface (layer 1) of a switch, which can have different status codes such as "admin down", "down", and "up". "Admin down", as we've seen, comes from the `shut` interface command. The "down" and "up" are a result of the interface receiving power from a cable being connected to it. Protocol status refers to the data link (layer 2) established between two interfaces, and can either be in a down or up state. 

admin down/down = 	disabled 	- shutdown configured on interface
down/down 		= 	notconnect	- no cable, bad cable, wrong pinouts, neighbor device is: 1. off, 2. shutdown, 3. error disabled
up/down 		= 	notconnect 
down/err-down 	= err-disabled	- port security disabled the interface
up/up 			= connected

down/down coudl also occur if there is a speed mismatch. This highlights something interesting, if two interfaces a talking at different speeds the interfaces will be in a down/down state. Whereas, they have matching speeds, but a duplex-mismatch will lead to an up/up state. In this case, they can still talk, but performance is impacted. Additionally, the `show` command may not give any clues of the impact to performacne, as it will just say "up/up".

#### More on bad cables

1. EMI from other electrical equipment.
2. Degraded signal due to bending, people rolling over with chair, etc...
3. Macro-bending where an optical cable is bent too much.

#### Counter variables

Runts - Frames that don't meet the minimum 64 byte frame size. Can be caused by collisions.
Giants - Frames that exceed the 1500 byte max frame size.
Input Errors - counts runts, giansts, crc, etc.
CRC - Frames that did not pass FCS, meaning they were corrupted in transit. **These often occur due to high EMI.**
Frame - have an illegal format.
Packets Output - Number of frames forwarded on an interface.
Output Errors - Number of frames that interface attempted to output, but ran into issues.
Collisions - Counts all collisions that occurs while sending frames.
Late Collisions - Collisions that occur after the 64th byte has been transmitted. **These often occur due to duplex mismatch.**

Note that many of these counters occur when CSMA/CD is enabld in a half-duplex setting.

```BASH
jbone-switch#show interfaces vlan 1
...
	 0 packets input, 0 bytes, 0 no buffer
     Received 0 broadcasts (0 IP multicasts)
     0 runts, 0 giants, 0 throttles
     0 input errors, 0 CRC, 0 frame, 0 overrun, 0 ignored
     11 packets output, 764 bytes, 0 underruns
     0 output errors, 0 interface resets
     0 output buffer failures, 0 output buffers swapped out
```

### Speed, Duplex, & Description

By default, interfaces will autonegotiate the spped and duplex mode, but in the chance that the interfaces choose a less optimal configuration you can manually set the settings.

```BASH
jbone-desktop#conf t
jbone-desktop(conifg)#interface range fa1/0/1-48
jbone-desktop(conifg-if)#speed {auto|10|100|1000}
jbone-desktop(conifg-if)#duplex {auto|full|half}
jbone-desktop(conifg-if)#decription some random text
jbone-switch#show interface status
Port      Name               Status       Vlan       Duplex  Speed Type
Fa1/0/1   some random text   notconnect   1            full   100  10/100BaseTX
...
Fa1/0/48  some random text   notconnect   1            full   100  10/100BaseTX
Gi1/0/1                      notconnect   1            auto   auto Not Present
...
Gi1/0/4                      notconnect   1            auto   auto Not Present
```

Note that all of these configurations could be reversed to their defaults by prepending the command with no. Ex, `no description` will clear the configuration we made for the description.

Note that configuring these commands will disable IEEE autonegotation.

#### Autonegotiation interface status

If two interfaces finish autonegotiation, the `show interface status` command will have slightly different output. Anything starting with "a-" will indicate that the interface autonegotiated to get its configuration.

```BASH
jbone-switch#show interface status
Port      Name               Status       Vlan       Duplex  Speed Type
Fa1/0/1   some random text   notconnect   1          a-full   a-100 10/100BaseTX
... 
```

Note that "half" and "a-half" are two different things. One was manually configured, while the other was negotiated.

#### Autonegotiation concepts

Autonegotiation is defined by IEEE 802.3u and allows interfaces to communicate through an out-of-bounds frequency to determine the best speed to communicate at. This prevents two issues, the first being one device sending at 100 Mbps and the other device receiving at 1000 Mbps. Additionally, this helps prevent the headache of constant hardware upgrades. If one end had their hardware upgraded, the other end isn't forced to upgrade. For this second reason alone, highlights how/why autonegotiation is a very important feature in switches.

What do we do in the event only one side is autonegotiating? IEEE recommends sending data at 10 Mbps at half-duplex (half-duplex if 10/100 and full if 1000). However, if the Cisco switch notices there is no autonegotiation it will use smart logic, where it can sense the speed interfaces send at. Knowing this, Cisco switches will try to sense the speed an interface is sending data at. If this isn't possible, it defaults to the IEEE standard.

Autonegotiation "fails" if the other interface doesn't support or enable autonegotiation.

Switches could guess the correct speed, but may get a duplex mismatch. Duplex mismatch can occur due to IEEE defaults, where one host is on full and the other (switch) is on half. In this case, if the switch sends data and gets interrupted, it will stop, back off, and retransmit. The interface is working, but has very bad efficiency.

Hubs are Layer 1 devices that do not autonegotiate, for this reason interfaces side with IEEE defaults of 10 Mbps and half-duplex.

### Interface shutdown configuration

```BASH
jbone-switch#show interfaces fastEthernet 1/0/1
FastEthernet1/0/1 is down, line protocol is down (notconnect) // down & down (nothing connected)
  Hardware is Fast Ethernet, address is a40c.c39b.0203 (bia a40c.c39 b.0203)
...
jbone-switch#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
jbone-switch(config)#interface
jbone-switch(config)#interface fastEthernet 1/0/1
jbone-switch(config-if)#shut
jbone-switch(config-if)#
1d04h: %LINK-5-CHANGED: Interface FastEthernet1/0/1, changed state to administratively down
jbone-switch(config-if)#^Z
jbone-switch#show interfaces fastEthernet 1/0/1
FastEthernet1/0/1 is administratively down, line protocol is down (disabled) // admin down & disabled
  Hardware is Fast Ethernet, address is a40c.c39b.0203 (bia a40c.c39b.0203)
...
```

## Setting up VLANs

In my network I plan to setup VLANs for different types of devices. This will require just statically assigning a VLAN number to an access port. 

### Creating & assigning a vlan

```BASH
jbone-switch#
jbone-switch#conf t
Enter configuration commands, one per line.  End with CNTL/Z.
jbone-switch(config)#vlan 2
jbone-switch(config-vlan)#name WindowsVLAN
jbone-switch(config-vlan)#vlan 3
jbone-switch(config-vlan)#name LinuxVLAN
jbone-switch(config-vlan)#interface range fa1/0/1-24
jbone-switch(config-if-range)#switch
jbone-switch(config-if-range)#switchport access vla
jbone-switch(config-if-range)#switchport access vlan 2
jbone-switch(config-if-range)#interface range fa1/0/25-48
jbone-switch(config-if-range)#switchport access vlan 3
```

```BASH
jbone-switch#show vlan brief

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Gi1/0/1, Gi1/0/2, Gi1/0/3
                                                Gi1/0/4
2    WindowsVLAN                      active    Fa1/0/1, Fa1/0/2, Fa1/0/3
                                                Fa1/0/4, Fa1/0/5, Fa1/0/6
                                                Fa1/0/7, Fa1/0/8, Fa1/0/9
                                                Fa1/0/10, Fa1/0/11, Fa1/0/12
                                                Fa1/0/13, Fa1/0/14, Fa1/0/15
                                                Fa1/0/16, Fa1/0/17, Fa1/0/18
                                                Fa1/0/19, Fa1/0/20, Fa1/0/21
                                                Fa1/0/22, Fa1/0/23, Fa1/0/24
3    LinuxVLAN                        active    Fa1/0/25, Fa1/0/26, Fa1/0/27
                                                Fa1/0/28, Fa1/0/29, Fa1/0/30
                                                Fa1/0/31, Fa1/0/32, Fa1/0/33
                                                Fa1/0/34, Fa1/0/35, Fa1/0/36
                                                Fa1/0/37, Fa1/0/38, Fa1/0/39
                                                Fa1/0/40, Fa1/0/41, Fa1/0/42
                                                Fa1/0/43, Fa1/0/44, Fa1/0/45
                                                Fa1/0/46, Fa1/0/47, Fa1/0/48
1002 fddi-default                     act/unsup
1003 token-ring-default               act/unsup

VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1004 fddinet-default                  act/unsup
1005 trnet-default                    act/unsup
```

Note that these VLANs could be disabled by using the `no shutdown vlan {1-1004}` command. VLANs could either be in an 'active' or "act/lshut", when in the "act/lshut" state the switch will not forward frames to the VLAN.

### Quick aside on VTP and DTP

VLAN Trunking Protocol is used for switches to advertise their known VLANs. Most organizations seem to disabled it or put it in transparent mode. This will prevent a switch from learning any VLAN configurations and messing with the work an engineer puts in.

```BASH
jbone-switch(config)#vtp mode transparent
Setting device to VTP TRANSPARENT mode.
```

Note that my switch only allowed for the "transparent" option, I couldn't use the "off" option.

### Notes on trunking

I only have one switch, so I don't plan on doing any tunking. Still, I will use this section to talk about trunking concepts.

Trunking is used to send data from multiple VLANs across a single link. By doing this, we don't need to assign access ports per VLAN when interlinking switches. There are several caveats though. Firstly, both switches' interfaces must be configured to trunk mode, and they must have the same "active" VLANs. Without this, frames could be dropped because an access port received a frame with a VLAN tag, or they don't know a VLAN or believe it's disabled. We can use `#show vlan brief` to view the status of each VLAN.

When setting up trunk ports, one should be listening for negotiations (`(config-if)#switchport mode desirable auto`) and the other should be initiaitng the negotiation (`(config-if)#switchport mode desirable desirable`). If both are set to passively listening, then no negotiations will occur.

When entering the `#show interfaces trunk` command, if there is no output, this means that the trunk conifugarion isn't working. Also, the `#show interfaces switchport` is another useful command for troubleshooting VLAN issues.
