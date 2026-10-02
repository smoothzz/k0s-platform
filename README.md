# k0s-platform

An Ansible **collection** (`k0s.platform`) that stands up a multi-node
[k0s](https://k0sproject.io) Kubernetes cluster on Linux hosts. It supports
four topologies, selected with one variable (`k0s_topology`):

| `k0s_topology` | Controllers | Datastore | Load balancing | Extra hosts |
|---|---|---|---|---|
| `single` | 1 (all-in-one: controller + worker) | kine / SQLite | none | none |
| `ha-cplb` | 3 or 5 | etcd | **k0s CPLB** (keepalived VIP on the controllers) | none |
| `ha-lb-single` | 3 or 5 | etcd | **one external Envoy LB** (its own IP) | 1 LB |
| `ha-lb-ha` | 3 or 5 | etcd | **2+ external Envoy LBs** + keepalived VIP | 2+ LB |

Envoy is the external LB engine (HAProxy is available via `k0s_lb_engine`).
Envoy has no VRRP, so for the floating VIP keepalived is always the piece that
owns the address.

```
k0s-platform/
├── ansible.cfg                 # points at the local collection + inventory
├── requirements.yml            # ansible.posix, community.general
├── inventory/hosts.ini         # optional static inventory (nodes usually in group_vars)
├── group_vars/all.yml          # the only file you normally edit
├── playbooks/
│   ├── _inventory.yml          # builds groups from the k0s_*_hosts lists
│   ├── site.yml                # full deployment
│   ├── kubeconfig.yml          # fetch admin kubeconfig
│   └── k0s-versions.yml        # list available k0s versions
└── collections/ansible_collections/k0s/platform/   # the collection
    ├── galaxy.yml
    ├── meta/runtime.yml
    └── roles/
        ├── common/             # OS prep + k0s binary (all nodes)
        ├── loadbalancer/       # Envoy/HAProxy + keepalived (external LB only)
        ├── controller/         # bootstrap / join controllers, CPLB
        └── worker/             # join workers
```

## Quick start

On a Linux control node with Ansible installed:

```bash
ansible-galaxy collection install -r requirements.yml
```

1. Edit `group_vars/all.yml`: pick `k0s_topology`, list your nodes, set the VIP
   and SSH details.
2. Deploy (add `-k -K` if you use passwords instead of SSH keys):

```bash
ansible-playbook playbooks/site.yml
ansible-playbook playbooks/kubeconfig.yml   # optional: pull kubeconfig
```

## Find available k0s versions

Not sure what to put in `k0s_version`? List the published k0s releases:

```bash
ansible-playbook playbooks/k0s-versions.yml
ansible-playbook playbooks/k0s-versions.yml -e k0s_versions_filter=v1.31
ansible-playbook playbooks/k0s-versions.yml -e k0s_versions_include_prerelease=true
```

Then pin it in `group_vars/all.yml`, e.g. `k0s_version: v1.31.2+k0s.0` (leave it
as `latest` to always install the newest stable).

## Define your nodes

Everything lives in `group_vars/all.yml`. Nodes are lists:

```yaml
k0s_controllers_hosts:
  - { name: ctrl-1, ip: 192.168.1.11 }
  - { name: ctrl-2, ip: 192.168.1.12 }
  - { name: ctrl-3, ip: 192.168.1.13 }

k0s_workers_hosts:
  - { name: worker-1, ip: 192.168.1.21 }
  - { name: worker-2, ip: 192.168.1.22 }

k0s_lb_hosts:                   # only for ha-lb-single / ha-lb-ha
  - { name: lb-1, ip: 192.168.1.10 }
  - { name: lb-2, ip: 192.168.1.9 }

ansible_user: ubuntu
# ansible_password: "secret"
# ansible_become_password: "secret"
```

`playbooks/site.yml` turns these into the `k0s_controllers` / `k0s_workers` /
`k0s_lb` groups at runtime, validates them against the chosen topology, then
runs the roles. A static `inventory/hosts.ini` still works if you prefer it.

### Per topology

**`single`** — one master, no LB:

```yaml
k0s_topology: single
k0s_controllers_hosts:
  - { name: ctrl-1, ip: 192.168.1.11 }
k0s_workers_hosts:
  - { name: worker-1, ip: 192.168.1.21 }
k0s_lb_hosts: []
```

**`ha-cplb`** — 3 masters, k0s CPLB (keepalived VIP on the controllers), no LB
hosts. `k0s_lb_vip` is the VIP:

```yaml
k0s_topology: ha-cplb
k0s_lb_vip: 192.168.1.100
k0s_cplb_vip_cidr: 192.168.1.100/24
# k0s_cplb_interface: ens192     # optional; k0s auto-detects if empty
# k0s_cplb_unicast: true         # if multicast is blocked (e.g. VMware NAT)
k0s_lb_hosts: []
```

**`ha-lb-single`** — 3 masters, one external Envoy LB. `k0s_lb_vip` is that LB
host's own IP (no VIP takeover):

```yaml
k0s_topology: ha-lb-single
k0s_lb_vip: 192.168.1.10        # the single LB host
k0s_lb_hosts:
  - { name: lb-1, ip: 192.168.1.10 }
```

**`ha-lb-ha`** — 3 masters, 2+ external Envoy LBs with a floating VIP:

```yaml
k0s_topology: ha-lb-ha
k0s_lb_engine: envoy
k0s_lb_vip: 192.168.1.100
k0s_lb_interface: ens192        # NIC carrying the VIP (keepalived)
k0s_lb_hosts:
  - { name: lb-1, ip: 192.168.1.10 }
  - { name: lb-2, ip: 192.168.1.9 }
```

## Choose the CNI

`k0s_cni` selects the cluster network:

```yaml
k0s_cni: kuberouter   # kuberouter (default) | calico | cilium
```

- **`kuberouter`** — k0s built-in, BGP, no overlay, lowest overhead.
- **`calico`** — k0s built-in, VXLAN by default.
- **`cilium`** — deployed as a **custom** CNI via k0s's Helm extension with
  **kube-proxy replacement** (`spec.network.provider: custom` +
  `spec.network.kubeProxy.disabled: true`). k0s applies the chart at bootstrap;
  no `helm` CLI is needed.

Cilium options:

```yaml
k0s_cilium_version: "1.17.1"
k0s_cilium_kube_proxy_replacement: true
k0s_cilium_operator_replicas: 1
k0s_cilium_ipv4_mask_size: 24
# k0s_cilium_k8s_service_host: ""   # default: VIP (HA) or the first controller IP
# k0s_cilium_extra_values: {}       # deep-merged over the generated values
```

Cilium's `k8sServiceHost` is derived automatically (the VIP/`externalAddress`
when HA, otherwise the first controller's IP) and pod IPs come from
`k0s_pod_cidr` via `ipam.mode: cluster-pool`.

> Changing the CNI on a running cluster requires a full redeploy — k0s cannot
> switch providers in place.

## Install Argo CD

Set `k0s_argocd_enabled: true` and Argo CD is installed during bootstrap via
k0s's Helm extension (same mechanism as Cilium) — no `helm` CLI needed.

```yaml
k0s_argocd_enabled: true
k0s_argocd_version: "10.9.6"     # Helm chart version
k0s_argocd_namespace: argocd
k0s_argocd_values: {}            # extra chart values
```

The chart is applied after the CNI (order 1 vs 10), so its pods land on a ready
network. Expose the UI however you like, e.g.:

```yaml
k0s_argocd_values:
  server:
    service:
      type: NodePort
```

Retrieve the initial admin password after install:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d
```

## How each HA topology works

- **`ha-cplb`** — k0s renders `spec.network.controlPlaneLoadBalancing`
  (keepalived, userspace proxy on port 6444 with an iptables redirect from the
  VIP). No external LB host. `spec.api.externalAddress` is set to the VIP so all
  nodes and joining controllers use the floating address. Requires keepalived on
  the controllers (the role installs it).
- **`ha-lb-single`** — the `loadbalancer` role installs Envoy on the single LB
  host, TCP-proxying `6443`/`8132`/`9443` to every controller. `externalAddress`
  is the LB host's IP. Simple, but the LB is a single point of failure.
- **`ha-lb-ha`** — Envoy on each LB host plus keepalived (VRRP) providing a
  floating VIP across them. `externalAddress` is the VIP. No SPOF.

In all HA topologies every controller gets an identical `k0s.yaml` (only
`etcd.peerAddress` differs), the primary is bootstrapped first, then the others
join via a controller join token.

## What a highly available control plane needs

1. **An odd number of controllers (3 or 5)** — etcd quorum. The playbook asserts
   an odd count ≥ 3.
2. **A stable address** — either k0s CPLB (`ha-cplb`) or an external Envoy LB
   (`ha-lb-single` / `ha-lb-ha`), set as `spec.api.externalAddress`.
3. **Shared PKI** — handled by the controller join token
   (`k0s token create --role=controller`).
4. **etcd, not `--single`/kine** — HA roles use `storage.type: etcd`.
5. **Reachable, consistent addresses/hostnames** on all nodes.

### Why not run Envoy/HAProxy on the controllers?

The k0s controller already binds `6443` (apiserver), `8132` (konnectivity) and
`9443` (join API) on `0.0.0.0`, so an external proxy on the same host collides.
That's exactly what CPLB solves with its `6444` proxy + iptables redirect. So:
external proxy ⇒ non-controller hosts; LB on the masters ⇒ CPLB.

## Roles

| Role | Runs on | Purpose |
|------|---------|---------|
| `k0s.platform.common`       | `k0s_cluster` | swap off, kernel modules, sysctl, `/etc/hosts`, k0s binary |
| `k0s.platform.loadbalancer` | `k0s_lb`      | Envoy/HAProxy (TCP) + optional keepalived |
| `k0s.platform.controller`   | `k0s_controllers` | render config (incl. CPLB), bootstrap primary, join secondaries |
| `k0s.platform.worker`       | `k0s_workers` | fetch join token, install/start worker |

## Notable variables

| Variable | Default | Description |
|----------|---------|-------------|
| `k0s_topology` | `ha-lb-ha` | `single` / `ha-cplb` / `ha-lb-single` / `ha-lb-ha` |
| `k0s_version` | `latest` | k0s version, e.g. `v1.31.2+k0s.0` |
| `k0s_cluster_name` | `k0s` | Cluster name (ClusterConfig metadata) |
| `k0s_lb_vip` | `192.168.1.100` | The stable address (CPLB VIP, single-LB IP, or floating VIP) |
| `k0s_lb_interface` | `ens192` | NIC for the VIP (keepalived, `ha-lb-ha`) |
| `k0s_lb_engine` | `envoy` | External proxy: `envoy` or `haproxy` |
| `k0s_lb_envoy_version` | `latest` | Envoy release, or pin e.g. `1.39.1` |
| `k0s_lb_vrrp_id` | `51` | Keepalived VRRP router ID (must be unique on the network) |
| `k0s_cplb_vip_cidr` | `{{ k0s_lb_vip }}/24` | CPLB VIP with netmask |
| `k0s_cplb_auth_pass` | `k0s` | CPLB VRRP password |
| `k0s_cplb_interface` | `""` | CPLB VRRP NIC (empty = auto-detect) |
| `k0s_cplb_unicast` | `false` | Use unicast VRRP (networks without multicast, e.g. NAT) |
| `k0s_cplb_unicast_source_ip` | `""` | This controller's VRRP source IP (default: its primary IP) |
| `k0s_cplb_unicast_peers` | `[]` | Other controllers' IPs (default: derived from the node list) |
| `k0s_controller_enable_worker` | `false` | Also run workloads on HA controllers |
| `k0s_cni` | `kuberouter` | CNI: `kuberouter` / `calico` / `cilium` |
| `k0s_cilium_version` | `1.17.1` | Cilium chart version |
| `k0s_cilium_kube_proxy_replacement` | `true` | Cilium replaces kube-proxy |
| `k0s_cilium_operator_replicas` | `1` | Cilium operator replicas |
| `k0s_cilium_k8s_service_host` | `""` | Cilium API host (default: VIP or first controller) |
| `k0s_cilium_extra_values` | `{}` | Deep-merged over the generated Cilium values |
| `k0s_service_cidr` | `10.96.0.0/12` | Kubernetes service CIDR (`k0s_pod_cidr` is the pod CIDR) |
| `k0s_argocd_enabled` | `false` | Install Argo CD during bootstrap |
| `k0s_argocd_version` | `10.9.6` | Argo CD Helm chart version |
| `k0s_argocd_namespace` | `argocd` | Argo CD namespace |
| `k0s_argocd_values` | `{}` | Extra Argo CD chart values |
| `k0s_telemetry_enabled` | `false` | k0s telemetry |
| `k0s_config_extra` | `{}` | Deep-merged into the ClusterConfig `spec` |
| `k0s_manage_firewall` | `false` | Open k0s ports with ufw/firewalld |

## Building / distributing the collection

```bash
cd collections/ansible_collections/k0s/platform
ansible-galaxy collection build          # -> k0s-platform-1.0.0.tar.gz
ansible-galaxy collection install k0s-platform-1.0.0.tar.gz
```

## Notes

- Written for systemd-based Linux distros (Ubuntu/Debian/RHEL family).
- **CPLB/VRRP and networks:** keepalived uses VRRP multicast by default, which
  usually works on a **bridged** NIC but often not on **NAT**. Set
  `k0s_cplb_unicast: true` for unicast VRRP (source and peers are derived from
  the controller list automatically).
- Line endings are forced to LF via `.gitattributes`. If you copy the tree to a
  Linux box and see `\r` errors, run `find . -type f -exec dos2unix {} +`.
- Re-running `site.yml` is idempotent: services are only (re)installed when the
  rendered config changes.
