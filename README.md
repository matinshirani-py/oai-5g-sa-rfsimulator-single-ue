# OAI 5G SA RFSimulator — Single UE

A laboratory implementation and validation of a **5G Standalone (5G SA) network using OpenAirInterface (OAI) and RFSimulator**.

The project demonstrates the complete end-to-end procedure from **5G Core Network deployment** to **gNB/UE synchronization, Random Access, RRC connection, authentication, registration, PDU session establishment, and User Plane connectivity**.

---

## 1. Project Overview

This project implements a single-UE 5G Standalone network using:

* OpenAirInterface (OAI)
* OAI 5G Core Network
* OAI gNB
* OAI UE
* RFSimulator
* Docker / Docker Compose
* Linux networking tools
* ICMP ping for User Plane validation

The RF interface is simulated in software using **RFSimulator**, so no physical SDR hardware is required.

### Validation Flow

```text
5G Core Network
       │
       │ N2 / N3
       ▼
     gNB
       │
   RFSimulator
       │
       ▼
      UE
       │
       ▼
   PDU Session
       │
       ▼
   User Plane
       │
       ▼
    EXT-DN
```

The complete validation follows this sequence:

```text
Core Deployment
      ↓
gNB Startup
      ↓
UE Startup
      ↓
Cell Synchronization
      ↓
PBCH / SIB1
      ↓
Random Access
      ↓
RRC Connection
      ↓
Authentication
      ↓
Registration
      ↓
PDU Session Establishment
      ↓
UE IP Assignment
      ↓
User Plane Connectivity
      ↓
Ping Validation
```

---

# 2. Environment

The experiment was performed on a Linux environment using OpenAirInterface.

### Main Software

| Component        | Version / Configuration |
| ---------------- | ----------------------- |
| Operating System | Ubuntu 24.04            |
| Architecture     | amd64                   |
| Docker           | 29.7.2                  |
| Docker Compose   | v5.4.0                  |
| OpenAirInterface | `develop` branch        |
| OAI commit       | `42bf80e9b2`            |
| RF interface     | RFSimulator             |

> Note: Docker version information recorded in different sections of the original laboratory documentation is not completely consistent. The environment verification used for this single-UE setup is documented here as Docker 29.7.2 / Compose v5.4.0.

---

# 3. OpenAirInterface Build

The OAI RAN binaries were already built before running the experiment.

Build directory:

```bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
```

Verify the binaries:

```bash
ls -lh nr-softmodem nr-uesoftmodem
```

Expected result:

```text
nr-softmodem
nr-uesoftmodem
```

The experimental build produced:

* `nr-softmodem` ≈ 105 MB
* `nr-uesoftmodem` ≈ 46 MB

---

# 4. 5G Core Network

The 5G Core is deployed using Docker Compose.

Move to the OAI Core directory:

```bash
cd ~/oai-cn5g
```

Start the Core Network:

```bash
docker compose up -d
```

Check the running containers:

```bash
docker compose ps
```

The deployment contains the following network functions:

| Network Function | Role                          | IP Address     |
| ---------------- | ----------------------------- | -------------- |
| NRF              | NF discovery                  | 192.168.70.130 |
| AMF              | Registration / Mobility / NAS | 192.168.70.132 |
| SMF              | PDU Session Management        | 192.168.70.133 |
| UPF              | User Plane                    | 192.168.70.134 |
| EXT-DN           | External Data Network         | 192.168.70.135 |
| UDR              | Subscriber Data               | 192.168.70.136 |
| UDM              | Subscriber Management         | 192.168.70.137 |
| AUSF             | Authentication                | 192.168.70.138 |

Additional services include MySQL and IMS.

The Core network uses:

```text
Network: oai-cn5g-public-net
Subnet: 192.168.70.128/26
```

---

# 5. Core Network Verification

Check the AMF logs:

```bash
docker compose logs amf | grep -E "NRF|REGISTER|UPDATE"
```

The logs confirm that the AMF communicates with the NRF and successfully registers its NF instance.

Expected behavior includes:

```text
NF Update
REGISTERED
HTTP 204
```

