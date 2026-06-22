# Crucible Appliance

A virtual appliance for building cyber labs, challenges and competitions

## Overview

Crucible Appliance is a virtual machine that integrates cyber workforce development apps from the [Software Engineering Institute](https://www.sei.cmu.edu) at [Carnegie Mellon University](https://www.cmu.edu).

This project builds the virtual appliance using Ubuntu and [K3s](https://k3s.io/)&mdash;a lightweight Kubernetes environment. Pre-built OVA images are also available under [Releases](https://github.com/cmu-sei/crucible-appliance/releases).

## Getting Started

Download a pre-built OVA from [Releases](https://github.com/cmu-sei/crucible-appliance/releases), then import it into your hypervisor.

### Deploy on Proxmox

1. In the Proxmox **Datacenter** view, select **Storage**. You need a storage volume that allows the **Disk Image** and **Import** content types (for example, a volume named `Import`).
2. Open that storage volume and choose **Import > Download from URL**, then paste the OVA URL so Proxmox downloads it. (Alternatively, copy the OVA directly into the folder the datastore points to on disk instead of downloading through Proxmox.)
3. Once the OVA appears in the Import volume, select it and click **Import** to create a VM. Accept the default settings in the dialog, then power on the VM.
4. Open the VM **Console** and run `ip a | less` to find the IP address assigned to the VM.

> **Note:** See the [Proxmox documentation on Importing VMs for more details](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#qm_import_virtual_machines).

### Access the appliance

If your network does not resolve `.local` domains automatically, add a hosts file entry mapping `crucible.local` to the VM's IP address:

- **Linux/macOS:** edit `/etc/hosts`
- **Windows:** edit `C:\Windows\System32\drivers\etc\hosts`

```
<vm-ip-address>  crucible.local
```

Then visit https://crucible.local to begin using the apps. The appliance uses a self-signed certificate, so your browser will warn about the connection the first time — accept it to continue.

You can log in to the VM console with credentials:

```
username: crucible
password: crucible
```

> **Note:** On first boot the appliance installs K3s and deploys the full application stack via Helm. This can take several minutes after the VM powers on before https://crucible.local responds. You can check K3s deployment status using the command `kubectl get pods` (or the `kls` [k-alias](https://github.com/jaggedmountain/k-alias) command) and waiting for all Crucible pods to be listed as "Running".

## Apps

The following apps are deployed on the appliance, all accessible under `https://crucible.local`:

| App                                                                 | Path           | Description                        |
| ------------------------------------------------------------------- | -------------- | ---------------------------------- |
| [Keycloak](https://www.keycloak.org/)                               | `/keycloak`    | OIDC identity provider             |
| [TopoMojo](https://github.com/cmu-sei/topomojo)                     | `/topomojo`    | Virtual lab builder and player     |
| [Gameboard](https://github.com/cmu-sei/gameboard)                   | `/gameboard`   | Competition manager                |
| [Player](https://github.com/cmu-sei/crucible/wiki/player)           | `/player`      | Exercise presentation platform     |
| [Alloy](https://github.com/cmu-sei/crucible/wiki/alloy)             | `/alloy`       | Just-in-time lab deployment        |
| [Blueprint](https://github.com/cmu-sei/crucible/wiki/blueprint)     | `/blueprint`   | Exercise template editor           |
| [Caster](https://github.com/cmu-sei/crucible/wiki/caster)           | `/caster`      | Infrastructure-as-code environment |
| [CITE](https://github.com/cmu-sei/crucible/wiki/cite)               | `/cite`        | Incident tabletop evaluator        |
| [Gallery](https://github.com/cmu-sei/crucible/wiki/gallery)         | `/gallery`     | Information feed and reporting     |
| [Steamfitter](https://github.com/cmu-sei/crucible/wiki/steamfitter) | `/steamfitter` | Scripted scenario automation       |
| [Moodle](https://moodle.org/)                                       | `/moodle`      | Learning management system         |
| [Gitea](https://gitea.io/)                                          | `/gitea`       | Git server for content hosting     |
| [MkDocs](https://www.mkdocs.org/)                                   | `/start`       | Documentation site                 |
| [pgAdmin](https://www.pgadmin.org/)                                 | `/pgadmin`     | PostgreSQL database management     |

You can toggle which apps you want enabled by setting `enabled: false` for apps that you do not want deployed in the [Crucible values.yaml](crucible/charts/crucible/values.yaml). On a deployed appliance, edit `/home/crucible/charts/crucible/values.yaml` and then apply the change:

```bash
cd /home/crucible/charts/crucible
helm upgrade -n crucible crucible . --set global.version=$(cat /etc/appliance_version)
```

If the appliance is resource-constrained, disable unused apps or increase the VM's CPU/RAM.

## Helm Charts

The appliance uses a three-chart deployment model installed into the `crucible` namespace:

1. **`operators`** — Operator prerequisites; installs the Keycloak Operator and CloudNative-PG cluster-wide.
2. **`infra`** — Infrastructure chart; installs cert-manager (self-signed CA), ingress-nginx, PostgreSQL, NFS storage provisioner, pgAdmin, and all pre-created secrets.
3. **`crucible`** — Application chart; wraps the upstream `sei/crucible-apps` chart (Keycloak + all Crucible apps) along with Gitea and MkDocs as subchart dependencies.

To upgrade the charts after modifying values or templates on a deployed appliance:

```bash
helm upgrade -n crucible crucible-operators /home/crucible/charts/operators
helm upgrade -n crucible crucible-infra /home/crucible/charts/infra
helm upgrade -n crucible crucible /home/crucible/charts/crucible --set global.version=$(cat /etc/appliance_version)
```

See [`crucible/charts/README.md`](crucible/charts/README.md) for detailed chart architecture documentation.

## Build

To build the appliance, you will need:

- [Packer](https://www.packer.io/) 1.7+
- A compatible hypervisor:
  - [VirtualBox](https://www.virtualbox.org/) (`virtualbox`)
  - [Proxmox Virtual Environment](https://www.proxmox.com/en/products/proxmox-virtual-environment/overview) (`proxmox`)

### Proxmox Build (optional)

To build the appliance using Proxmox, create a file named `proxmox.auto.pkrvars.hcl` in this directory and add these settings:

```
proxmox_url      = "https://<proxmox.fqdn>:8006/api2/json" # replace with your PVE server
proxmox_username = "root@pam"
proxmox_password = "<password>"
proxmox_node     = "pve.lan" # replace with the Proxmox node name that should build the appliance
```

### Build Script

Run the following command, where `<hypervisor>` is a comma-delimited list of target hypervisors:

```
./build-appliance.sh <hypervisor>
```

For example, to build the appliance with VirtualBox, run this command:

```
./build-appliance.sh virtualbox
```

To add Proxmox to the previous build, run this command:

```
./build-appliance.sh virtualbox,proxmox
```

[Packer `build` options](https://www.packer.io/docs/commands/build) can be appended to the end of the command. For example, this will save partial builds and automatically overwrite the previous build (useful for debugging):

```
./build-appliance.sh <hypervisor> -on-error=abort -force
```

### CD-ROM Autoinstall (`use_cidata`)

By default, Packer starts a local HTTP server to serve the Ubuntu autoinstall configuration. This requires the target hypervisor to have network access back to the machine running Packer.

If you are building from an environment where the hypervisor cannot reach Packer's HTTP server (e.g., a dev container, a CI runner behind NAT, or a remote Proxmox host), set `use_cidata` to deliver the autoinstall data via a mounted CD-ROM instead:

```
./build-appliance.sh proxmox -var use_cidata=true
```

Or add it to your `proxmox.auto.pkrvars.hcl`:

```
use_cidata = true
```
