# Open5GS 5GC & UERANSIM UE / RAN Sample Configuration - UPG-VPP(VPP/DPDK UPF)
This describes a simple configuration for working Open5GS 5GC and UPG-VPP.
In particular, see [here](https://github.com/s5uishida/install_vpp_upf_dpdk) for UPG-VPP configuration.

---

### [Sample Configurations and Miscellaneous for Mobile Network](https://github.com/s5uishida/sample_config_misc_for_mobile_network)

---

<a id="toc"></a>

## Table of Contents

- [Overview of Open5GS 5GC Simulation Mobile Network](#overview)
- [Changes in configuration files of Open5GS 5GC, UPG-VPP and UERANSIM UE / RAN](#changes)
  - [Changes in configuration files of Open5GS 5GC C-Plane](#changes_cp)
  - [Changes in configuration files of UPG-VPP](#changes_up)
  - [Changes in configuration files of UERANSIM UE / RAN](#changes_ueransim)
    - [Changes in configuration files of RAN](#changes_ran)
    - [Changes in configuration files of UE (IMSI-001010000000000)](#changes_ue)
- [Network settings of Open5GS 5GC, UPG-VPP and UERANSIM UE / RAN](#network_settings)
  - [Network settings of UPG-VPP and Data Network Gateway](#network_settings_up)
- [Build Open5GS, UPG-VPP and UERANSIM](#build)
- [Run Open5GS 5GC, UPG-VPP and UERANSIM UE / RAN](#run)
  - [Run UPG-VPP](#run_up)
  - [Run Open5GS 5GC C-Plane](#run_cp)
  - [Run UERANSIM](#run_ueran)
    - [Start gNB](#start_gnb)
    - [Start UE](#start_ue)
- [Ping google.com](#ping)
  - [Case for going through DN 10.45.0.0/16](#ping_1)
- [Changelog (summary)](#changelog)

---

<a id="overview"></a>

## Overview of Open5GS 5GC Simulation Mobile Network

This describes a simple configuration of C-Plane, UPG-VPP and Data Network Gateway for Open5GS 5GC.
**Note that this configuration is implemented with Proxmox VE VMs.**

The following minimum configuration was set as a condition.
- One UPF and Data Network Gateway
- One UE and one DNN

The built simulation environment is as follows.

<img src="./images/network-overview.png" title="./images/network-overview.png" width=1000px></img>

The 5GC / VPP/DPDK UPF / UE / RAN used are as follows.
- 5GC - Open5GS v2.7.5 (2025.04.25) - https://github.com/open5gs/open5gs
- VPP/DPDK UPF - UPG-VPP v1.13.0 (2024.03.25) - https://github.com/travelping/upg-vpp
- UE / RAN - UERANSIM v3.2.7 (2025.04.28) - https://github.com/aligungr/UERANSIM

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Mem<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | Open5GS 5GC C-Plane | 192.168.0.111/24 | Ubuntu 24.04 | 1 | 2GB | 20GB |
| VM-UP | UPG-VPP U-Plane | 192.168.0.151/24 | Ubuntu **22.04** | 2 | 8GB | 20GB |
| VM-DN | Data Network Gateway  | 192.168.0.152/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |
| VM2 | UERANSIM RAN (gNodeB) | 192.168.0.131/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |
| VM3 | UERANSIM UE | 192.168.0.132/24 | Ubuntu 24.04 | 1 | 1GB | 10GB |

The network interfaces of each VM are as follows.
**Note. Do not enable(up) any devices that will be under the control of DPDK.
These devices will be enabled and set IP addresses in the `init.conf` file of UPG-VPP.**
| VM | Device | Model | Linux Bridge | IP address | Interface | Under DPDK |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | ens18 | VirtIO | vmbr1 | 10.0.0.111/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.111/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr4 | 192.168.14.111/24 | N4 | -- |
| VM-UP | ~~ens18~~ | ~~VirtIO~~ | ~~vmbr1~~ | ~~10.0.0.151/24~~ | ~~(NAPT NW)~~ ***down*** | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.151/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr3 | 192.168.13.151/24 | N3 | x |
| | ens21 | VirtIO | vmbr4 | 192.168.14.151/24 | N4 | x |
| | ens22 | VirtIO | vmbr6 | 192.168.16.151/24 | N6 | x |
| VM-DN | ens18 | VirtIO | vmbr1 | 10.0.0.152/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.152/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr6 | 192.168.16.152/24 | N6 | -- |
| VM2 | ens18 | VirtIO | vmbr1 | 10.0.0.131/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.131/24 | (Mgmt NW) | -- |
| | ens20 | VirtIO | vmbr3 | 192.168.13.131/24 | N3 | -- |
| VM3 | ens18 | VirtIO | vmbr1 | 10.0.0.132/24 | (NAPT NW) | -- |
| | ens19 | VirtIO | mgbr0 | 192.168.0.132/24 | (Mgmt NW) | -- |

Linux Bridges of Proxmox VE are as follows.
| Linux Bridge | Network CIDR | Interface |
| --- | --- | --- |
| vmbr1 | 10.0.0.0/24 | NAPT NW |
| mgbr0 | 192.168.0.0/24 | Mgmt NW |
| vmbr3 | 192.168.13.0/24 | N3 |
| vmbr4 | 192.168.14.0/24 | N4 |
| vmbr6 | 192.168.16.0/24 | N6 |

Set network instance to `internet`.
| Network Instance |
| --- |
| internet |

Subscriber Information (other information is the same) is as follows.  
**Note. Please select OP or OPc according to the setting of UERANSIM UE configuration file.**
| UE | IMSI | DNN | OP/OPc |
| --- | --- | --- | --- |
| UE | 001010000000000 | internet | OPc |

I registered these information with the Open5GS WebUI.
In addition, [3GPP TS 35.208](https://www.3gpp.org/DynaReport/35208.htm) "4.3 Test Sets" is published by 3GPP as test data for the 3GPP authentication and key generation functions (MILENAGE).

The DN is as follows.
| DN | DNN | TUNnel interface of UE |
| --- | --- | --- |
| 10.45.0.0/16 | internet | uesimtun0 |

<a id="changes"></a>

## Changes in configuration files of Open5GS 5GC, UPG-VPP and UERANSIM UE / RAN

Please refer to the following for building Open5GS, UPG-VPP and UERANSIM respectively.
- Open5GS v2.7.5 (2025.04.25) - https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/
- UPG-VPP v1.13.0 (2024.03.25) - https://github.com/s5uishida/install_vpp_upf_dpdk
- UERANSIM v3.2.7 (2025.04.28) - https://github.com/aligungr/UERANSIM/wiki/Installation

<a id="changes_cp"></a>

### Changes in configuration files of Open5GS 5GC C-Plane

The following parameters can be used in the logic that selects UPF as the connection destination by PFCP.

- DNN
- TAC (Tracking Area Code)
- nr_CellID

For the sake of simplicity, I used only DNN this time.

- `open5gs/install/etc/open5gs/amf.yaml`
```diff
--- amf.yaml.orig       2025-04-27 11:38:05.000000000 +0900
+++ amf.yaml    2025-05-04 08:12:34.196824268 +0900
@@ -20,27 +20,27 @@
         - uri: http://127.0.0.200:7777
   ngap:
     server:
-      - address: 127.0.0.5
+      - address: 192.168.0.111
   metrics:
     server:
       - address: 127.0.0.5
         port: 9090
   guami:
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       amf_id:
         region: 2
         set: 1
   tai:
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       tac: 1
   plmn_support:
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
       s_nssai:
         - sst: 1
   security:
```
- `open5gs/install/etc/open5gs/nrf.yaml`
```diff
--- nrf.yaml.orig       2025-04-27 11:38:05.000000000 +0900
+++ nrf.yaml    2025-05-04 08:13:05.973154453 +0900
@@ -11,8 +11,8 @@
 nrf:
   serving:  # 5G roaming requires PLMN in NRF
     - plmn_id:
-        mcc: 999
-        mnc: 70
+        mcc: 001
+        mnc: 01
   sbi:
     server:
       - address: 127.0.0.10
```
- `open5gs/install/etc/open5gs/smf.yaml`
```diff
--- smf.yaml.orig       2025-04-27 11:38:05.000000000 +0900
+++ smf.yaml    2025-05-04 09:51:25.314794940 +0900
@@ -7,6 +7,8 @@
   max:
     ue: 1024  # The number of UE can be increased depending on memory size.
 #    peer: 64
+  parameter:
+    use_upg_vpp: true
 
 smf:
   sbi:
@@ -20,16 +22,14 @@
         - uri: http://127.0.0.200:7777
   pfcp:
     server:
-      - address: 127.0.0.4
+      - address: 192.168.14.111
     client:
       upf:
-        - address: 127.0.0.7
-  gtpc:
-    server:
-      - address: 127.0.0.4
+        - address: 192.168.14.151
+          dnn: internet
   gtpu:
     server:
-      - address: 127.0.0.4
+      - address: 192.168.14.111
   metrics:
     server:
       - address: 127.0.0.4
@@ -37,20 +37,17 @@
   session:
     - subnet: 10.45.0.0/16
       gateway: 10.45.0.1
-    - subnet: 2001:db8:cafe::/48
-      gateway: 2001:db8:cafe::1
+      dnn: internet
   dns:
     - 8.8.8.8
     - 8.8.4.4
-    - 2001:4860:4860::8888
-    - 2001:4860:4860::8844
   mtu: 1400
 #  p-cscf:
 #    - 127.0.0.1
 #    - ::1
 #  ctf:
 #    enabled: auto   # auto(default)|yes|no
-  freeDiameter: /root/open5gs/install/etc/freeDiameter/smf.conf
+#  freeDiameter: /root/open5gs/install/etc/freeDiameter/smf.conf
 
 ################################################################################
 # SMF Info
```

<a id="changes_up"></a>

### Changes in configuration files of UPG-VPP

See [here](https://github.com/s5uishida/install_vpp_upf_dpdk#conf) for the original files.

- `upg-vpp/startup.conf`  
There is no change.

- `upg-vpp/init.conf`  
There is no change.

<a id="changes_ueransim"></a>

### Changes in configuration files of UERANSIM UE / RAN

<a id="changes_ran"></a>

#### Changes in configuration files of RAN

- `UERANSIM/config/open5gs-gnb.yaml`
```diff
--- open5gs-gnb.yaml.orig       2023-12-02 06:14:20.000000000 +0900
+++ open5gs-gnb.yaml    2025-05-04 08:59:07.242339870 +0900
@@ -1,17 +1,17 @@
-mcc: '999'          # Mobile Country Code value
-mnc: '70'           # Mobile Network Code value (2 or 3 digits)
+mcc: '001'          # Mobile Country Code value
+mnc: '01'           # Mobile Network Code value (2 or 3 digits)
 
 nci: '0x000000010'  # NR Cell Identity (36-bit)
 idLength: 32        # NR gNB ID length in bits [22...32]
 tac: 1              # Tracking Area Code
 
-linkIp: 127.0.0.1   # gNB's local IP address for Radio Link Simulation (Usually same with local IP)
-ngapIp: 127.0.0.1   # gNB's local IP address for N2 Interface (Usually same with local IP)
-gtpIp: 127.0.0.1    # gNB's local IP address for N3 Interface (Usually same with local IP)
+linkIp: 192.168.0.131   # gNB's local IP address for Radio Link Simulation (Usually same with local IP)
+ngapIp: 192.168.0.131   # gNB's local IP address for N2 Interface (Usually same with local IP)
+gtpIp: 192.168.13.131    # gNB's local IP address for N3 Interface (Usually same with local IP)
 
 # List of AMF address information
 amfConfigs:
-  - address: 127.0.0.5
+  - address: 192.168.0.111
     port: 38412
 
 # List of supported S-NSSAIs by this gNB
```

<a id="changes_ue"></a>

#### Changes in configuration files of UE (IMSI-001010000000000)

- `UERANSIM/config/open5gs-ue.yaml`
```diff
--- open5gs-ue.yaml.orig        2025-03-16 15:49:12.000000000 +0900
+++ open5gs-ue.yaml     2025-05-04 09:24:31.352554298 +0900
@@ -1,9 +1,9 @@
 # IMSI number of the UE. IMSI = [MCC|MNC|MSISDN] (In total 15 digits)
-supi: 'imsi-999700000000001'
+supi: 'imsi-001010000000000'
 # Mobile Country Code value of HPLMN
-mcc: '999'
+mcc: '001'
 # Mobile Network Code value of HPLMN (2 or 3 digits)
-mnc: '70'
+mnc: '01'
 # SUCI Protection Scheme : 0 for Null-scheme, 1 for Profile A and 2 for Profile B
 protectionScheme: 0
 # Home Network Public Key for protecting with SUCI Profile A
@@ -31,7 +31,7 @@
 
 # List of gNB IP addresses for Radio Link Simulation
 gnbSearchList:
-  - 127.0.0.1
+  - 192.168.0.131
 
 # UAC Access Identities Configuration
 uacAic:
```

<a id="network_settings"></a>

## Network settings of Open5GS 5GC, UPG-VPP and UERANSIM UE / RAN

<a id="network_settings_up"></a>

### Network settings of UPG-VPP and Data Network Gateway

See [this1](https://github.com/s5uishida/install_vpp_upf_dpdk#setup_up) and [this2](https://github.com/s5uishida/install_vpp_upf_dpdk#setup_dn).

<a id="build"></a>

## Build Open5GS, UPG-VPP and UERANSIM

Please refer to the following for building Open5GS, UPG-VPP and UERANSIM respectively.
- Open5GS v2.7.5 (2025.04.25) - https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/
- UPG-VPP v1.13.0 (2024.03.25) - https://github.com/s5uishida/install_vpp_upf_dpdk
- UERANSIM v3.2.7 (2025.04.28) - https://github.com/aligungr/UERANSIM/wiki/Installation

Install MongoDB on Open5GS 5GC C-Plane machine.
[MongoDB Compass](https://www.mongodb.com/products/compass) is a convenient tool to look at the MongoDB database.

<a id="run"></a>

## Run Open5GS 5GC, UPG-VPP and UERANSIM UE / RAN

First run UPG-VPP, then the 5GC and UERANSIM (UE & RAN implementation).

<a id="run_up"></a>

### Run UPG-VPP

See [this](https://github.com/s5uishida/install_vpp_upf_dpdk#run).

<a id="run_cp"></a>

### Run Open5GS 5GC C-Plane

```
./install/bin/open5gs-nrfd &
sleep 2
./install/bin/open5gs-scpd &
sleep 2
./install/bin/open5gs-amfd &
sleep 2
./install/bin/open5gs-smfd &
./install/bin/open5gs-ausfd &
./install/bin/open5gs-udmd &
./install/bin/open5gs-udrd &
./install/bin/open5gs-pcfd &
./install/bin/open5gs-nssfd &
./install/bin/open5gs-bsfd &
```

The status of PFCP association between UPG-VPP and Open5GS SMF is as follows.
```
vpp# show upf association 
Node: 192.168.14.111
  Recovery Time Stamp: 2025/05/04 11:18:55:000
  Sessions: 0
vpp#
```

<a id="run_ueran"></a>

### Run UERANSIM

Here, the case of UE (IMSI-001010000000000) & RAN is described.
First, do an NG Setup between gNodeB and 5GC, then register the UE with 5GC and establish a PDU session.

Please refer to the following for usage of UERANSIM.

https://github.com/aligungr/UERANSIM/wiki/Usage

<a id="start_gnb"></a>

#### Start gNB

Start gNB as follows.
```
# ./nr-gnb -c ../config/open5gs-gnb.yaml
UERANSIM v3.2.7
[2025-05-04 11:19:59.312] [sctp] [info] Trying to establish SCTP connection... (192.168.0.111:38412)
[2025-05-04 11:19:59.339] [sctp] [info] SCTP connection established (192.168.0.111:38412)
[2025-05-04 11:19:59.339] [sctp] [debug] SCTP association setup ascId[3]
[2025-05-04 11:19:59.339] [ngap] [debug] Sending NG Setup Request
[2025-05-04 11:19:59.346] [ngap] [debug] NG Setup Response received
[2025-05-04 11:19:59.346] [ngap] [info] NG Setup procedure is successful
```
The Open5GS C-Plane log when executed is as follows.
```
05/04 11:19:59.364: [amf] INFO: gNB-N2 accepted[192.168.0.131]:56660 in ng-path module (../src/amf/ngap-sctp.c:113)
05/04 11:19:59.364: [amf] INFO: gNB-N2 accepted[192.168.0.131] in master_sm module (../src/amf/amf-sm.c:813)
05/04 11:19:59.370: [amf] INFO: [Added] Number of gNBs is now 1 (../src/amf/context.c:1277)
05/04 11:19:59.370: [amf] INFO: gNB-N2[192.168.0.131] max_num_of_ostreams : 10 (../src/amf/amf-sm.c:860)
```

<a id="start_ue"></a>

#### Start UE

Start UE as follows. This will register the UE with 5GC and establish a PDU session.
```
# ./nr-ue -c ../config/open5gs-ue.yaml
UERANSIM v3.2.7
[2025-05-04 11:20:54.997] [nas] [info] UE switches to state [MM-DEREGISTERED/PLMN-SEARCH]
[2025-05-04 11:20:54.998] [rrc] [debug] New signal detected for cell[1], total [1] cells in coverage
[2025-05-04 11:20:54.998] [nas] [info] Selected plmn[001/01]
[2025-05-04 11:20:54.998] [rrc] [info] Selected cell plmn[001/01] tac[1] category[SUITABLE]
[2025-05-04 11:20:54.998] [nas] [info] UE switches to state [MM-DEREGISTERED/PS]
[2025-05-04 11:20:54.998] [nas] [info] UE switches to state [MM-DEREGISTERED/NORMAL-SERVICE]
[2025-05-04 11:20:54.998] [nas] [debug] Initial registration required due to [MM-DEREG-NORMAL-SERVICE]
[2025-05-04 11:20:54.999] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2025-05-04 11:20:54.999] [nas] [debug] Sending Initial Registration
[2025-05-04 11:20:55.000] [rrc] [debug] Sending RRC Setup Request
[2025-05-04 11:20:55.000] [nas] [info] UE switches to state [MM-REGISTER-INITIATED]
[2025-05-04 11:20:55.000] [rrc] [info] RRC connection established
[2025-05-04 11:20:55.000] [rrc] [info] UE switches to state [RRC-CONNECTED]
[2025-05-04 11:20:55.000] [nas] [info] UE switches to state [CM-CONNECTED]
[2025-05-04 11:20:55.007] [nas] [debug] Authentication Request received
[2025-05-04 11:20:55.007] [nas] [debug] Received SQN [000000000100]
[2025-05-04 11:20:55.007] [nas] [debug] SQN-MS [000000000000]
[2025-05-04 11:20:55.011] [nas] [debug] Security Mode Command received
[2025-05-04 11:20:55.011] [nas] [debug] Selected integrity[2] ciphering[0]
[2025-05-04 11:20:55.023] [nas] [debug] Registration accept received
[2025-05-04 11:20:55.023] [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
[2025-05-04 11:20:55.023] [nas] [debug] Sending Registration Complete
[2025-05-04 11:20:55.024] [nas] [info] Initial Registration is successful
[2025-05-04 11:20:55.024] [nas] [debug] Sending PDU Session Establishment Request
[2025-05-04 11:20:55.024] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2025-05-04 11:20:55.230] [nas] [debug] Configuration Update Command received
[2025-05-04 11:20:55.244] [nas] [debug] PDU Session Establishment Accept received
[2025-05-04 11:20:55.246] [nas] [info] PDU Session establishment is successful PSI[1]
[2025-05-04 11:20:55.261] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.45.0.2] is up.
```
The Open5GS C-Plane log when executed is as follows.
```
05/04 11:20:55.022: [amf] INFO: InitialUEMessage (../src/amf/ngap-handler.c:437)
05/04 11:20:55.022: [amf] INFO: [Added] Number of gNB-UEs is now 1 (../src/amf/context.c:2789)
05/04 11:20:55.022: [amf] INFO:     RAN_UE_NGAP_ID[1] AMF_UE_NGAP_ID[1] TAC[1] CellID[0x10] (../src/amf/ngap-handler.c:598)
05/04 11:20:55.022: [amf] INFO: [suci-0-001-01-0000-0-0-0000000000] Unknown UE by SUCI (../src/amf/context.c:1906)
05/04 11:20:55.022: [amf] INFO: [Added] Number of AMF-UEs is now 1 (../src/amf/context.c:1682)
05/04 11:20:55.022: [gmm] INFO: Registration request (../src/amf/gmm-sm.c:1323)
05/04 11:20:55.022: [gmm] INFO: [suci-0-001-01-0000-0-0-0000000000]    SUCI (../src/amf/gmm-handler.c:174)
05/04 11:20:55.022: [sbi] INFO: [215691d0-288e-41f0-a15d-9ba19c82b07f] Setup NF Instance [type:AUSF] (../lib/sbi/path.c:307)
05/04 11:20:55.023: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.11:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.023: [sbi] INFO: [2156f076-288e-41f0-9e06-0562d8e22070] Setup NF Instance [type:UDM] (../lib/sbi/path.c:307)
05/04 11:20:55.023: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.024: [sbi] INFO: [21581028-288e-41f0-be7f-2dede91cecc9] Setup NF Instance [type:UDR] (../lib/sbi/path.c:307)
05/04 11:20:55.024: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.027: [amf] INFO: Setup NF EndPoint(addr) [127.0.0.11:7777] (../src/amf/nausf-handler.c:130)
05/04 11:20:55.029: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.11:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.029: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.029: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.031: [ausf] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/ausf/nudm-handler.c:337)
05/04 11:20:55.033: [sbi] INFO: [2156f076-288e-41f0-9e06-0562d8e22070] Setup NF Instance [type:UDM] (../lib/sbi/path.c:307)
05/04 11:20:55.033: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.034: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.035: [sbi] INFO: [2156f076-288e-41f0-9e06-0562d8e22070] Setup NF Instance [type:UDM] (../lib/sbi/path.c:307)
05/04 11:20:55.035: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.036: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.037: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.037: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.039: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.039: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.040: [amf] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/amf/nudm-handler.c:355)
05/04 11:20:55.040: [sbi] INFO: [2158610e-288e-41f0-aa63-5d9bf2c8951b] Setup NF Instance [type:PCF] (../lib/sbi/path.c:307)
05/04 11:20:55.040: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.13:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.041: [pcf] INFO: Setup NF EndPoint(addr) [127.0.0.5:7777] (../src/pcf/npcf-handler.c:114)
05/04 11:20:55.041: [sbi] INFO: [21581028-288e-41f0-be7f-2dede91cecc9] Setup NF Instance [type:UDR] (../lib/sbi/path.c:307)
05/04 11:20:55.041: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.043: [amf] INFO: Setup NF EndPoint(addr) [127.0.0.13:7777] (../src/amf/npcf-handler.c:143)
05/04 11:20:55.250: [gmm] INFO: [imsi-001010000000000] Registration complete (../src/amf/gmm-sm.c:2658)
05/04 11:20:55.250: [amf] INFO: [imsi-001010000000000] Configuration update command (../src/amf/nas-path.c:607)
05/04 11:20:55.250: [gmm] INFO:     UTC [2025-05-04T02:20:55] Timezone[0]/DST[0] (../src/amf/gmm-build.c:551)
05/04 11:20:55.250: [gmm] INFO:     LOCAL [2025-05-04T11:20:55] Timezone[32400]/DST[0] (../src/amf/gmm-build.c:556)
05/04 11:20:55.250: [amf] INFO: [Added] Number of AMF-Sessions is now 1 (../src/amf/context.c:2810)
05/04 11:20:55.250: [gmm] INFO: UE SUPI[imsi-001010000000000] DNN[internet] S_NSSAI[SST:1 SD:0xffffff] smContextRef[NULL] smContextResourceURI[NULL] (../src/amf/gmm-handler.c:1374)
05/04 11:20:55.251: [amf] INFO: [21690770-288e-41f0-9420-036109b58534] Setup NF Instance [type:SMF] (../src/amf/context.c:2433)
05/04 11:20:55.251: [gmm] INFO: SMF Instance [21690770-288e-41f0-9420-036109b58534] (../src/amf/gmm-handler.c:1415)
05/04 11:20:55.251: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.4:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.251: [smf] INFO: [Added] Number of SMF-UEs is now 1 (../src/smf/context.c:1033)
05/04 11:20:55.252: [smf] INFO: [Added] Number of SMF-Sessions is now 1 (../src/smf/context.c:3190)
05/04 11:20:55.252: [smf] INFO: Setup NF EndPoint(addr) [127.0.0.5:7777] (../src/smf/nsmf-handler.c:270)
05/04 11:20:55.252: [sbi] INFO: [2156f076-288e-41f0-9e06-0562d8e22070] Setup NF Instance [type:UDM] (../lib/sbi/path.c:307)
05/04 11:20:55.252: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.253: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.255: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.255: [smf] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/smf/nudm-handler.c:461)
05/04 11:20:55.256: [sbi] INFO: [2158610e-288e-41f0-aa63-5d9bf2c8951b] Setup NF Instance [type:PCF] (../lib/sbi/path.c:307)
05/04 11:20:55.256: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.13:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.256: [amf] INFO: Setup NF EndPoint(addr) [127.0.0.4:7777] (../src/amf/nsmf-handler.c:144)
05/04 11:20:55.257: [pcf] INFO: Setup NF EndPoint(addr) [127.0.0.4:7777] (../src/pcf/npcf-handler.c:442)
05/04 11:20:55.257: [sbi] INFO: [21581028-288e-41f0-be7f-2dede91cecc9] Setup NF Instance [type:UDR] (../lib/sbi/path.c:307)
05/04 11:20:55.257: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.258: [sbi] INFO: [21559942-288e-41f0-9a97-2596579b443d] Setup NF Instance [type:BSF] (../lib/sbi/path.c:307)
05/04 11:20:55.259: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.15:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.259: [pcf] INFO: Setup NF EndPoint(addr) [127.0.0.15:7777] (../src/pcf/nbsf-handler.c:121)
05/04 11:20:55.260: [smf] INFO: Setup NF EndPoint(addr) [127.0.0.13:7777] (../src/smf/npcf-handler.c:367)
05/04 11:20:55.260: [smf] INFO: UE SUPI[imsi-001010000000000] DNN[internet] IPv4[10.45.0.2] IPv6[] (../src/smf/npcf-handler.c:580)
05/04 11:20:55.262: [gtp] INFO: gtp_connect() [192.168.13.151]:2152 (../lib/gtp/path.c:60)
05/04 11:20:55.262: [sbi] INFO: [201eb41e-288e-41f0-aa4e-e360bb66c564] Setup NF Instance [type:AMF] (../lib/sbi/path.c:307)
05/04 11:20:55.262: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.5:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.265: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.4:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.269: [sbi] INFO: [2156f076-288e-41f0-9e06-0562d8e22070] Setup NF Instance [type:UDM] (../lib/sbi/path.c:307)
05/04 11:20:55.269: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.12:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.270: [sbi] INFO: [21581028-288e-41f0-be7f-2dede91cecc9] Setup NF Instance [type:UDR] (../lib/sbi/path.c:307)
05/04 11:20:55.270: [scp] INFO: Setup NF EndPoint(addr) [127.0.0.20:7777] (../src/scp/sbi-path.c:461)
05/04 11:20:55.271: [amf] INFO: [imsi-001010000000000:1:11][0:0:NULL] /nsmf-pdusession/v1/sm-contexts/{smContextRef}/modify (../src/amf/nsmf-handler.c:942)
```
The PDU session establishment status of UPG-VPP is as follows.
```
vpp# show upf session 
CP F-SEID: 0x0000000000000f75 (3957) @ 192.168.14.111
UP F-SEID: 0x0000000000000f75 (3957) @ 192.168.14.151 (192.168.14.111 ::)
User ID: IMEI:4370816125816151
  PFCP Association: 0
  TEID assignment per choose ID
PDR: 1 @ 0x7f5ee899ab78
  Precedence: 65535
  PDI:
    Fields: 0000000c
    Source Interface: Core
    Network Instance: internet
    UE IP address (destination):
      IPv4 address: 10.45.0.2
    SDF Filter [1]:
      permit out ip from any to assigned 
  Outer Header Removal: no
  FAR Id: 1
  URR Ids: [1] @ 0x7f5ee7ae7ee8
  QER Ids: [1] @ 0x7f5e92963928
PDR: 2 @ 0x7f5ee899abf8
  Precedence: 65535
  PDI:
    Fields: 0000000d
    Source Interface: Access
    Network Instance: internet
    Local F-TEID: 514300325 (0x1ea799a5)
            IPv4: 192.168.13.151
    UE IP address (source):
      IPv4 address: 10.45.0.2
    SDF Filter [1]:
      permit out ip from any to assigned 
  Outer Header Removal: GTP-U/UDP/IPv4
  FAR Id: 2
  URR Ids: [] @ 0x0
  QER Ids: [1] @ 0x7f5ee89a97c8
PDR: 3 @ 0x7f5ee899ac78
  Precedence: 255
  PDI:
    Fields: 00000001
    Source Interface: CP-function
    Network Instance: internet
    Local F-TEID: 323387072 (0x13467ec0)
            IPv4: 192.168.13.151
  Outer Header Removal: GTP-U/UDP/IPv4
  FAR Id: 1
  URR Ids: [] @ 0x0
  QER Ids: [] @ 0x0
PDR: 4 @ 0x7f5ee899acf8
  Precedence: 255
  PDI:
    Fields: 00000009
    Source Interface: Access
    Network Instance: internet
    Local F-TEID: 514300325 (0x1ea799a5)
            IPv4: 192.168.13.151
    SDF Filter [1]:
      permit out 58 from ff02::2 to assigned 
  Outer Header Removal: GTP-U/UDP/IPv4
  FAR Id: 3
  URR Ids: [] @ 0x0
  QER Ids: [] @ 0x0
FAR: 1
  Apply Action: 00000002 == [FORWARD]
  Forward:
    Network Instance: internet
    Destination Interface: 0
    Outer Header Creation: [GTP-U/UDP/IPv4],TEID:00000001,IP:192.168.13.131
FAR: 2
  Apply Action: 00000002 == [FORWARD]
  Forward:
    Network Instance: internet
    Destination Interface: 1
FAR: 3
  Apply Action: 00000002 == [FORWARD]
  Forward:
    Network Instance: internet
    Destination Interface: 3
    Outer Header Creation: [GTP-U/UDP/IPv4],TEID:00000001,IP:192.168.14.111
URR: 1
  Measurement Method: 0002 == [VOLUME]
  Reporting Triggers: 0002 == [VOLUME THRESHOLD]
  Status: 0 == []
  Start Time: 2025/05/04 11:20:55:238
  vTime of First Usage:       0.0000 
  vTime of Last Usage:        0.0000 
  Volume
    Up:    Measured:                    0, Theshold:                    0, Pkts:          0
           Consumed:                    0, Quota:                       0
    Down:  Measured:                    0, Theshold:                    0, Pkts:          0
           Consumed:                    0, Quota:                       0
    Total: Measured:                    0, Theshold:            104857600, Pkts:          0
           Consumed:                    0, Quota:                       0
vpp#
```
Looking at the console log of the `nr-ue` command, UE has been assigned the IP address `10.45.0.2` from Open5GS 5GC.
```
[2025-05-04 11:20:55.261] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.45.0.2] is up.
```
Just in case, make sure it matches the IP address of the UE's TUNnel interface.
```
# ip addr show
...
4: uesimtun0: <POINTOPOINT,PROMISC,NOTRAILERS,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.45.0.2/24 scope global uesimtun0
       valid_lft forever preferred_lft forever
    inet6 fe80::37d1:46c8:6afa:2df2/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
...
```

<a id="ping"></a>

## Ping google.com

Specify the UE's TUNnel interface and try ping.

Please refer to the following for usage of TUNnel interface.

https://github.com/aligungr/UERANSIM/wiki/Usage

<a id="ping_1"></a>

### Case for going through DN 10.45.0.0/16

Run `tcpdump` on VM-DN and check that the packet goes through N6 (ens20).
- `ping google.com` on VM3 (UE)
```
# ping google.com -I uesimtun0 -n
PING google.com (172.217.175.78) from 10.45.0.2 uesimtun0: 56(84) bytes of data.
64 bytes from 172.217.175.78: icmp_seq=1 ttl=51 time=17.5 ms
64 bytes from 172.217.175.78: icmp_seq=2 ttl=51 time=17.9 ms
64 bytes from 172.217.175.78: icmp_seq=3 ttl=51 time=17.5 ms
```
- Run `tcpdump` on VM-DN
```
# tcpdump -i ens20 -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on ens20, link-type EN10MB (Ethernet), snapshot length 262144 bytes
11:25:06.699034 IP 10.45.0.2 > 172.217.175.78: ICMP echo request, id 1137, seq 1, length 64
11:25:06.715577 IP 172.217.175.78 > 10.45.0.2: ICMP echo reply, id 1137, seq 1, length 64
11:25:07.700670 IP 10.45.0.2 > 172.217.175.78: ICMP echo request, id 1137, seq 2, length 64
11:25:07.717652 IP 172.217.175.78 > 10.45.0.2: ICMP echo reply, id 1137, seq 2, length 64
11:25:08.702595 IP 10.45.0.2 > 172.217.175.78: ICMP echo request, id 1137, seq 3, length 64
11:25:08.719268 IP 172.217.175.78 > 10.45.0.2: ICMP echo reply, id 1137, seq 3, length 64
```

You could specify the IP address assigned to the TUNnel interface to run almost any applications (iperf3 etc.) as in the following example using `nr-binder` tool.

- `curl google.com` on VM3 (UE)
```
# sh nr-binder 10.45.0.2 curl google.com
<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```
- Run `tcpdump` on VM-DN
```
11:26:04.693062 IP 10.45.0.2.60561 > 142.250.196.110.80: Flags [S], seq 1752236202, win 65280, options [mss 1360,sackOK,TS val 2385808587 ecr 0,nop,wscale 7], length 0
11:26:04.709722 IP 142.250.196.110.80 > 10.45.0.2.60561: Flags [S.], seq 838692509, ack 1752236203, win 65535, options [mss 1412,sackOK,TS val 3356607603 ecr 2385808587,nop,wscale 8], length 0
11:26:04.710622 IP 10.45.0.2.60561 > 142.250.196.110.80: Flags [.], ack 1, win 510, options [nop,nop,TS val 2385808604 ecr 3356607603], length 0
11:26:04.710622 IP 10.45.0.2.60561 > 142.250.196.110.80: Flags [P.], seq 1:74, ack 1, win 510, options [nop,nop,TS val 2385808604 ecr 3356607603], length 73: HTTP: GET / HTTP/1.1
11:26:04.728380 IP 142.250.196.110.80 > 10.45.0.2.60561: Flags [.], ack 74, win 1050, options [nop,nop,TS val 3356607622 ecr 2385808604], length 0
11:26:04.769936 IP 142.250.196.110.80 > 10.45.0.2.60561: Flags [P.], seq 1:774, ack 74, win 1050, options [nop,nop,TS val 3356607663 ecr 2385808604], length 773: HTTP: HTTP/1.1 301 Moved Permanently
11:26:04.770833 IP 10.45.0.2.60561 > 142.250.196.110.80: Flags [.], ack 774, win 504, options [nop,nop,TS val 2385808664 ecr 3356607663], length 0
11:26:04.770833 IP 10.45.0.2.60561 > 142.250.196.110.80: Flags [F.], seq 74, ack 774, win 504, options [nop,nop,TS val 2385808665 ecr 3356607663], length 0
11:26:04.787114 IP 142.250.196.110.80 > 10.45.0.2.60561: Flags [F.], seq 774, ack 75, win 1050, options [nop,nop,TS val 3356607680 ecr 2385808665], length 0
11:26:04.787921 IP 10.45.0.2.60561 > 142.250.196.110.80: Flags [.], ack 775, win 504, options [nop,nop,TS val 2385808682 ecr 3356607680], length 0
```
Please note that the `ping` tool does not work with `nr-binder`. Please refer to [here](https://github.com/aligungr/UERANSIM/issues/186#issuecomment-729534464) for the reason.
You could now connect to the DN and send any packets on the network using UPG-VPP.

---

Now you could work Open5GS 5GC with UPG-VPP.
I would like to thank the excellent developers and all the contributors of Open5GS, UPG-VPP, VPP, DPDK and UERANSIM.

<a id="changelog"></a>

## Changelog (summary)

- [2025.05.04] Updated to UPG-VPP `v1.13.0`, Open5GS `v2.7.5 (2025.04.25)` and UERANSIM `v3.2.7 (2025.04.28)`. Changed the VM environment from Virtualbox to Proxmox VE.
- [2024.03.31] [This commit](https://github.com/open5gs/open5gs/commit/e8a3b76af395a9986234b7d339a7a96dc5bb537f) fixed the issue where SMF crashes without `gtpc` section in `smf.yaml`. So deleted the `gtpc` section in `smf.yaml` for 5G use.
- [2024.03.24] Updated to UPG-VPP `v1.12.0`.
- [2023.11.10] Changed from [oai-cn5g-upf-vpp](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf-vpp) to its base [travelping/upg-vpp](https://github.com/travelping/upg-vpp).
- [2023.06.18] Used the original without changing `init.conf`. This made the N3 and N4 networks separate like the original.
- [2023.06.15] Initial release.