This confirms that the Service-Based Architecture (SBA) control-plane communication is operational.

---

# 6. Start the gNB with RFSimulator

Move to the OAI RAN build directory:

```bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
```

Start the gNB:

```bash
sudo ./nr-softmodem \
-O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/gnb.sa.band78.fr1.106PRB.usrpb210.conf \
--gNBs.[0].min_rxtxtime 6 \
--rfsim
```

### Important Parameters

| Parameter                   | Meaning                    |
| --------------------------- | -------------------------- |
| `-O`                        | gNB configuration file     |
| `--gNBs.[0].min_rxtxtime 6` | RX/TX timing configuration |
| `--rfsim`                   | Enable RFSimulator         |

The gNB configuration uses:

* 5G NR Band 78
* 106 PRBs
* 30 kHz subcarrier spacing
* 3.6192 GHz carrier frequency
* PCI 0
* TDD configuration
* AMF address `192.168.70.132`
* RFSimulator port `4043`

---

# 7. Verify gNB Startup

After starting the gNB, the logs should show initialization of:

```text
PHY
MAC
RLC
PDCP
RRC
```

The RFSimulator server should become available on:

```text
Port: 4043
```

The gNB should also establish NGAP connectivity toward the AMF.

At this point the basic architecture is:

```text
        5G Core
           │
          N2
           │
           ▼
          gNB
           │
      RFSimulator
           │
           ▼
           UE
```

---

# 8. Start the UE

Open another terminal and move to the RAN build directory:

```bash
cd ~/openairinterface5g/cmake_targets/ran_build/build
```

Start the UE:

```bash
sudo ./nr-uesoftmodem \
-O ../../../targets/PROJECTS/GENERIC-NR-5GC/CONF/ue.conf \
-C 3619200000 \
-r 106 \
--numerology 1 \
--band 78 \
--ssb 516 \
--rfsim \
--uicc0.imsi 001010000000001
```

### UE Parameters

| Parameter         |           Value |
| ----------------- | --------------: |
| Carrier Frequency |      3.6192 GHz |
| PRBs              |             106 |
| Numerology        |               1 |
| SCS               |          30 kHz |
| NR Band           |             n78 |
| SSB               |             516 |
| IMSI              | 001010000000001 |
| RF Interface      |     RFSimulator |

---

# 9. Cell Synchronization

After starting the UE, the UE searches for the simulated NR cell.

The UE successfully decodes:

```text
PBCH
SIB1
```

Important information obtained from the logs includes:

```text
PCI = 0
SSB Index = 0
```

The UE also obtains the cell configuration including:

* Downlink frequency
* Uplink frequency
* TDD configuration
* Number of DL slots
* Number of UL slots

The synchronization stage confirms that the UE has successfully detected and synchronized with the simulated NR cell.

---

# 10. Random Access Procedure

After synchronization, the UE performs the Random Access procedure.

The experiment uses the **4-step Contention-Based Random Access (CBRA)** procedure.

The logs show:

```text
PRACH preamble
Random Access Response
RRC Request
Timing Advance
Contention Resolution
```

Example values recorded during the experiment:

```text
preambleIndex = 37
RAPID = 37
Timing Advance = 31
```

The procedure completes successfully:

```text
4-Step RA procedure succeeded
```

This confirms that the UE has successfully established initial uplink access to the gNB.

---

# 11. RRC Connection

Following Random Access, the UE establishes the Radio Resource Control connection.

The procedure includes:

```text
RRCSetup
SRB1 establishment
RRC_CONNECTED
RRCSetupComplete
```

The UE then sends the initial NAS Registration Request toward the Core Network.

At this point the radio connection between UE and gNB is established.

---

# 12. Authentication and Registration

The UE proceeds with 5G Core authentication and registration.

The signaling includes:

```text
Authentication Request
Security Mode Command
Registration Accept
```

The experiment used:

```text
NEA0
NIA2
```

After successful registration, the UE receives a:

```text
5G-GUTI
```

The UE is therefore successfully registered with the 5G Core Network.

---

# 13. PDU Session Establishment

After registration, the UE requests a PDU Session.

The network accepts the session and assigns an IPv4 address:

