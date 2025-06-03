# MikroTik Monitoring with Grafana Dashboard

A self-contained monitoring stack (Grafana + Prometheus + SNMP Exporter) deployed inside a MikroTik RouterOS v7+ container environment to collect and visualize device metrics using SNMP.

> This solution leverages the [MikroTik Container](https://help.mikrotik.com/docs/display/ROS/Container) feature introduced in RouterOS v7 to avoid the need for external servers.

---

## Table of Contents

* [Overview](#overview)
* [Prerequisites](#prerequisites)
* [Setup](#setup)
  * [1. Enable SNMP on MikroTik](#1-enable-snmp-on-mikrotik)
  * [2. Enable Container Support](#2-enable-container-support)
  * [3. Configure VETH and Network](#3-configure-veth-and-network)
  * [4. Prepare Configuration Files](#4-prepare-configuration-files)
  * [5. Define Environment Variables and Mounts](#5-define-environment-variables-and-mounts)
  * [6. Deploy and Start Containers](#6-deploy-and-start-containers)
* [Accessing the Services](#accessing-the-services)
* [Known Issues & Troubleshooting](#known-issues--troubleshooting)
* [References](#references)

---

## Overview

This project demonstrates how to deploy a complete monitoring stack directly on a MikroTik router using containers. It consists of:

* **SNMP Exporter** for retrieving SNMP metrics.
* **Prometheus** as the time-series database and metrics scraper.
* **Grafana** for dashboard visualization.

All components run within the MikroTik device itself, avoiding the cost and complexity of external infrastructure.

![Grafana Dashboard Screenshot](https://github.com/IgorKha/Grafana-Mikrotik/blob/master/readme/screen.png)

---

## Prerequisites

* MikroTik RouterOS **v7+** with **Container** feature support.
* A device with sufficient resources (e.g., RouterBoard L009UiGS ARM).
* Basic familiarity with WinBox or CLI.
* External storage (e.g., USB drive) recommended for container volumes.

---

## Setup

### 1. Enable SNMP on MikroTik

* Navigate to **IP > SNMP**
* Enable SNMP (tick the checkbox), leave defaults.

---

### 2. Enable Container Support

#### a. Check if Container is enabled:

From a **New Terminal** in WinBox:

```bash
/system/device-mode/print
```

Ensure the output includes `container: yes`.

#### b. If not enabled:

1. **Download**:

   * Visit [MikroTik Download Archive](https://mikrotik.com/download/archive)
   * Select your RouterOS version and architecture (from WinBox in System > Resources)
   * Download the `all_packages` ZIP

2. **Install**:

   * Extract `container-*.npk` from the ZIP
   * Upload it to **Files** via WinBox
   * Reboot the device

3. **Activate**:

   * Run: `system/device-mode/update container=yes`
   * Perform a full shutdown (unplug power or press the power button of the physical device)
   * Verify again with: `system/device-mode/print`

---

### 3. Configure VETH and Network

> After each step press Apply and then OK buttons.

#### a. Create a Docker bridge:

1. **Bridge > Bridge**: Add new interface named `docker`, leave defaults
2. **IP > Addresses**: Add Address `192.168.50.1/24`, Network `192.168.50.0`, Interface `docker`
3. **IP > Firewall > NAT**:

   * Chain: `srcnat`
   * Src. Address: `192.168.50.0/24`
   * Action: `masquerade`

#### b. Add VETH interfaces:

* Interface → VETH tab, add 3 interfaces (Name → Address, Gateway):

  * `grafana` → `192.168.50.100/24`, `192.168.50.1`
  * `snmp_exporter` → `192.168.50.101/24`, `192.168.50.1`
  * `prometheus` → `192.168.50.102/24`, `192.168.50.1`

#### c. Bridge the interfaces:

* Bridge > Ports: Create a **New Bridge Port** for each VETH interface and add it to the `docker` bridge.

---

### 4. Prepare Configuration Files

1. Clone this repository and extract `grafana/`, `prometheus/`, `snmp/` folders.
2. Edit the following:

   * **Grafana Datasource** (`grafana/provisioning/datasources.yml`):
	
     ```yaml
     url: http://192.168.50.102:9090 # change it according with your prometheus VETH IP
     ```
   * **Prometheus Config** (`prometheus/prometheus.yml`):

     * `static_configs.targets`: _use your MikroTik LAN IP_
     * `relabel_configs.replacement`: `192.168.50.101:9116` (change it according with your snmp_exporter VETH IP)
3. Upload `grafana/`, `prometheus/`, `snmp/` folders to MikroTik via **Files**.

---

### 5. Define Environment Variables and Mounts

#### a. Container > Envs (Grafana only):

| Key                           | Value    |
| ----------------------------- | -------- |
| GF\_SECURITY\_ADMIN\_USER     | admin    |
| GF\_SECURITY\_ADMIN\_PASSWORD | mikrotik |
| GF\_SECURITY\_SIGN\_UP        | false    |

#### b. Container > Mounts:
> Use your own `src path` (e.g. path to an external usb drive attached to MikroTik)

| Container  | Src Path                     | Dst Path                     |
| ---------- | ---------------------------- | ---------------------------- |
| grafana    | `/usb1/grafana/provisioning` | `/etc/grafana/provisioning/` |
| prometheus | `/usb1/prometheus`           | `/etc/prometheus`            |
| snmp       | `/usb1/snmp`                 | `/etc/snmp_exporter`         |

---

### 6. Deploy and Start Containers

#### a. Configure Container settings:

**Container > Container > Config:**
* Registry URL: `https://registry-1.docker.io`
* Tmp Dir: `/pull` (customizable e.g. path to an external usb drive attached to MikroTik)

#### b. Create containers:

> After you fill out the fields of each container press Apply and **wait for download/extracting** until in status appear `stopped` (if it appear is all OK) then go to create the next container. If it shows `error`, Copy the container, delete the old, and re-try to Apply.

**Container > New Container**
| Container         | Remote Image                       | Interface            | Others                                                                |
| ----------------- | --------------------------- | --------------- | -------------------------------------------------------------------- |
| **Grafana**       | `grafana/grafana:9.1.0`     | `grafana`       | Envslist `grafana`, Mounts `grafana`, Root Dir `/usb1/grafana_dir`                        |
| **Prometheus**    | `prom/prometheus:v2.53.0`    | `prometheus`    | Cmd: `--config.file=/etc/prometheus/prometheus.yml`, Mounts `prometheus`, Root Dir `/usb1/prometheus_dir`|
| **SNMP Exporter** | `prom/snmp-exporter:v0.26.0` | `snmp_exporter` | Cmd: `--config.file=/etc/snmp_exporter/snmp.yml`, Mounts `snmp`, Root Dir `/usb1/snmp_dir` |

> `Root Dir` is customizable e.g. path to an external usb drive attached to MikroTik

---

## Accessing the Services
> Ensure to start containers in order (SNMP Exporter, Prometheus and Grafana) and enjoy.  
> Start and Stop the container using the respective button.  
> Ensure the router and host are on the same LAN or routing is configured properly.

| Service       | URL                                                      |
| ------------- | -------------------------------------------------------- |
| Grafana       | [http://192.168.50.100:3000](http://192.168.50.100:3000) |
| Prometheus    | [http://192.168.50.102:9090](http://192.168.50.102:9090) |
| SNMP Exporter | [http://192.168.50.101:9116](http://192.168.50.101:9116) |

Grafana credentials: `admin / mikrotik`

---

## Known Issues & Troubleshooting

* **Grafana “No Data” Bug on ARM**:

  * Try increasing `$scrape_interval` to 30s in the dashboard query.
  * Fixed in Grafana ≥ 9.1.0

* **Container Store Errors**:

  * Ensure container root dirs are unique.
  * Deleting a container deletes its root directory.
  * Always verify the root dir is marked as `container store` in Files. (and **not as `directory`**)

* **Startup Failures**:

  * If container fails with `execve: No such file or directory`, verify paths, mounts, and retry with a different root dir.

---

## References

* Configuration and dashboard by [@IgorKha](https://github.com/IgorKha/): [Dashboard #14420](https://grafana.com/grafana/dashboards/14420-mikrotik-monitoring/)
* Special thanks to @rolando for guidance on MikroTik world.
