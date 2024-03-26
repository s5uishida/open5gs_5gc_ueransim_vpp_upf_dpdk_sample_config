# Open5GS 5GC & UERANSIM UE / RAN Sample Configuration - VPP-UPF with DPDK
This describes a simple configuration for working Open5GS 5GC and VPP-UPF with DPDK.
In particular, see [here](https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1) for VPP-UPF with DPDK configuration.

**If UPG-VPP built with [this instruction](https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1) does not work well, please try OAI-CN5G-UPF-VPP built with [this instruction](https://github.com/s5uishida/install_vpp_upf_dpdk#build).**

---

### [Sample Configurations and Miscellaneous for Mobile Network](https://github.com/s5uishida/sample_config_misc_for_mobile_network)

---

<a id="toc"></a>

## Table of Contents

- [Overview of Open5GS 5GC Simulation Mobile Network](#overview)
- [Changes in configuration files of Open5GS 5GC, VPP-UPF and UERANSIM UE / RAN](#changes)
  - [Changes in configuration files of Open5GS 5GC C-Plane](#changes_cp)
  - [Changes in configuration files of VPP-UPF](#changes_up)
  - [Changes in configuration files of UERANSIM UE / RAN](#changes_ueransim)
    - [Changes in configuration files of RAN](#changes_ran)
    - [Changes in configuration files of UE (IMSI-001010000000000)](#changes_ue)
- [Network settings of Open5GS 5GC, VPP-UPF and UERANSIM UE / RAN](#network_settings)
  - [Network settings of VPP-UPF and Data Network Gateway](#network_settings_up)
- [Build Open5GS, VPP-UPF and UERANSIM](#build)
- [Run Open5GS 5GC, VPP-UPF and UERANSIM UE / RAN](#run)
  - [Run VPP-UPF](#run_up)
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

This describes a simple configuration of C-Plane, VPP-UPF and Data Network Gateway for Open5GS 5GC.
**Note that this configuration is implemented with Virtualbox VMs.**

The following minimum configuration was set as a condition.
- One UPF and Data Network Gateway
- One UE and one DNN

The built simulation environment is as follows.

<img src="./images/network-overview.png" title="./images/network-overview.png" width=1000px></img>

The 5GC / VPP-UPF / UE / RAN used are as follows.
- 5GC - Open5GS v2.7.0 (2024.03.24) - https://github.com/open5gs/open5gs
- VPP-UPF - UPG-VPP v1.12.0 (2024.01.25) - https://github.com/travelping/upg-vpp
- UE / RAN - UERANSIM v3.2.6 (2024.03.08) - https://github.com/aligungr/UERANSIM

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Memory<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | Open5GS 5GC C-Plane | 192.168.0.111/24 | Ubuntu 22.04 | 1 | 2GB | 20GB |
| VM-UP | UPG-VPP U-Plane | 192.168.0.151/24 | Ubuntu **20.04** | 2 | 8GB | 20GB |
| VM-DN | Data Network Gateway  | 192.168.0.152/24 | Ubuntu 22.04 | 1 | 1GB | 10GB |
| VM2 | UERANSIM RAN (gNodeB) | 192.168.0.131/24 | Ubuntu 22.04 | 1 | 1GB | 10GB |
| VM3 | UERANSIM UE | 192.168.0.132/24 | Ubuntu 22.04 | 1 | 1GB | 10GB |

The network interfaces of each VM are as follows.
**Note. Do not enable(up) any devices that will be under the control of DPDK.
These devices will be enabled and set IP addresses in the `init.conf` file of VPP-UPF.**
| VM | Device | Network Adapter | IP address | Interface | Under DPDK |
| --- | --- | --- | --- | --- | --- |
| VM1 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.111/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.14.111/24 | N4 | -- |
| VM-UP | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.151/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.13.151/24 | N3 | x |
| | enp0s10 | NAT Network | 192.168.14.151/24 | N4 | x |
| | enp0s16 | NAT Network | 192.168.16.151/24 | N6 | x |
| VM-DN | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.152/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.16.152/24 | N6 | -- |
| VM2 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.131/24 | (Mgmt NW) | -- |
| | enp0s9 | NAT Network | 192.168.13.131/24 | N3 | -- |
| VM3 | enp0s3 | NAT(default) | 10.0.2.15/24 | (VM default NW) | -- |
| | enp0s8 | Bridged Adapter | 192.168.0.132/24 | (Mgmt NW) | -- |

NAT networks of Virtualbox  are as follows.
| Network Name | Network CIDR |
| --- | --- |
| N3 | 192.168.13.0/24 |
| N4 | 192.168.14.0/24 |
| N6 | 192.168.16.0/24 |

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

## Changes in configuration files of Open5GS 5GC, VPP-UPF and UERANSIM UE / RAN

Please refer to the following for building Open5GS, VPP-UPF and UERANSIM respectively.
- Open5GS v2.7.0 (2024.03.24) - https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/
- UPG-VPP v1.12.0 (2024.01.25) - https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1
- UERANSIM v3.2.6 (2024.03.08) - https://github.com/aligungr/UERANSIM/wiki/Installation

<a id="changes_cp"></a>

### Changes in configuration files of Open5GS 5GC C-Plane

The following parameters including DNN can be used in the logic that selects UPF as the connection destination by PFCP.

- DNN
- TAC (Tracking Area Code)
- nr_CellID

For the sake of simplicity, I used only DNN this time. Please refer to [here](https://github.com/open5gs/open5gs/pull/560#issue-483001043) for the logic to select UPF.

- `open5gs/install/etc/open5gs/amf.yaml`
```diff
--- amf.yaml.orig       2024-03-24 15:36:48.000000000 +0900
+++ amf.yaml    2024-03-24 20:19:57.528590659 +0900
@@ -19,27 +19,27 @@
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
```
--- nrf.yaml.orig       2024-03-24 15:36:48.000000000 +0900
+++ nrf.yaml    2024-03-24 20:58:02.421044544 +0900
@@ -10,8 +10,8 @@
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
--- smf.yaml.orig       2024-03-24 15:36:48.000000000 +0900
+++ smf.yaml    2024-03-24 20:30:06.635351026 +0900
@@ -19,35 +19,34 @@
         - uri: http://127.0.0.200:7777
   pfcp:
     server:
-      - address: 127.0.0.4
+      - address: 192.168.14.111
     client:
       upf:
-        - address: 127.0.0.7
+        - address: 192.168.14.151
+          dnn: internet
   gtpc:
     server:
       - address: 127.0.0.4
   gtpu:
     server:
-      - address: 127.0.0.4
+      - address: 192.168.14.111
   metrics:
     server:
       - address: 127.0.0.4
         port: 9090
   session:
     - subnet: 10.45.0.1/16
-    - subnet: 2001:db8:cafe::1/48
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

### Changes in configuration files of VPP-UPF

See [here](https://github.com/s5uishida/install_vpp_upf_dpdk#changes_up) for the original files.

- `openair-upf/startup.conf`  
There is no change.

- `openair-upf/init.conf`  
There is no change.

<a id="changes_ueransim"></a>

### Changes in configuration files of UERANSIM UE / RAN

<a id="changes_ran"></a>

#### Changes in configuration files of RAN

- `UERANSIM/config/open5gs-gnb.yaml`
```diff
--- open5gs-gnb.yaml.orig       2023-12-02 06:14:20.000000000 +0900
+++ open5gs-gnb.yaml    2024-03-24 20:33:09.114942556 +0900
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
--- open5gs-ue.yaml.orig        2023-12-02 06:14:20.000000000 +0900
+++ open5gs-ue.yaml     2024-03-24 20:36:02.968447533 +0900
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
@@ -28,7 +28,7 @@
 
 # List of gNB IP addresses for Radio Link Simulation
 gnbSearchList:
-  - 127.0.0.1
+  - 192.168.0.131
 
 # UAC Access Identities Configuration
 uacAic:
```

<a id="network_settings"></a>

## Network settings of Open5GS 5GC, VPP-UPF and UERANSIM UE / RAN

<a id="network_settings_up"></a>

### Network settings of VPP-UPF and Data Network Gateway

See [this1](https://github.com/s5uishida/install_vpp_upf_dpdk#setup_up) and [this2](https://github.com/s5uishida/install_vpp_upf_dpdk#setup_dn).

<a id="build"></a>

## Build Open5GS, VPP-UPF and UERANSIM

Please refer to the following for building Open5GS, VPP-UPF and UERANSIM respectively.
- Open5GS v2.7.0 (2024.03.24) - https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/
- UPG-VPP v1.12.0 (2024.01.25) - https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1
- UERANSIM v3.2.6 (2024.03.08) - https://github.com/aligungr/UERANSIM/wiki/Installation

Install MongoDB on Open5GS 5GC C-Plane machine.
[MongoDB Compass](https://www.mongodb.com/products/compass) is a convenient tool to look at the MongoDB database.

<a id="run"></a>

## Run Open5GS 5GC, VPP-UPF and UERANSIM UE / RAN

First run VPP-UPF, then the 5GC and UERANSIM (UE & RAN implementation).

<a id="run_up"></a>

### Run VPP-UPF

See [this](https://github.com/s5uishida/install_vpp_upf_dpdk#run_upg_vpp).

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

The status of PFCP association between VPP-UPF and Open5GS SMF is as follows.
```
vpp# show upf association 
Node: 192.168.14.111
  Recovery Time Stamp: 2024/03/24 21:37:00:000
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
UERANSIM v3.2.6
[2024-03-24 21:37:34.385] [sctp] [info] Trying to establish SCTP connection... (192.168.0.111:38412)
[2024-03-24 21:37:34.389] [sctp] [info] SCTP connection established (192.168.0.111:38412)
[2024-03-24 21:37:34.389] [sctp] [debug] SCTP association setup ascId[20]
[2024-03-24 21:37:34.389] [ngap] [debug] Sending NG Setup Request
[2024-03-24 21:37:34.395] [ngap] [debug] NG Setup Response received
[2024-03-24 21:37:34.395] [ngap] [info] NG Setup procedure is successful
```
The Open5GS C-Plane log when executed is as follows.
```
03/24 21:37:34.491: [amf] INFO: gNB-N2 accepted[192.168.0.131]:45106 in ng-path module (../src/amf/ngap-sctp.c:113)
03/24 21:37:34.491: [amf] INFO: gNB-N2 accepted[192.168.0.131] in master_sm module (../src/amf/amf-sm.c:754)
03/24 21:37:34.497: [amf] INFO: [Added] Number of gNBs is now 1 (../src/amf/context.c:1236)
03/24 21:37:34.497: [amf] INFO: gNB-N2[192.168.0.131] max_num_of_ostreams : 10 (../src/amf/amf-sm.c:793)
```

<a id="start_ue"></a>

#### Start UE

Start UE as follows. This will register the UE with 5GC and establish a PDU session.
```
# ./nr-ue -c ../config/open5gs-ue.yaml
UERANSIM v3.2.6
[2024-03-24 21:38:05.792] [nas] [info] UE switches to state [MM-DEREGISTERED/PLMN-SEARCH]
[2024-03-24 21:38:05.792] [rrc] [debug] New signal detected for cell[1], total [1] cells in coverage
[2024-03-24 21:38:05.792] [nas] [info] Selected plmn[001/01]
[2024-03-24 21:38:05.793] [rrc] [info] Selected cell plmn[001/01] tac[1] category[SUITABLE]
[2024-03-24 21:38:05.793] [nas] [info] UE switches to state [MM-DEREGISTERED/PS]
[2024-03-24 21:38:05.793] [nas] [info] UE switches to state [MM-DEREGISTERED/NORMAL-SERVICE]
[2024-03-24 21:38:05.793] [nas] [debug] Initial registration required due to [MM-DEREG-NORMAL-SERVICE]
[2024-03-24 21:38:05.794] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2024-03-24 21:38:05.794] [nas] [debug] Sending Initial Registration
[2024-03-24 21:38:05.794] [nas] [info] UE switches to state [MM-REGISTER-INITIATED]
[2024-03-24 21:38:05.794] [rrc] [debug] Sending RRC Setup Request
[2024-03-24 21:38:05.794] [rrc] [info] RRC connection established
[2024-03-24 21:38:05.795] [rrc] [info] UE switches to state [RRC-CONNECTED]
[2024-03-24 21:38:05.795] [nas] [info] UE switches to state [CM-CONNECTED]
[2024-03-24 21:38:05.802] [nas] [debug] Authentication Request received
[2024-03-24 21:38:05.802] [nas] [debug] Received SQN [000000000081]
[2024-03-24 21:38:05.802] [nas] [debug] SQN-MS [000000000000]
[2024-03-24 21:38:05.807] [nas] [debug] Security Mode Command received
[2024-03-24 21:38:05.807] [nas] [debug] Selected integrity[2] ciphering[0]
[2024-03-24 21:38:05.819] [nas] [debug] Registration accept received
[2024-03-24 21:38:05.819] [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
[2024-03-24 21:38:05.819] [nas] [debug] Sending Registration Complete
[2024-03-24 21:38:05.820] [nas] [info] Initial Registration is successful
[2024-03-24 21:38:05.820] [nas] [debug] Sending PDU Session Establishment Request
[2024-03-24 21:38:05.820] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2024-03-24 21:38:06.021] [nas] [debug] Configuration Update Command received
[2024-03-24 21:38:06.042] [nas] [debug] PDU Session Establishment Accept received
[2024-03-24 21:38:06.045] [nas] [info] PDU Session establishment is successful PSI[1]
[2024-03-24 21:38:06.068] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.45.0.2] is up.
```
The Open5GS C-Plane log when executed is as follows.
```
03/24 21:38:05.793: [amf] INFO: InitialUEMessage (../src/amf/ngap-handler.c:401)
03/24 21:38:05.794: [amf] INFO: [Added] Number of gNB-UEs is now 1 (../src/amf/context.c:2656)
03/24 21:38:05.794: [amf] INFO:     RAN_UE_NGAP_ID[1] AMF_UE_NGAP_ID[1] TAC[1] CellID[0x10] (../src/amf/ngap-handler.c:562)
03/24 21:38:05.794: [amf] INFO: [suci-0-001-01-0000-0-0-0000000000] Unknown UE by SUCI (../src/amf/context.c:1840)
03/24 21:38:05.794: [amf] INFO: [Added] Number of AMF-UEs is now 1 (../src/amf/context.c:1621)
03/24 21:38:05.794: [gmm] INFO: Registration request (../src/amf/gmm-sm.c:1224)
03/24 21:38:05.794: [gmm] INFO: [suci-0-001-01-0000-0-0-0000000000]    SUCI (../src/amf/gmm-handler.c:172)
03/24 21:38:05.796: [sbi] WARNING: [UDM] (NRF-discover) NF has already been added [35a21454-e9db-41ee-979c-43a8539608e3:1] (../lib/sbi/nnrf-handler.c:1162)
03/24 21:38:05.796: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.12:80] (../lib/sbi/context.c:2210)
03/24 21:38:05.796: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.12:7777] (../lib/sbi/context.c:1946)
03/24 21:38:05.796: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.12:7777] (../lib/sbi/context.c:1946)
03/24 21:38:05.796: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.12:7777] (../lib/sbi/context.c:1946)
03/24 21:38:05.797: [sbi] INFO: [UDM] (NF-discover) NF Profile updated [35a21454-e9db-41ee-979c-43a8539608e3:1] (../lib/sbi/nnrf-handler.c:1200)
03/24 21:38:05.799: [sbi] INFO: [UDM] (SCP-discover) NF registered [35a21454-e9db-41ee-979c-43a8539608e3:1] (../lib/sbi/path.c:211)
03/24 21:38:05.814: [sbi] WARNING: [UDR] (NRF-discover) NF has already been added [35a90c96-e9db-41ee-a8a8-9d616c965da9:1] (../lib/sbi/nnrf-handler.c:1162)
03/24 21:38:05.814: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.20:80] (../lib/sbi/context.c:2210)
03/24 21:38:05.814: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.20:7777] (../lib/sbi/context.c:1946)
03/24 21:38:05.815: [sbi] INFO: [UDR] (NF-discover) NF Profile updated [35a90c96-e9db-41ee-a8a8-9d616c965da9:1] (../lib/sbi/nnrf-handler.c:1200)
03/24 21:38:05.815: [sbi] INFO: [UDR] (SCP-discover) NF registered [35a90c96-e9db-41ee-a8a8-9d616c965da9:1] (../lib/sbi/path.c:211)
03/24 21:38:06.018: [gmm] INFO: [imsi-001010000000000] Registration complete (../src/amf/gmm-sm.c:2321)
03/24 21:38:06.018: [amf] INFO: [imsi-001010000000000] Configuration update command (../src/amf/nas-path.c:591)
03/24 21:38:06.018: [gmm] INFO:     UTC [2024-03-24T12:38:06] Timezone[0]/DST[0] (../src/amf/gmm-build.c:558)
03/24 21:38:06.019: [gmm] INFO:     LOCAL [2024-03-24T21:38:06] Timezone[32400]/DST[0] (../src/amf/gmm-build.c:563)
03/24 21:38:06.019: [amf] INFO: [Added] Number of AMF-Sessions is now 1 (../src/amf/context.c:2677)
03/24 21:38:06.019: [gmm] INFO: UE SUPI[imsi-001010000000000] DNN[internet] S_NSSAI[SST:1 SD:0xffffff] smContextRef [NULL] (../src/amf/gmm-handler.c:1285)
03/24 21:38:06.019: [gmm] INFO: SMF Instance [35b4b55a-e9db-41ee-a583-0f7331902c62] (../src/amf/gmm-handler.c:1324)
03/24 21:38:06.020: [smf] INFO: [Added] Number of SMF-UEs is now 1 (../src/smf/context.c:1019)
03/24 21:38:06.021: [smf] INFO: [Added] Number of SMF-Sessions is now 1 (../src/smf/context.c:3090)
03/24 21:38:06.022: [sbi] WARNING: [UDM] (NRF-discover) NF has already been added [35a21454-e9db-41ee-979c-43a8539608e3:1] (../lib/sbi/nnrf-handler.c:1162)
03/24 21:38:06.022: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.12:80] (../lib/sbi/context.c:2210)
03/24 21:38:06.022: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.12:7777] (../lib/sbi/context.c:1946)
03/24 21:38:06.022: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.12:7777] (../lib/sbi/context.c:1946)
03/24 21:38:06.022: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.12:7777] (../lib/sbi/context.c:1946)
03/24 21:38:06.023: [sbi] INFO: [UDM] (NF-discover) NF Profile updated [35a21454-e9db-41ee-979c-43a8539608e3:1] (../lib/sbi/nnrf-handler.c:1200)
03/24 21:38:06.025: [sbi] INFO: [UDM] (SCP-discover) NF registered [35a21454-e9db-41ee-979c-43a8539608e3:1] (../lib/sbi/path.c:211)
03/24 21:38:06.026: [sbi] WARNING: [PCF] (NRF-discover) NF has already been added [35a8fc1a-e9db-41ee-9093-63fe5e5c5710:1] (../lib/sbi/nnrf-handler.c:1162)
03/24 21:38:06.026: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.13:80] (../lib/sbi/context.c:2210)
03/24 21:38:06.026: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.13:7777] (../lib/sbi/context.c:1946)
03/24 21:38:06.027: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.13:7777] (../lib/sbi/context.c:1946)
03/24 21:38:06.027: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.13:7777] (../lib/sbi/context.c:1946)
03/24 21:38:06.027: [sbi] INFO: [PCF] (NF-discover) NF Profile updated [35a8fc1a-e9db-41ee-9093-63fe5e5c5710:1] (../lib/sbi/nnrf-handler.c:1200)
03/24 21:38:06.028: [sbi] WARNING: [UDR] (NRF-discover) NF has already been added [35a90c96-e9db-41ee-a8a8-9d616c965da9:1] (../lib/sbi/nnrf-handler.c:1162)
03/24 21:38:06.029: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.20:80] (../lib/sbi/context.c:2210)
03/24 21:38:06.029: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.20:7777] (../lib/sbi/context.c:1946)
03/24 21:38:06.029: [sbi] INFO: [UDR] (NF-discover) NF Profile updated [35a90c96-e9db-41ee-a8a8-9d616c965da9:1] (../lib/sbi/nnrf-handler.c:1200)
03/24 21:38:06.030: [sbi] WARNING: [UDR] (SCP-discover) NF has already been added [35a90c96-e9db-41ee-a8a8-9d616c965da9:2] (../lib/sbi/path.c:216)
03/24 21:38:06.031: [sbi] WARNING: [BSF] (NRF-discover) NF has already been added [35a19420-e9db-41ee-9cff-fd3474e91cf7:1] (../lib/sbi/nnrf-handler.c:1162)
03/24 21:38:06.031: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.15:80] (../lib/sbi/context.c:2210)
03/24 21:38:06.031: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.15:7777] (../lib/sbi/context.c:1946)
03/24 21:38:06.031: [sbi] INFO: [BSF] (NF-discover) NF Profile updated [35a19420-e9db-41ee-9cff-fd3474e91cf7:1] (../lib/sbi/nnrf-handler.c:1200)
03/24 21:38:06.032: [sbi] INFO: [BSF] (SCP-discover) NF registered [35a19420-e9db-41ee-9cff-fd3474e91cf7:1] (../lib/sbi/path.c:211)
03/24 21:38:06.033: [sbi] INFO: [PCF] (SCP-discover) NF registered [35a8fc1a-e9db-41ee-9093-63fe5e5c5710:1] (../lib/sbi/path.c:211)
03/24 21:38:06.034: [smf] INFO: UE SUPI[imsi-001010000000000] DNN[internet] IPv4[10.45.0.2] IPv6[] (../src/smf/npcf-handler.c:542)
03/24 21:38:06.037: [gtp] INFO: gtp_connect() [192.168.13.151]:2152 (../lib/gtp/path.c:60)
03/24 21:38:06.042: [sbi] WARNING: [UDM] (NRF-discover) NF has already been added [35a21454-e9db-41ee-979c-43a8539608e3:1] (../lib/sbi/nnrf-handler.c:1162)
03/24 21:38:06.042: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.12:80] (../lib/sbi/context.c:2210)
03/24 21:38:06.042: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.12:7777] (../lib/sbi/context.c:1946)
03/24 21:38:06.043: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.12:7777] (../lib/sbi/context.c:1946)
03/24 21:38:06.043: [sbi] WARNING: NF EndPoint(addr) updated [127.0.0.12:7777] (../lib/sbi/context.c:1946)
03/24 21:38:06.043: [sbi] INFO: [UDM] (NF-discover) NF Profile updated [35a21454-e9db-41ee-979c-43a8539608e3:1] (../lib/sbi/nnrf-handler.c:1200)
03/24 21:38:06.044: [sbi] WARNING: [UDM] (SCP-discover) NF has already been added [35a21454-e9db-41ee-979c-43a8539608e3:2] (../lib/sbi/path.c:216)
03/24 21:38:06.045: [amf] INFO: [imsi-001010000000000:1:11][0:0:NULL] /nsmf-pdusession/v1/sm-contexts/{smContextRef}/modify (../src/amf/nsmf-handler.c:867)
```
The PDU session establishment status of VPP-UPF is as follows.
```
vpp# show upf session 
CP F-SEID: 0x000000000000077b (1915) @ 192.168.14.111
UP F-SEID: 0x000000000000077b (1915) @ 192.168.14.151 (192.168.14.111 ::)
User ID: IMEI:4370816125816151
  PFCP Association: 0
  TEID assignment per choose ID
PDR: 1 @ 0x7f2fd70d4fc8
  Precedence: 255
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
  URR Ids: [1] @ 0x7f2fd70d7fe8
  QER Ids: [1] @ 0x7f2fd70dfc68
PDR: 2 @ 0x7f2fd70d5048
  Precedence: 255
  PDI:
    Fields: 0000000d
    Source Interface: Access
    Network Instance: internet
    Local F-TEID: 192923801 (0x0b7fc899)
            IPv4: 192.168.13.151
    UE IP address (source):
      IPv4 address: 10.45.0.2
    SDF Filter [1]:
      permit out ip from any to assigned 
  Outer Header Removal: GTP-U/UDP/IPv4
  FAR Id: 2
  URR Ids: [] @ 0x0
  QER Ids: [1] @ 0x7f2fd70e0d08
PDR: 3 @ 0x7f2fd70d50c8
  Precedence: 1000
  PDI:
    Fields: 00000001
    Source Interface: CP-function
    Network Instance: internet
    Local F-TEID: 315234596 (0x12ca1924)
            IPv4: 192.168.13.151
  Outer Header Removal: GTP-U/UDP/IPv4
  FAR Id: 1
  URR Ids: [] @ 0x0
  QER Ids: [1] @ 0x7f2fd70e0d28
PDR: 4 @ 0x7f2fd70d5148
  Precedence: 1
  PDI:
    Fields: 00000009
    Source Interface: Access
    Network Instance: internet
    Local F-TEID: 192923801 (0x0b7fc899)
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
  Start Time: 2024/03/24 21:38:05:986
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
[2024-03-24 21:38:06.068] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.45.0.2] is up.
```
Just in case, make sure it matches the IP address of the UE's TUNnel interface.
```
# ip addr show
...
12: uesimtun0: <POINTOPOINT,PROMISC,NOTRAILERS,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.45.0.2/32 scope global uesimtun0
       valid_lft forever preferred_lft forever
    inet6 fe80::e8ba:542d:857e:f37c/64 scope link stable-privacy 
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

Run `tcpdump` on VM-DN and check that the packet goes through N6 (enp0s9).
- `ping google.com` on VM3 (UE)
```
# ping google.com -I uesimtun0 -n
PING google.com (172.217.175.78) from 10.45.0.2 uesimtun0: 56(84) bytes of data.
64 bytes from 172.217.175.78: icmp_seq=2 ttl=59 time=44.4 ms
64 bytes from 172.217.175.78: icmp_seq=3 ttl=59 time=41.9 ms
64 bytes from 172.217.175.78: icmp_seq=4 ttl=59 time=35.2 ms
```
- Run `tcpdump` on VM-DN
```
# tcpdump -i enp0s9 -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21:41:06.316182 IP 10.45.0.2 > 172.217.175.78: ICMP echo request, id 12, seq 2, length 64
21:41:06.359640 IP 172.217.175.78 > 10.45.0.2: ICMP echo reply, id 12, seq 2, length 64
21:41:07.317358 IP 10.45.0.2 > 172.217.175.78: ICMP echo request, id 12, seq 3, length 64
21:41:07.358332 IP 172.217.175.78 > 10.45.0.2: ICMP echo reply, id 12, seq 3, length 64
21:41:08.318535 IP 10.45.0.2 > 172.217.175.78: ICMP echo request, id 12, seq 4, length 64
21:41:08.352850 IP 172.217.175.78 > 10.45.0.2: ICMP echo reply, id 12, seq 4, length 64
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
21:41:49.056521 IP 10.45.0.2.33399 > 172.217.26.238.80: Flags [S], seq 2623546508, win 65280, options [mss 1360,sackOK,TS val 2259322400 ecr 0,nop,wscale 7], length 0
21:41:49.133877 IP 172.217.26.238.80 > 10.45.0.2.33399: Flags [S.], seq 2944001, ack 2623546509, win 65535, options [mss 1460], length 0
21:41:49.134566 IP 10.45.0.2.33399 > 172.217.26.238.80: Flags [.], ack 1, win 65280, length 0
21:41:49.134787 IP 10.45.0.2.33399 > 172.217.26.238.80: Flags [P.], seq 1:75, ack 1, win 65280, length 74: HTTP: GET / HTTP/1.1
21:41:49.134862 IP 172.217.26.238.80 > 10.45.0.2.33399: Flags [.], ack 75, win 65535, length 0
21:41:49.250897 IP 172.217.26.238.80 > 10.45.0.2.33399: Flags [P.], seq 1:774, ack 75, win 65535, length 773: HTTP: HTTP/1.1 301 Moved Permanently
21:41:49.251695 IP 10.45.0.2.33399 > 172.217.26.238.80: Flags [.], ack 774, win 64507, length 0
21:41:49.253468 IP 10.45.0.2.33399 > 172.217.26.238.80: Flags [F.], seq 75, ack 774, win 64507, length 0
21:41:49.253608 IP 172.217.26.238.80 > 10.45.0.2.33399: Flags [.], ack 76, win 65535, length 0
21:41:49.314546 IP 172.217.26.238.80 > 10.45.0.2.33399: Flags [F.], seq 774, ack 76, win 65535, length 0
21:41:49.315231 IP 10.45.0.2.33399 > 172.217.26.238.80: Flags [.], ack 775, win 64507, length 0
```
Please note that the `ping` tool does not work with `nr-binder`. Please refer to [here](https://github.com/aligungr/UERANSIM/issues/186#issuecomment-729534464) for the reason.
You could now connect to the DN and send any packets on the network using VPP-UPF with DPDK.

---

Now you could work Open5GS 5GC with VPP-UPF.
I would like to thank the excellent developers and all the contributors of Open5GS, OpenAir CN 5G for UPF, UPG-VPP, VPP, DPDK and UERANSIM.

<a id="changelog"></a>

## Changelog (summary)

- [2024.03.24] Updated to UPG-VPP v1.12.0.
- [2023.11.10] Changed VPP-UPF from [oai-cn5g-upf-vpp](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf-vpp) to its base [travelping/upg-vpp](https://github.com/travelping/upg-vpp).
- [2023.06.18] Used the original without changing `init.conf`. This made the N3 and N4 networks separate like the original.
- [2023.06.15] Initial release.
