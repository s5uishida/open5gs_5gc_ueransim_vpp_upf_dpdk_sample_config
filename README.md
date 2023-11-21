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
- 5GC - Open5GS v2.6.6 (2023.11.10) - https://github.com/open5gs/open5gs
- VPP-UPF - UPG-VPP v1.10.0 (2023.11.10) - https://github.com/travelping/upg-vpp
- UE / RAN - UERANSIM v3.2.6 (2023.06.14) - https://github.com/aligungr/UERANSIM

Each VMs are as follows.  
| VM | SW & Role | IP address | OS | CPU<br>(Min) | Memory<br>(Min) | HDD<br>(Min) |
| --- | --- | --- | --- | --- | --- | --- |
| VM1 | Open5GS 5GC C-Plane | 192.168.0.111/24 | Ubuntu 22.04 | 1 | 1GB | 20GB |
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
- Open5GS v2.6.6 (2023.11.10) - https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/
- UPG-VPP v1.10.0 (2023.11.10) - https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1
- UERANSIM v3.2.6 (2023.06.14) - https://github.com/aligungr/UERANSIM/wiki/Installation

<a id="changes_cp"></a>

### Changes in configuration files of Open5GS 5GC C-Plane

The following parameters including DNN can be used in the logic that selects UPF as the connection destination by PFCP.

- DNN
- TAC (Tracking Area Code)
- nr_CellID

