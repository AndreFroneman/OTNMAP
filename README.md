OTNMAP v0.1

I created a working, Docker-based starter platform for Debian:

Download OTNMAP Starter v0.1

The Python source passed syntax compilation validation.

Included
otnmap command-line interface
FastAPI REST API
Basic dark-theme web dashboard
SQLite asset and evidence database
Passive ARP observation
Passive LLDP and CDP discovery
Passive OSPF and VRRP observation
Conservative TCP connect discovery
OPC UA GetEndpoints discovery
Authorized SNMPv3 ARP and routing-table collection
Asset-role classification
Confidence and evidence tracking
JSON, NDJSON, CSV and HTML exports
Dockerfile and Docker Compose deployment
OT protocol safety matrix
Product architecture and development roadmap
Initial automated safety-profile tests

OPC UA endpoint discovery is based on the standard OPC UA Discovery Service Set, which provides services for discovering servers, endpoints and endpoint security configuration.

Running it on Debian

Extract the archive:

unzip OTNMAP_starter_v0.1.zip
cd otnmap
cp .env.example .env
mkdir -p data

Build and start:

docker compose build
docker compose up -d

Docker officially supports Debian 13 Trixie, Debian 12 Bookworm and Debian 11 Bullseye.

Initialize the database:

docker compose exec otnmap otnmap init-db
docker compose exec otnmap otnmap status

Open the dashboard:

http://localhost:8080
Passive OT monitoring

Connect the Debian machine to a TAP or SPAN destination port and configure the capture interface in .env:

OTNMAP_INTERFACE=eth1

Capture for five minutes:

docker compose exec otnmap \
  otnmap passive --interface eth1 --seconds 300

Capture and preserve a PCAP:

docker compose exec otnmap \
  otnmap passive \
  --interface eth1 \
  --seconds 300 \
  --pcap /data/site-capture.pcap

Scapy provides the packet capture, decoding and packet-construction foundation used by this starter.

Safety-controlled discovery

OTNMAP has four profiles:

Profile	Behaviour
observe	Transmits nothing
fragile	One concurrent probe, long delays and strict request limits
conservative	Limited approved OT discovery
lab	Faster testing for non-production environments

Example fragile scan:

docker compose exec otnmap \
  otnmap discover 192.168.10.0/24 \
  --profile fragile \
  --ports 22,80,443,102,502,4840,44818,20000,2404,47808

The current active engine uses completed TCP connections rather than SYN stealth scanning. This is intentionally basic and predictable. Nmap itself offers extensive timing controls and safety-category scripts, but OTNMAP’s longer-term goal is to make OT safety policy mandatory rather than operator-dependent.

OPC UA discovery
docker compose exec otnmap \
  otnmap opcua opc.tcp://192.168.10.20:4840

It currently collects:

Endpoint URL
Message security mode
Security policy URI
Transport profile
Security level

It does not:

Create a normal application session
Browse the address space
Read process values
Create subscriptions
Call methods
Write values

That separation is important because OPC UA discovery can often establish useful identity and security information without interacting with the server’s operational data.

ARP and routing tables

There are two distinct collection methods.

Passive observation

OTNMAP can observe:

ARP requests and replies
IP-to-MAC associations
Gratuitous ARP
MAC movement over time
Duplicate-IP indicators
LLDP and CDP neighbours
OSPF traffic
VRRP traffic

This only captures information visible on the TAP or SPAN feed.

Authoritative device tables

For a complete ARP cache or routing table, OTNMAP needs authorized management access. The starter supports SNMPv3:

docker compose exec otnmap \
  otnmap snmp-tables 192.168.10.1 \
  --user otnmap_ro \
  --auth-key 'AUTHENTICATION_KEY' \
  --priv-key 'PRIVACY_KEY'

It attempts standard tables from:

IP-MIB neighbour tables
Legacy ipNetToMediaTable
IP-FORWARD-MIB route tables
Legacy IPv4 route tables

Some Hirschmann, Siemens, Rockwell, Cisco, Fortinet and other industrial network devices may require vendor-specific MIB adapters. Future modules should also support:

SSH command collectors
NETCONF
RESTCONF
FortiOS API
Cisco IOS/IOS-XE API
Siemens SCALANCE management interfaces
Hirschmann HiOS
Rockwell Stratix/Cisco-derived platforms

Credentials should eventually be loaded through Docker secrets or a vault rather than command-line arguments.

Exporting data

HTML:

docker compose exec otnmap \
  otnmap export \
  --format html \
  --output /data/otnmap-report.html

JSON:

docker compose exec otnmap \
  otnmap export \
  --format json \
  --output /data/otnmap-report.json

Other included formats:

csv
ndjson
html
json

The next export targets I recommend are:

GraphML topology maps
draw.io diagrams
CycloneDX OT asset inventory
STIX 2.1 observations
Splunk HEC
Microsoft Sentinel
QRadar LEEF
Elastic
ServiceNow CMDB
NetBox and Nautobot
IEC 62443 zone-and-conduit candidate reports
Proposed command structure

The mature product should grow toward:

otnmap observe
otnmap discover
otnmap verify
otnmap protocol opcua
otnmap protocol enip
otnmap protocol modbus
otnmap protocol s7
otnmap protocol bacnet
otnmap collect snmp
otnmap collect ssh
otnmap assets
otnmap topology
otnmap changes
otnmap explain
otnmap export
otnmap policy
otnmap simulate
AI features

The starter contains a deterministic classifier rather than a cloud AI dependency. It classifies probable roles from protocol evidence, for example:

TCP/102                   → likely Siemens PLC/controller
EtherNet/IP + TCP/44818   → likely Rockwell/CIP device
OPC UA                    → likely OPC UA server or gateway
DNP3 or IEC 104           → likely SCADA/protection device
LLDP/CDP/OSPF/VRRP        → likely network infrastructure

The next AI capabilities should be:

Asset identity reconciliation

Combine MAC address, IP history, protocol behaviour, hostname, serial number, OPC UA application URI and SNMP data into one durable asset identity.

Explainable classification

Produce conclusions such as:

This device is probably a Siemens controller because it communicates over RFC 1006 on TCP/102, has a Siemens OUI and exchanges S7 traffic with two engineering stations.

Change detection

Highlight:

New devices
Disappeared devices
New services
Firmware changes
MAC changes
Duplicate addresses
New cross-zone communications
Engineering workstation activity outside maintenance windows
IEC 62443 assistance

Suggest:

Candidate zones
Candidate conduits
Unexpected communications
Assets that may require higher security levels
Remote-access paths
Unmanaged Level 1 or Level 2 devices
Missing network boundaries
Local report assistant

An Ollama-based assistant could answer:

Why does OTNMAP think this is a PLC?
What changed since the previous capture?
Which devices communicate across the proposed cell boundary?
Create an IEC 62443 remediation list.
Show all devices using insecure OPC UA endpoints.

The local model should receive normalized metadata, not passwords, raw process values or full packet payloads.

Recommended next protocol modules

The safest implementation order is:

EtherNet/IP passive decoding and unicast ListIdentity
Modbus TCP passive decoding and supported Device Identification
Siemens S7 passive fingerprinting
BACnet/IP passive decoding
DNP3 passive decoding
IEC 60870-5-104 passive decoding
PROFINET DCP passive decoding
IEC 61850 MMS and GOOSE
Omron FINS
Mitsubishi MELSEC
HART-IP
OPC Classic visibility through DCE/RPC correlation

The core rule should remain: no generic probe is safe merely because it is read-only. Every active protocol module needs a documented packet budget, device-fragility list, stop conditions and simulator tests before production use.
