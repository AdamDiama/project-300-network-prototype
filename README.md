# Project 300: Secure Banking Network

> **Status:** Planning & Build Phase — Year 3, Semester 1 
> **Role:** Sole Network Lead (Team of 4)

## Overview
Designing and building a multi-VLAN network connected to a hardened back-end database for a high-security banking environment. Built as part of a 4-person team at ATU Sligo, with end-to-end responsibility for the physical network hardware, cabling, and network configuration.

## Planned Architecture & Scope
- **Physical Layer:** Rack assembly, structured cabling, and hardware deployment.
- **Network Layer:** Multi-VLAN segmentation, trunking, and routing across routers, switches, and firewall.
- **Backend Server Virtualisation:** Hosting three virtual/emulated servers on one physical server connected to the physical network:
  - **User Management Server:** Windows Server 2016 for managing user accounts and access.
  - **Database Server:** Hosting Oracle DB and SQL databases.
  - **Security Server:** Connected to the network for cybersecurity and monitoring functions.
- **Validation:** End-to-end connectivity and segmentation testing across all VLANs and virtualised hosts.

## Planned Hardware & Tech Stack
- **Networking Hardware:** 1 Router, 3 Switches, 1 Firewall
- **Servers & Virtualisation:** 1 Physical Server hosting 3 virtual/emulated servers (Windows Server 2016, Oracle DB, SQL, plus a network-connected security server)
- **Networking Concepts:** VLANs, Trunking, Connectivity Testing

## Current Progress
- [x] Project scoping, team role assignment, and initial architecture planning
- [ ] Physical rack build and cabling
- [ ] Switch, router, and firewall configuration
- [ ] Virtualised server deployment, backend integration, and connectivity testing