For the sake of simplicity, I used only DNN this time. Please refer to [here](https://github.com/open5gs/open5gs/pull/560#issue-483001043) for the logic to select UPF.

- `open5gs/install/etc/open5gs/amf.yaml`
```diff
--- amf.yaml.orig       2023-11-10 20:39:04.000000000 +0900
+++ amf.yaml    2023-11-10 20:48:05.148995242 +0900
@@ -474,26 +474,26 @@
       - addr: 127.0.0.5
         port: 7777
     ngap:
-      - addr: 127.0.0.5
+      - addr: 192.168.0.111
     metrics:
       - addr: 127.0.0.5
         port: 9090
     guami:
       - plmn_id:
-          mcc: 999
-          mnc: 70
+          mcc: 001
+          mnc: 01
         amf_id:
           region: 2
           set: 1
     tai:
       - plmn_id:
-          mcc: 999
-          mnc: 70
+          mcc: 001
+          mnc: 01
         tac: 1
     plmn_support:
       - plmn_id:
-          mcc: 999
-          mnc: 70
+          mcc: 001
+          mnc: 01
         s_nssai:
           - sst: 1
     security:
```
- `open5gs/install/etc/open5gs/smf.yaml`
```diff
--- smf.yaml.orig       2023-11-10 20:39:04.000000000 +0900
+++ smf.yaml    2023-11-10 20:48:09.380932509 +0900
@@ -602,25 +602,20 @@
       - addr: 127.0.0.4
         port: 7777
     pfcp:
-      - addr: 127.0.0.4
-      - addr: ::1
+      - addr: 192.168.14.111
     gtpc:
       - addr: 127.0.0.4
-      - addr: ::1
     gtpu:
-      - addr: 127.0.0.4
-      - addr: ::1
+      - addr: 192.168.14.111
     metrics:
       - addr: 127.0.0.4
         port: 9090
     subnet:
       - addr: 10.45.0.1/16
-      - addr: 2001:db8:cafe::1/48
+        dnn: internet
     dns:
       - 8.8.8.8
       - 8.8.4.4
-      - 2001:4860:4860::8888
-      - 2001:4860:4860::8844
     mtu: 1400
     ctf:
       enabled: auto
@@ -808,7 +803,8 @@
 #
 upf:
     pfcp:
-      - addr: 127.0.0.7
+      - addr: 192.168.14.151
+        dnn: internet
 
 #
 #  o Disable use of IPv4 addresses (only IPv6)
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
--- open5gs-gnb.yaml.orig       2022-07-03 13:06:44.000000000 +0900
+++ open5gs-gnb.yaml    2023-06-18 21:25:34.307164598 +0900
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
--- open5gs-ue.yaml.orig        2023-05-10 19:00:38.000000000 +0900
+++ open5gs-ue.yaml     2023-06-15 21:42:14.363706123 +0900
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
- Open5GS v2.6.6 (2023.11.10) - https://open5gs.org/open5gs/docs/guide/02-building-open5gs-from-sources/
- UPG-VPP v1.10.0 (2023.11.10) - https://github.com/s5uishida/install_vpp_upf_dpdk#annex_1
- UERANSIM v3.2.6 (2023.06.14) - https://github.com/aligungr/UERANSIM/wiki/Installation

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
  Recovery Time Stamp: 2023/11/10 21:15:07:000
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
[2023-11-10 21:15:43.834] [sctp] [info] Trying to establish SCTP connection... (192.168.0.111:38412)
[2023-11-10 21:15:43.837] [sctp] [info] SCTP connection established (192.168.0.111:38412)
[2023-11-10 21:15:43.838] [sctp] [debug] SCTP association setup ascId[4]
[2023-11-10 21:15:43.838] [ngap] [debug] Sending NG Setup Request
[2023-11-10 21:15:43.839] [ngap] [debug] NG Setup Response received
[2023-11-10 21:15:43.839] [ngap] [info] NG Setup procedure is successful
```
The Open5GS C-Plane log when executed is as follows.
```
11/10 21:15:43.835: [amf] INFO: gNB-N2 accepted[192.168.0.131]:59453 in ng-path module (../src/amf/ngap-sctp.c:113)
11/10 21:15:43.835: [amf] INFO: gNB-N2 accepted[192.168.0.131] in master_sm module (../src/amf/amf-sm.c:741)
11/10 21:15:43.835: [amf] INFO: [Added] Number of gNBs is now 1 (../src/amf/context.c:1203)
11/10 21:15:43.835: [amf] INFO: gNB-N2[192.168.0.131] max_num_of_ostreams : 10 (../src/amf/amf-sm.c:780)
```

<a id="start_ue"></a>

#### Start UE

Start UE as follows. This will register the UE with 5GC and establish a PDU session.
```
# ./nr-ue -c ../config/open5gs-ue.yaml
UERANSIM v3.2.6
[2023-11-10 21:16:30.704] [nas] [info] UE switches to state [MM-DEREGISTERED/PLMN-SEARCH]
[2023-11-10 21:16:30.704] [rrc] [debug] New signal detected for cell[1], total [1] cells in coverage
[2023-11-10 21:16:30.705] [nas] [info] Selected plmn[001/01]
[2023-11-10 21:16:30.705] [rrc] [info] Selected cell plmn[001/01] tac[1] category[SUITABLE]
[2023-11-10 21:16:30.705] [nas] [info] UE switches to state [MM-DEREGISTERED/PS]
[2023-11-10 21:16:30.705] [nas] [info] UE switches to state [MM-DEREGISTERED/NORMAL-SERVICE]
[2023-11-10 21:16:30.705] [nas] [debug] Initial registration required due to [MM-DEREG-NORMAL-SERVICE]
[2023-11-10 21:16:30.706] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2023-11-10 21:16:30.706] [nas] [debug] Sending Initial Registration
[2023-11-10 21:16:30.706] [rrc] [debug] Sending RRC Setup Request
[2023-11-10 21:16:30.707] [nas] [info] UE switches to state [MM-REGISTER-INITIATED]
[2023-11-10 21:16:30.707] [rrc] [info] RRC connection established
[2023-11-10 21:16:30.707] [rrc] [info] UE switches to state [RRC-CONNECTED]
[2023-11-10 21:16:30.708] [nas] [info] UE switches to state [CM-CONNECTED]
[2023-11-10 21:16:30.717] [nas] [debug] Authentication Request received
[2023-11-10 21:16:30.722] [nas] [debug] Security Mode Command received
[2023-11-10 21:16:30.722] [nas] [debug] Selected integrity[2] ciphering[0]
[2023-11-10 21:16:30.735] [nas] [debug] Registration accept received
[2023-11-10 21:16:30.735] [nas] [info] UE switches to state [MM-REGISTERED/NORMAL-SERVICE]
[2023-11-10 21:16:30.736] [nas] [debug] Sending Registration Complete
[2023-11-10 21:16:30.736] [nas] [info] Initial Registration is successful
[2023-11-10 21:16:30.736] [nas] [debug] Sending PDU Session Establishment Request
[2023-11-10 21:16:30.736] [nas] [debug] UAC access attempt is allowed for identity[0], category[MO_sig]
[2023-11-10 21:16:30.945] [nas] [debug] Configuration Update Command received
[2023-11-10 21:16:30.974] [nas] [debug] PDU Session Establishment Accept received
[2023-11-10 21:16:30.979] [nas] [info] PDU Session establishment is successful PSI[1]
[2023-11-10 21:16:31.003] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.45.0.2] is up.
```
The Open5GS C-Plane log when executed is as follows.
```
11/10 21:16:30.715: [amf] INFO: InitialUEMessage (../src/amf/ngap-handler.c:401)
11/10 21:16:30.715: [amf] INFO: [Added] Number of gNB-UEs is now 1 (../src/amf/context.c:2522)
11/10 21:16:30.715: [amf] INFO:     RAN_UE_NGAP_ID[1] AMF_UE_NGAP_ID[1] TAC[1] CellID[0x10] (../src/amf/ngap-handler.c:562)
11/10 21:16:30.715: [amf] INFO: [suci-0-001-01-0000-0-0-0000000000] Unknown UE by SUCI (../src/amf/context.c:1807)
11/10 21:16:30.715: [amf] INFO: [Added] Number of AMF-UEs is now 1 (../src/amf/context.c:1588)
11/10 21:16:30.715: [gmm] INFO: Registration request (../src/amf/gmm-sm.c:1165)
11/10 21:16:30.715: [gmm] INFO: [suci-0-001-01-0000-0-0-0000000000]    SUCI (../src/amf/gmm-handler.c:157)
11/10 21:16:30.718: [sbi] WARNING: [UDM] (NRF-discover) NF has already been added [c9ab0490-7fc2-41ee-9633-0fc4532caf73:1] (../lib/sbi/nnrf-handler.c:850)
11/10 21:16:30.718: [sbi] WARNING: NF EndPoint updated [127.0.0.12:80] (../lib/sbi/context.c:1743)
11/10 21:16:30.718: [sbi] WARNING: NF EndPoint updated [127.0.0.12:7777] (../lib/sbi/context.c:1621)
11/10 21:16:30.718: [sbi] WARNING: NF EndPoint updated [127.0.0.12:7777] (../lib/sbi/context.c:1621)
11/10 21:16:30.719: [sbi] WARNING: NF EndPoint updated [127.0.0.12:7777] (../lib/sbi/context.c:1621)
11/10 21:16:30.719: [sbi] INFO: [UDM] (NF-discover) NF Profile updated [c9ab0490-7fc2-41ee-9633-0fc4532caf73:1] (../lib/sbi/nnrf-handler.c:880)
11/10 21:16:30.722: [sbi] INFO: [UDM] (SCP-discover) NF registered [c9ab0490-7fc2-41ee-9633-0fc4532caf73:1] (../lib/sbi/path.c:149)
11/10 21:16:30.738: [sbi] WARNING: [UDR] (NRF-discover) NF has already been added [c9b0e95a-7fc2-41ee-9b25-3f8614f890eb:1] (../lib/sbi/nnrf-handler.c:850)
11/10 21:16:30.738: [sbi] WARNING: NF EndPoint updated [127.0.0.20:80] (../lib/sbi/context.c:1743)
11/10 21:16:30.738: [sbi] WARNING: NF EndPoint updated [127.0.0.20:7777] (../lib/sbi/context.c:1621)
11/10 21:16:30.738: [sbi] INFO: [UDR] (NF-discover) NF Profile updated [c9b0e95a-7fc2-41ee-9b25-3f8614f890eb:1] (../lib/sbi/nnrf-handler.c:880)
11/10 21:16:30.740: [sbi] INFO: [UDR] (SCP-discover) NF registered [c9b0e95a-7fc2-41ee-9b25-3f8614f890eb:1] (../lib/sbi/path.c:149)
11/10 21:16:30.950: [gmm] INFO: [imsi-001010000000000] Registration complete (../src/amf/gmm-sm.c:2097)
11/10 21:16:30.951: [amf] INFO: [imsi-001010000000000] Configuration update command (../src/amf/nas-path.c:612)
11/10 21:16:30.951: [gmm] INFO:     UTC [2023-11-10T12:16:30] Timezone[0]/DST[0] (../src/amf/gmm-build.c:558)
11/10 21:16:30.951: [gmm] INFO:     LOCAL [2023-11-10T21:16:30] Timezone[32400]/DST[0] (../src/amf/gmm-build.c:563)
11/10 21:16:30.952: [amf] INFO: [Added] Number of AMF-Sessions is now 1 (../src/amf/context.c:2543)
11/10 21:16:30.952: [gmm] INFO: UE SUPI[imsi-001010000000000] DNN[internet] S_NSSAI[SST:1 SD:0xffffff] smContextRef [NULL] (../src/amf/gmm-handler.c:1247)
11/10 21:16:30.952: [gmm] INFO: SMF Instance [c9bcfc0e-7fc2-41ee-97b5-45d4186e7900] (../src/amf/gmm-handler.c:1286)
11/10 21:16:30.954: [smf] INFO: [Added] Number of SMF-UEs is now 1 (../src/smf/context.c:1010)
11/10 21:16:30.954: [smf] INFO: [Added] Number of SMF-Sessions is now 1 (../src/smf/context.c:3057)
11/10 21:16:30.956: [sbi] WARNING: [UDM] (NRF-discover) NF has already been added [c9ab0490-7fc2-41ee-9633-0fc4532caf73:1] (../lib/sbi/nnrf-handler.c:850)
11/10 21:16:30.956: [sbi] WARNING: NF EndPoint updated [127.0.0.12:80] (../lib/sbi/context.c:1743)
11/10 21:16:30.956: [sbi] WARNING: NF EndPoint updated [127.0.0.12:7777] (../lib/sbi/context.c:1621)
11/10 21:16:30.956: [sbi] WARNING: NF EndPoint updated [127.0.0.12:7777] (../lib/sbi/context.c:1621)
11/10 21:16:30.957: [sbi] WARNING: NF EndPoint updated [127.0.0.12:7777] (../lib/sbi/context.c:1621)
11/10 21:16:30.957: [sbi] INFO: [UDM] (NF-discover) NF Profile updated [c9ab0490-7fc2-41ee-9633-0fc4532caf73:1] (../lib/sbi/nnrf-handler.c:880)
11/10 21:16:30.960: [sbi] INFO: [UDM] (SCP-discover) NF registered [c9ab0490-7fc2-41ee-9633-0fc4532caf73:1] (../lib/sbi/path.c:149)
11/10 21:16:30.961: [sbi] WARNING: [PCF] (NRF-discover) NF has already been added [c9b1a0a2-7fc2-41ee-bc55-0f4f6f3fb26f:1] (../lib/sbi/nnrf-handler.c:850)
11/10 21:16:30.961: [sbi] WARNING: NF EndPoint updated [127.0.0.13:80] (../lib/sbi/context.c:1743)
11/10 21:16:30.962: [sbi] WARNING: NF EndPoint updated [127.0.0.13:7777] (../lib/sbi/context.c:1621)
11/10 21:16:30.962: [sbi] WARNING: NF EndPoint updated [127.0.0.13:7777] (../lib/sbi/context.c:1621)
11/10 21:16:30.962: [sbi] WARNING: NF EndPoint updated [127.0.0.13:7777] (../lib/sbi/context.c:1621)
11/10 21:16:30.962: [sbi] INFO: [PCF] (NF-discover) NF Profile updated [c9b1a0a2-7fc2-41ee-bc55-0f4f6f3fb26f:1] (../lib/sbi/nnrf-handler.c:880)
11/10 21:16:30.964: [sbi] WARNING: [UDR] (NRF-discover) NF has already been added [c9b0e95a-7fc2-41ee-9b25-3f8614f890eb:1] (../lib/sbi/nnrf-handler.c:850)
11/10 21:16:30.964: [sbi] WARNING: NF EndPoint updated [127.0.0.20:80] (../lib/sbi/context.c:1743)
11/10 21:16:30.964: [sbi] WARNING: NF EndPoint updated [127.0.0.20:7777] (../lib/sbi/context.c:1621)
11/10 21:16:30.964: [sbi] INFO: [UDR] (NF-discover) NF Profile updated [c9b0e95a-7fc2-41ee-9b25-3f8614f890eb:1] (../lib/sbi/nnrf-handler.c:880)
11/10 21:16:30.965: [sbi] WARNING: [UDR] (SCP-discover) NF has already been added [c9b0e95a-7fc2-41ee-9b25-3f8614f890eb:2] (../lib/sbi/path.c:154)
11/10 21:16:30.966: [sbi] WARNING: [BSF] (NRF-discover) NF has already been added [c9aa2e12-7fc2-41ee-8fca-d574fcd41f2b:1] (../lib/sbi/nnrf-handler.c:850)
11/10 21:16:30.966: [sbi] WARNING: NF EndPoint updated [127.0.0.15:80] (../lib/sbi/context.c:1743)
11/10 21:16:30.966: [sbi] WARNING: NF EndPoint updated [127.0.0.15:7777] (../lib/sbi/context.c:1621)
11/10 21:16:30.967: [sbi] INFO: [BSF] (NF-discover) NF Profile updated [c9aa2e12-7fc2-41ee-8fca-d574fcd41f2b:1] (../lib/sbi/nnrf-handler.c:880)
11/10 21:16:30.968: [sbi] INFO: [BSF] (SCP-discover) NF registered [c9aa2e12-7fc2-41ee-8fca-d574fcd41f2b:1] (../lib/sbi/path.c:149)
11/10 21:16:30.969: [sbi] INFO: [PCF] (SCP-discover) NF registered [c9b1a0a2-7fc2-41ee-bc55-0f4f6f3fb26f:1] (../lib/sbi/path.c:149)
11/10 21:16:30.969: [smf] INFO: UE SUPI[imsi-001010000000000] DNN[internet] IPv4[10.45.0.2] IPv6[] (../src/smf/npcf-handler.c:539)
11/10 21:16:30.977: [gtp] INFO: gtp_connect() [192.168.13.151]:2152 (../lib/gtp/path.c:60)
11/10 21:16:30.988: [sbi] WARNING: [UDM] (NRF-discover) NF has already been added [c9ab0490-7fc2-41ee-9633-0fc4532caf73:1] (../lib/sbi/nnrf-handler.c:850)
11/10 21:16:30.988: [sbi] WARNING: NF EndPoint updated [127.0.0.12:80] (../lib/sbi/context.c:1743)
11/10 21:16:30.988: [sbi] WARNING: NF EndPoint updated [127.0.0.12:7777] (../lib/sbi/context.c:1621)
11/10 21:16:30.989: [sbi] WARNING: NF EndPoint updated [127.0.0.12:7777] (../lib/sbi/context.c:1621)
11/10 21:16:30.989: [sbi] WARNING: NF EndPoint updated [127.0.0.12:7777] (../lib/sbi/context.c:1621)
11/10 21:16:30.989: [sbi] INFO: [UDM] (NF-discover) NF Profile updated [c9ab0490-7fc2-41ee-9633-0fc4532caf73:1] (../lib/sbi/nnrf-handler.c:880)
11/10 21:16:30.990: [sbi] WARNING: [UDM] (SCP-discover) NF has already been added [c9ab0490-7fc2-41ee-9633-0fc4532caf73:2] (../lib/sbi/path.c:154)
11/10 21:16:30.991: [amf] INFO: [imsi-001010000000000:1:11][0:0:NULL] /nsmf-pdusession/v1/sm-contexts/{smContextRef}/modify (../src/amf/nsmf-handler.c:837)
```
The PDU session establishment status of VPP-UPF is as follows.
```
vpp# show upf session 
CP F-SEID: 0x0000000000000022 (34) @ 192.168.14.111
UP F-SEID: 0x0000000000000022 (34) @ 192.168.14.151
User ID: IMEI:4370816125816151
  PFCP Association: 0
  TEID assignment per choose ID
PDR: 1 @ 0x7f2389da6e78
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
  URR Ids: [1] @ 0x7f2389dacd58
  QER Ids: [1] @ 0x7f2389dacd78
PDR: 2 @ 0x7f2389da6ef8
  Precedence: 255
  PDI:
    Fields: 0000000d
    Source Interface: Access
    Network Instance: internet
    Local F-TEID: 500860926 (0x1dda87fe)
            IPv4: 192.168.13.151
    UE IP address (source):
      IPv4 address: 10.45.0.2
    SDF Filter [1]:
      permit out ip from any to assigned 
  Outer Header Removal: GTP-U/UDP/IPv4
  FAR Id: 2
  URR Ids: [] @ 0x0
  QER Ids: [1] @ 0x7f2389dacd98
PDR: 3 @ 0x7f2389da6f78
  Precedence: 1000
  PDI:
    Fields: 00000001
    Source Interface: CP-function
    Network Instance: internet
    Local F-TEID: 444927813 (0x1a850f45)
            IPv4: 192.168.13.151
  Outer Header Removal: GTP-U/UDP/IPv4
  FAR Id: 1
  URR Ids: [] @ 0x0
  QER Ids: [1] @ 0x7f2389dacdb8
PDR: 4 @ 0x7f2389da6ff8
  Precedence: 1
  PDI:
    Fields: 00000009
    Source Interface: Access
    Network Instance: internet
    Local F-TEID: 500860926 (0x1dda87fe)
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
  Start Time: 2023/11/10 21:16:31:099
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
[2023-11-10 21:16:31.003] [app] [info] Connection setup for PDU session[1] is successful, TUN interface[uesimtun0, 10.45.0.2] is up.
```
Just in case, make sure it matches the IP address of the UE's TUNnel interface.
```
# ip addr show
...
5: uesimtun0: <POINTOPOINT,PROMISC,NOTRAILERS,UP,LOWER_UP> mtu 1400 qdisc fq_codel state UNKNOWN group default qlen 500
    link/none 
    inet 10.45.0.2/32 scope global uesimtun0
       valid_lft forever preferred_lft forever
    inet6 fe80::4438:1328:ca47:4501/64 scope link stable-privacy 
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
PING google.com (142.251.42.206) from 10.45.0.2 uesimtun0: 56(84) bytes of data.
64 bytes from 142.251.42.206: icmp_seq=2 ttl=59 time=33.6 ms
64 bytes from 142.251.42.206: icmp_seq=3 ttl=59 time=18.5 ms
64 bytes from 142.251.42.206: icmp_seq=4 ttl=59 time=29.0 ms
```
- Run `tcpdump` on VM-DN
```
# tcpdump -i enp0s9 -n
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), snapshot length 262144 bytes
21:20:09.915799 IP 10.45.0.2 > 142.251.42.206: ICMP echo request, id 3, seq 2, length 64
21:20:09.948421 IP 142.251.42.206 > 10.45.0.2: ICMP echo reply, id 3, seq 2, length 64
21:20:10.916895 IP 10.45.0.2 > 142.251.42.206: ICMP echo request, id 3, seq 3, length 64
21:20:10.934595 IP 142.251.42.206 > 10.45.0.2: ICMP echo reply, id 3, seq 3, length 64
21:20:11.918218 IP 10.45.0.2 > 142.251.42.206: ICMP echo request, id 3, seq 4, length 64
21:20:11.946208 IP 142.251.42.206 > 10.45.0.2: ICMP echo reply, id 3, seq 4, length 64
```

You could specify the IP address assigned to the TUNnel interface to run almost any applications as in the following example using `nr-binder` tool.

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
21:21:14.239705 IP 10.45.0.2.43823 > 142.251.42.206.80: Flags [S], seq 3135597275, win 65280, options [mss 1360,sackOK,TS val 818843220 ecr 0,nop,wscale 7], length 0
21:21:14.256820 IP 142.251.42.206.80 > 10.45.0.2.43823: Flags [S.], seq 2752001, ack 3135597276, win 65535, options [mss 1460], length 0
21:21:14.258158 IP 10.45.0.2.43823 > 142.251.42.206.80: Flags [.], ack 1, win 65280, length 0
21:21:14.258158 IP 10.45.0.2.43823 > 142.251.42.206.80: Flags [P.], seq 1:75, ack 1, win 65280, length 74: HTTP: GET / HTTP/1.1
21:21:14.258390 IP 142.251.42.206.80 > 10.45.0.2.43823: Flags [.], ack 75, win 65535, length 0
21:21:14.313537 IP 142.251.42.206.80 > 10.45.0.2.43823: Flags [P.], seq 1:774, ack 75, win 65535, length 773: HTTP: HTTP/1.1 301 Moved Permanently
21:21:14.314969 IP 10.45.0.2.43823 > 142.251.42.206.80: Flags [.], ack 774, win 64507, length 0
21:21:14.315435 IP 10.45.0.2.43823 > 142.251.42.206.80: Flags [F.], seq 75, ack 774, win 64507, length 0
21:21:14.315508 IP 142.251.42.206.80 > 10.45.0.2.43823: Flags [.], ack 76, win 65535, length 0
21:21:14.331111 IP 142.251.42.206.80 > 10.45.0.2.43823: Flags [F.], seq 774, ack 76, win 65535, length 0
21:21:14.331974 IP 10.45.0.2.43823 > 142.251.42.206.80: Flags [.], ack 775, win 64507, length 0
```
Please note that the `ping` tool does not work with `nr-binder`. Please refer to [here](https://github.com/aligungr/UERANSIM/issues/186#issuecomment-729534464) for the reason.
You could now connect to the DN and send any packets on the network using VPP-UPF with DPDK.

---

Now you could work Open5GS 5GC with VPP-UPF.
I would like to thank the excellent developers and all the contributors of Open5GS, OpenAir CN 5G for UPF, UPG-VPP, VPP, DPDK and UERANSIM.

<a id="changelog"></a>

## Changelog (summary)

- [2023.11.10] Changed VPP-UPF from [oai-cn5g-upf-vpp](https://gitlab.eurecom.fr/oai/cn5g/oai-cn5g-upf-vpp) to its base [travelping/upg-vpp](https://github.com/travelping/upg-vpp).
- [2023.06.18] Used the original without changing `init.conf`. This made the N3 and N4 networks separate like the original.
- [2023.06.15] Initial release.
