# k0s.platform

An Ansible collection that installs a multi-node [k0s](https://k0sproject.io)
Kubernetes cluster. Pick a topology with `k0s_topology`:

| `k0s_topology` | Topology | Datastore | Load balancing |
|----------------|----------|-----------|----------------|
| `single`       | One controller (controller + worker) | kine (SQLite) | none |
| `ha-cplb`      | 3/5 controllers | etcd | k0s CPLB (keepalived VIP on the controllers) |
| `ha-lb-single` | 3/5 controllers | etcd | one external LB (Envoy) |
| `ha-lb-ha`     | 3/5 controllers | etcd | 2+ external LBs (Envoy) + keepalived VIP |

The CNI is selected with `k0s_cni`: `kuberouter` (default), `calico`, or
`cilium` (deployed as a custom CNI via k0s's Helm extension, with kube-proxy
replacement).

## Roles

| Role | Target group | Purpose |
|------|--------------|---------|
| `k0s.platform.common`       | `k0s_cluster` | OS prerequisites, kernel modules, sysctl, k0s binary |
| `k0s.platform.loadbalancer` | `k0s_lb`      | Envoy/HAProxy (TCP) + optional keepalived |
| `k0s.platform.controller`   | `k0s_controllers` | Render config (CPLB/CNI), bootstrap primary, join secondaries |
| `k0s.platform.worker`       | `k0s_workers` | Join worker nodes |

See the repository README for usage, topology/CNI details and HA requirements.
