# vagrant-ansible-kubeadm-env-setup

Three Ubuntu 24.04 VMs (`node01`, `node02`, `controlplane`) provisioned into a
kubeadm + Cilium Kubernetes cluster. Ansible runs **inside** the `controlplane`
VM (Vagrant's `ansible_local` provisioner), so the host needs no Ansible.

| Host                  | Provider   | Networking                                   |
|-----------------------|------------|----------------------------------------------|
| Apple Silicon Mac     | Multipass  | Multipass NAT (dynamic IPs)                  |
| Windows               | Hyper-V    | Bridged to your LAN, `192.168.0.101`–`103`   |
| Linux / Intel Mac     | VirtualBox | Bridged to your LAN, `192.168.0.101`–`103`   |

The provider is chosen automatically from the host. Override it with the
`K8S_PROVIDER` environment variable (`multipass`, `hyperv` or `virtualbox`).

## Prerequisites

### Windows (PowerShell)
- Windows 10/11 Pro or Enterprise and [Vagrant](https://developer.hashicorp.com/vagrant/install).
- One-time setup, from an **Administrator** PowerShell, then reboot:
  ```powershell
  Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
  Add-LocalGroupMember -Group "Hyper-V Administrators" -Member $env:USERNAME
  ```
  Being in "Hyper-V Administrators" lets you run `vagrant` from a normal
  (non-admin) PowerShell afterwards.
- On the first `vagrant up`, if there's no External Hyper-V switch yet, one
  called `k8s-external` is created on your LAN adapter. The host's network
  drops for a few seconds while that happens. To use a different switch, set
  `$env:K8S_HYPERV_SWITCH = "<name>"`.

Why not VirtualBox on Windows: whenever the Windows hypervisor is running
(WSL2, Docker Desktop, Virtualization-Based Security), VirtualBox runs on top
of it and loses timer interrupts in multi-CPU VMs, so VMs hang during boot or
freeze for minutes. If you've disabled the hypervisor entirely
(`bcdedit /set hypervisorlaunchtype off`), VirtualBox works fine:
`$env:K8S_PROVIDER = "virtualbox"`.

### Apple Silicon Mac
- Multipass: `brew install --cask multipass`
- The `vagrant-multipass` plugin (not on RubyGems, build from source):
  ```sh
  git clone https://github.com/Fred78290/vagrant-multipass.git
  cd vagrant-multipass && gem build vagrant-multipass.gemspec
  vagrant plugin install ./vagrant-multipass-*.gem
  ```
- An SSH keypair Vagrant logs in with: `ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa`

## Configuration

Edit the top of the `Vagrantfile`:
- `NODE_IPS` — VirtualBox and Hyper-V. Must be free addresses on your LAN,
  outside the router's DHCP range. (Hyper-V nodes also keep a DHCP address;
  Kubernetes uses the static one.)
- `NODE_SPECS` — memory / CPUs per node.

VirtualBox auto-detects the LAN adapter to bridge onto. If it picks the wrong
one, set it explicitly with a name from `VBoxManage list bridgedifs`:
```powershell
$env:K8S_BRIDGE = "Intel(R) Ethernet Connection I219-V"
```

## Usage

```sh
vagrant up                                   # boots node01, node02, then controlplane, which runs Ansible
vagrant ssh -c "kubectl get nodes -o wide"   # controlplane is the default machine
vagrant provision controlplane               # re-run the playbook
vagrant destroy -f
```

Always bring up the whole cluster (plain `vagrant up`): `controlplane`
provisions the workers, so they have to be running first.

## How it works

1. On first run the Vagrantfile generates a cluster SSH keypair in
   `.vagrant/cluster_key` and writes `ansible/inventory.ini`.
2. Every node authorizes the cluster public key.
3. `controlplane` gets the private key, installs Ansible from Ubuntu's archive,
   receives a copy of `ansible/` and runs the playbook against all three nodes.

On the Multipass path the worker IPs aren't known in advance, so the inventory
is rewritten from `multipass info` right before `controlplane` is brought up,
provisioned or reloaded.