```text
UE IPv4 Address: 10.0.0.2
```

The UE creates the tunnel interface:

```text
oaitun_ue1
```

The PDU session uses:

```text
PDU Session ID: 1
```

The resulting user-plane path is:

```text
UE
 │
 │ oaitun_ue1
 │
 ▼
gNB
 │
 │ N3 / GTP-U
 ▼
UPF
 │
 │ N6
 ▼
EXT-DN
```

---

# 14. Verify the UE Tunnel

On the UE host, verify the tunnel interface:

```bash
ip addr show oaitun_ue1
```

The interface should have:

```text
10.0.0.2
```

The routing table can also be checked using:

```bash
ip route
```

---

# 15. User Plane Validation — Uplink

The first connectivity test is performed from the UE toward the External Data Network.

Run:

```bash
ping -I oaitun_ue1 -c 5 10.0.0.1
```

The experiment produced:

```text
5 packets transmitted
5 packets received
0% packet loss
```

Measured latency:

```text
Minimum: 4.860 ms
Average: 8.971 ms
Maximum: 10.890 ms
```

This confirms successful uplink User Plane connectivity.

---

# 16. User Plane Validation — Downlink

The reverse direction is also tested.

Enter the EXT-DN container:

```bash
docker exec -it oai-ext-dn bash
```

Then run:

```bash
ping -c 5 10.0.0.2
```

The experiment produced:

```text
5 packets transmitted
5 packets received
0% packet loss
```

Average latency:

```text
≈ 9.080 ms
```

This confirms successful downlink connectivity.

---

# 17. End-to-End Validation

The complete validation can therefore be summarized as:

| Stage                  | Result       |
| ---------------------- | ------------ |
| Linux environment      | PASS         |
| Docker                 | PASS         |
| 5G Core deployment     | PASS         |
| Core Network Functions | Healthy      |
| NRF registration       | PASS         |
| gNB startup            | PASS         |
| RFSimulator            | PASS         |
| RFsim port 4043        | PASS         |
| UE synchronization     | PASS         |
| PBCH                   | PASS         |
| SIB1                   | PASS         |
| Random Access          | PASS         |
| RRC_CONNECTED          | PASS         |
| Authentication         | PASS         |
| 5G Registration        | PASS         |
| PDU Session            | PASS         |
| UE IP                  | `10.0.0.2`   |
| UE tunnel              | `oaitun_ue1` |
| Uplink ping            | 0% loss      |
| Downlink ping          | 0% loss      |

---

# 18. Final Result

The experiment successfully demonstrated a complete **5G Standalone end-to-end connection using OpenAirInterface and RFSimulator**.

The UE successfully:

1. Detected the NR cell
2. Synchronized using PBCH/SIB1
3. Completed Random Access
4. Established RRC connection
5. Completed authentication
6. Registered with the 5G Core
7. Established a PDU Session
8. Received IPv4 address `10.0.0.2`
9. Created `oaitun_ue1`
10. Exchanged User Plane traffic with the External Data Network

The final connectivity validation achieved:

```text
Uplink:   0% packet loss
Downlink: 0% packet loss
```

Therefore, the complete single-UE 5G SA RFSimulator scenario was successfully validated in the laboratory.

---

# 19. Troubleshooting

## Check Docker Containers

```bash
docker compose ps
```

## View Core Logs

```bash
docker compose logs <network-function>
```

For example:

```bash
docker compose logs amf
```

## Enter a Container

```bash
docker exec -it <container-name> bash
```

Example:

```bash
docker exec -it oai-ext-dn bash
```

## Inspect a Container

```bash
docker inspect <container-name>
```

## Check the UE Interface

```bash
ip addr show oaitun_ue1
```

## Check Routing

```bash
ip route
```

---

# 20. References

* OpenAirInterface 5G
* OAI 5G Core Network
* 3GPP 5G NR specifications
* OpenAirInterface RFSimulator documentation

---

## Project Status

**Status: Successfully validated in a laboratory RFSimulator environment.**

This repository documents the execution, configuration, and validation procedure for a single-UE OAI 5G SA deployment.
