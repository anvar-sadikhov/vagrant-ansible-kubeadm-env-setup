# -*- mode: ruby -*-
# vi: set ft=ruby :
#
# 3x Ubuntu 24.04 LTS nodes (node01, node02, controlplane), provisioned into
# a kubeadm + Cilium Kubernetes cluster by Ansible running INSIDE the
# controlplane VM (ansible_local). The host needs no Ansible at all.
#
# Provider, picked from the host (override with K8S_PROVIDER=<name>):
#   - Apple Silicon (macOS + arm64)  -> multipass  (native ARM64 VMs)
#   - Windows                        -> hyperv     (x86_64 VMs)
#   - Linux / Intel macOS            -> virtualbox (x86_64 VMs)
#
# Why Hyper-V and not VirtualBox on Windows: whenever the Windows hypervisor
# is running (WSL2, Docker Desktop, Virtualization-Based Security), VirtualBox
# has to run on top of it, and there it loses guest timer interrupts on
# multi-CPU VMs — boots hang at "Loading essential drivers" and running VMs
# freeze for minutes (RCU stalls). Hyper-V *is* that hypervisor, so it
# doesn't have the problem. With the hypervisor fully disabled, VirtualBox
# works fine on Windows too: set K8S_PROVIDER=virtualbox.
#
# How provisioning works (all providers):
#   1. Machines boot in definition order: node01, node02, then controlplane.
#      None of these providers boots in parallel, so the workers are always
#      up by the time controlplane is provisioned.
#   2. Every node gets a cluster-internal SSH public key (generated once on
#      the host into .vagrant/cluster_key) so controlplane can reach them.
#   3. controlplane installs Ansible from Ubuntu's archive, receives a copy of
#      ansible/ (playbook + generated inventory.ini), and runs the playbook
#      against all three nodes.
#
# Networking:
#   - VirtualBox and Hyper-V: bridged onto your real LAN at static 192.168.0.x
#     addresses (see NODE_IPS below — adjust if they collide with anything).
#       VirtualBox bridges onto the host's default-route adapter (override
#       with K8S_BRIDGE, a name from `VBoxManage list bridgedifs`).
#       Hyper-V uses an External virtual switch: an existing one if there is
#       one (or K8S_HYPERV_SWITCH), otherwise it creates "k8s-external" on
#       the host's default-route adapter on first `vagrant up` — expect the
#       host's network to drop for a few seconds while that happens. Hyper-V
#       can't assign static IPs itself, so each node keeps its DHCP address
#       and gets its static address added on top via netplan.
#   - Multipass: NOT bridged. Repeated attempts at real LAN bridging
#     (`multipass launch --network name=bridged,...`) reliably wedged the
#     Multipass daemon on this machine (had to be recovered by killing
#     orphaned qemu processes, restarting the daemon, and eventually a full
#     reboot — and it wedged again on the very next VM afterward). Given
#     that repeated, reproducible instability, this path uses Multipass's
#     default NAT network instead: reliable, but nodes get Multipass's own
#     private IPs, not real 192.168.0.x addresses. The worker IPs are looked
#     up right before controlplane is brought up / provisioned.
#
# Requires, for whichever path applies to you:
#   Hyper-V path:     Windows Pro/Enterprise with the Hyper-V feature enabled,
#                     and Vagrant run as Administrator or as a member of the
#                     "Hyper-V Administrators" group.
#   VirtualBox path:  VirtualBox 7.x (https://www.virtualbox.org/).
#   Multipass path:   Multipass (brew install --cask multipass), the
#                     vagrant-multipass plugin (built from source since it
#                     isn't published on RubyGems), and an SSH keypair at
#                     ~/.ssh/id_rsa[.pub] that Vagrant logs in with:
#                        git clone https://github.com/Fred78290/vagrant-multipass.git
#                        cd vagrant-multipass && gem build vagrant-multipass.gemspec
#                        vagrant plugin install ./vagrant-multipass-*.gem
#                        ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa

require 'json'
require 'open3'
require 'openssl'
require 'rbconfig'

PROJECT_DIR = __dir__

# ---------------------------------------------------------------------------
# Host detection
# ---------------------------------------------------------------------------
def host_os
  RbConfig::CONFIG['host_os']
end

def host_cpu
  RbConfig::CONFIG['host_cpu']
end

def apple_silicon?
  host_os =~ /darwin/ && host_cpu =~ /arm64|aarch64/
end

def windows_host?
  host_os =~ /mswin|mingw|cygwin/
end

def command_ok?(*cmd)
  system(*cmd, out: File::NULL, err: File::NULL)
rescue StandardError
  false
end

# Run a command without a shell and return its stdout, or "" on any failure.
def capture(*cmd)
  out, status = Open3.capture2(*cmd, err: File::NULL)
  status.success? ? out : ""
rescue StandardError
  ""
end

def powershell(script)
  capture("powershell", "-NoProfile", "-NonInteractive", "-Command", script).strip
end

SUPPORTED_PROVIDERS = ["multipass", "hyperv", "virtualbox"]

PROVIDER = (ENV["K8S_PROVIDER"] ||
            (apple_silicon? ? "multipass" : windows_host? ? "hyperv" : "virtualbox")).downcase

unless SUPPORTED_PROVIDERS.include?(PROVIDER)
  raise Vagrant::Errors::VagrantError.new,
    "K8S_PROVIDER=#{PROVIDER} isn't supported. Use one of: #{SUPPORTED_PROVIDERS.join(', ')}\n"
end

# Pin Vagrant to our choice. Otherwise, if the chosen provider isn't usable
# (e.g. Hyper-V without admin rights), Vagrant silently falls back to any
# other installed provider instead of reporting why.
ENV["VAGRANT_DEFAULT_PROVIDER"] = PROVIDER

USE_MULTIPASS = PROVIDER == "multipass"

# ---------------------------------------------------------------------------
# Configuration
# ---------------------------------------------------------------------------
# Definition order == boot order. controlplane MUST be last: it's the node
# that runs Ansible, so the workers have to be up before it provisions.
WORKER_NAMES = ["node01", "node02"]
MASTER_NAME  = "controlplane"
NODE_NAMES   = WORKER_NAMES + [MASTER_NAME]

# Static IPs on your real LAN, used by the VirtualBox and Hyper-V paths.
# Must be free addresses on your 192.168.0.0/24 network (outside your
# router's DHCP range, or reserved there) or you'll get a collision.
NODE_IPS = {
  "controlplane" => "192.168.0.101",
  "node01"       => "192.168.0.102",
  "node02"       => "192.168.0.103",
}
NODE_IP_PREFIX = 24

NODE_SPECS = {
  "controlplane" => { mem: 2048, cpus: 2 },
  "node01"       => { mem: 2048, cpus: 2 },
  "node02"       => { mem: 2048, cpus: 2 },
}

# All Ubuntu 24.04 LTS. Bento publishes no Hyper-V build, hence the
# different box there.
VBOX_BOX        = "bento/ubuntu-24.04"
HYPERV_BOX      = "gusztavvargadr/ubuntu-server-2404-lts"
MULTIPASS_IMAGE = "24.04"

SSH_USER = USE_MULTIPASS ? "ubuntu" : "vagrant"

# Host key Vagrant itself logs in with on the Multipass path.
SSH_PUBLIC_KEY_PATH  = File.join(Dir.home, ".ssh", "id_rsa.pub")
SSH_PRIVATE_KEY_PATH = File.join(Dir.home, ".ssh", "id_rsa")

# Cluster-internal key: controlplane's Ansible uses it to reach the workers.
CLUSTER_KEY_PATH       = File.join(PROJECT_DIR, ".vagrant", "cluster_key")
CLUSTER_KEY_GUEST_PATH = "/home/#{SSH_USER}/.ssh/id_cluster"

INVENTORY_PATH     = File.join(PROJECT_DIR, "ansible", "inventory.ini")
ANSIBLE_GUEST_PATH = "/home/#{SSH_USER}/k8s-ansible"

# ---------------------------------------------------------------------------
# Cluster SSH keypair. Generated with Ruby's OpenSSL rather than ssh-keygen
# so it works identically on Windows and macOS without extra host tools.
# ---------------------------------------------------------------------------
def openssh_public_key(rsa)
  field = ->(bytes) { [bytes.bytesize].pack("N") + bytes }
  mpint = ->(bn) do
    bytes = bn.to_s(2)
    bytes = "\x00".b + bytes if bytes.getbyte(0) & 0x80 != 0
    field.call(bytes)
  end
  blob = field.call("ssh-rsa".b) + mpint.call(rsa.e) + mpint.call(rsa.n)
  "ssh-rsa #{[blob].pack('m0')} k8s-cluster"
end

def cluster_keypair
  pub_path = "#{CLUSTER_KEY_PATH}.pub"
  unless File.file?(CLUSTER_KEY_PATH) && File.file?(pub_path)
    rsa = OpenSSL::PKey::RSA.new(3072)
    Dir.mkdir(File.dirname(CLUSTER_KEY_PATH)) unless Dir.exist?(File.dirname(CLUSTER_KEY_PATH))
    File.write(CLUSTER_KEY_PATH, rsa.to_pem)
    File.write(pub_path, openssh_public_key(rsa) + "\n")
  end
  [File.read(CLUSTER_KEY_PATH), File.read(pub_path).strip]
end

# ---------------------------------------------------------------------------
# VirtualBox bridge detection (best-effort; falls back to letting Vagrant
# prompt interactively if we can't figure it out). VirtualBox names
# bridgeable adapters differently per OS:
#   Windows: the adapter description, e.g. "Intel(R) Wi-Fi 6 AX201 160MHz"
#   macOS:   "en0: Wi-Fi"
#   Linux:   "eth0" / "enp3s0"
# so we find the host's default-route interface, then match it against
# `VBoxManage list bridgedifs` to get the exact name VirtualBox expects.
# ---------------------------------------------------------------------------
def vboxmanage_path
  if windows_host?
    dir = ENV["VBOX_MSI_INSTALL_PATH"] || ENV["VBOX_INSTALL_PATH"] ||
          "C:\\Program Files\\Oracle\\VirtualBox"
    File.join(dir, "VBoxManage.exe")
  else
    "VBoxManage"
  end
end

# Windows: the default-route adapter's property (InterfaceDescription for
# VirtualBox, Name for New-VMSwitch).
def windows_default_route_adapter(property)
  powershell(
    "$r = Get-NetRoute -DestinationPrefix '0.0.0.0/0' -ErrorAction SilentlyContinue | " \
    "Sort-Object RouteMetric | Select-Object -First 1; " \
    "if ($r) { (Get-NetAdapter -InterfaceIndex $r.ifIndex).#{property} }"
  )
end

def default_route_interface
  if windows_host?
    windows_default_route_adapter("InterfaceDescription")
  elsif host_os =~ /darwin/
    capture("route", "get", "default")[/interface: (\S+)/, 1].to_s
  else
    capture("ip", "route", "show", "default")[/dev (\S+)/, 1].to_s
  end
end

def detect_bridge_interface
  return ENV["K8S_BRIDGE"] if ENV["K8S_BRIDGE"] && !ENV["K8S_BRIDGE"].empty?

  iface = default_route_interface
  return nil if iface.empty?

  bridges = capture(vboxmanage_path, "list", "bridgedifs").scan(/^Name:\s+(.+?)\s*$/).flatten
  bridges.find { |b| b == iface } ||
    bridges.find { |b| b.start_with?("#{iface}:") } ||
    (bridges.empty? ? iface : nil)
end

# ---------------------------------------------------------------------------
# Hyper-V External switch. Use K8S_HYPERV_SWITCH, else any existing External
# switch (an adapter can only back one), else create "k8s-external" on the
# default-route adapter — done from a trigger so it only happens on `up`.
# ---------------------------------------------------------------------------
HYPERV_NEW_SWITCH = "k8s-external"

def hyperv_switch_name
  return ENV["K8S_HYPERV_SWITCH"] if ENV["K8S_HYPERV_SWITCH"] && !ENV["K8S_HYPERV_SWITCH"].empty?

  existing = powershell(
    "Get-VMSwitch -SwitchType External -ErrorAction SilentlyContinue | " \
    "Select-Object -First 1 -ExpandProperty Name"
  )
  existing.empty? ? HYPERV_NEW_SWITCH : existing
end

def ensure_hyperv_switch(name)
  exists = powershell("if (Get-VMSwitch -Name '#{name}' -ErrorAction SilentlyContinue) { 'yes' }")
  return if exists == "yes"

  adapter = windows_default_route_adapter("Name")
  if adapter.empty? || name != HYPERV_NEW_SWITCH
    raise Vagrant::Errors::VagrantError.new,
      "Hyper-V switch '#{name}' doesn't exist and couldn't be created automatically.\n" \
      "Create an External switch yourself (Hyper-V Manager > Virtual Switch Manager),\n" \
      "or from an Administrator PowerShell:\n" \
      "  New-VMSwitch -Name #{HYPERV_NEW_SWITCH} -NetAdapterName '<your LAN adapter>' -AllowManagementOS $true\n"
  end

  puts "==> Creating Hyper-V External switch '#{name}' on '#{adapter}'" \
       " (the host's network will drop for a few seconds)..."
  powershell(
    "New-VMSwitch -Name '#{name}' -NetAdapterName '#{adapter}' -AllowManagementOS $true " \
    "-ErrorAction Stop | Out-Null; 'ok'"
  ) == "ok" or raise Vagrant::Errors::VagrantError.new,
    "Failed to create Hyper-V switch '#{name}' on '#{adapter}'. Run Vagrant as Administrator\n" \
    "(or as a Hyper-V Administrators member), or create the switch yourself:\n" \
    "  New-VMSwitch -Name #{name} -NetAdapterName '#{adapter}' -AllowManagementOS $true\n"

  # Give the host time to get its address back on the new vEthernet adapter.
  sleep 15
end

# ---------------------------------------------------------------------------
# Multipass IP discovery (these get dynamic NAT IPs). Retry for a bit —
# Multipass assigns the address via DHCP shortly after boot and it isn't
# always available the instant the VM reports "Running".
# ---------------------------------------------------------------------------
def multipass_ip_for(name, retries: 30, delay: 2)
  retries.times do |attempt|
    raw = capture("multipass", "info", name, "--format", "json")
    unless raw.empty?
      ip = JSON.parse(raw).dig("info", name, "ipv4")&.first
      return ip if ip && !ip.empty?
    end
    sleep delay unless attempt == retries - 1
  end
  nil
rescue StandardError
  nil
end

# ---------------------------------------------------------------------------
# Inventory, as seen from inside controlplane. controlplane talks to itself
# over a local connection; workers are reached over SSH with the cluster key.
# node_ip is the address each node advertises to Kubernetes. On VirtualBox
# and Hyper-V it's the static LAN IP (VirtualBox's default route goes out
# the NAT adapter, and Hyper-V nodes also hold a DHCP address); on Multipass
# there's one address, so the playbook falls back to the default IPv4 fact
# and node_ip is left unset.
# ---------------------------------------------------------------------------
def write_inventory(worker_ips)
  node_ip = ->(n) { USE_MULTIPASS ? "" : " node_ip=#{NODE_IPS[n]}" }

  lines = []
  lines << "; AUTO-GENERATED by the Vagrantfile — do not hand-edit."
  lines << ""
  lines << "[master]"
  lines << "#{MASTER_NAME} ansible_connection=local#{node_ip.call(MASTER_NAME)}"
  lines << ""
  lines << "[workers]"
  WORKER_NAMES.each do |n|
    lines << "#{n} ansible_host=#{worker_ips[n]}#{node_ip.call(n)}"
  end
  lines << ""
  lines << "[all:children]"
  lines << "master"
  lines << "workers"
  lines << ""
  lines << "[all:vars]"
  lines << "ansible_user=#{SSH_USER}"
  lines << "ansible_ssh_private_key_file=#{CLUSTER_KEY_GUEST_PATH}"
  lines << "ansible_ssh_common_args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'"
  lines << "ansible_python_interpreter=/usr/bin/python3"

  File.write(INVENTORY_PATH, lines.join("\n") + "\n")
end

def write_multipass_inventory
  ips = WORKER_NAMES.to_h { |n| [n, multipass_ip_for(n)] }
  missing = ips.select { |_, ip| ip.nil? }.keys
  unless missing.empty?
    raise Vagrant::Errors::VagrantError.new,
      "Can't find the IP of #{missing.join(', ')} — bring the workers up first\n" \
      "(plain `vagrant up` does this, since they're defined before #{MASTER_NAME}).\n"
  end
  write_inventory(ips)
end

# VirtualBox and Hyper-V use fully static IPs, so the inventory can be
# written up front. The Multipass path writes it from a trigger (see below).
write_inventory(NODE_IPS) unless USE_MULTIPASS

# ---------------------------------------------------------------------------
# Guest-side scripts
# ---------------------------------------------------------------------------
def authorize_key_script(pub_key)
  <<~SH
    set -eu
    install -d -m 700 -o #{SSH_USER} -g #{SSH_USER} /home/#{SSH_USER}/.ssh
    AUTH=/home/#{SSH_USER}/.ssh/authorized_keys
    touch "$AUTH"
    grep -qxF '#{pub_key}' "$AUTH" || echo '#{pub_key}' >> "$AUTH"
    chown #{SSH_USER}:#{SSH_USER} "$AUTH"
    chmod 600 "$AUTH"
  SH
end

# Hyper-V only: add the node's static LAN IP next to its DHCP address.
# `netplan set` merges it into whichever netplan definition already owns the
# interface (so DHCP keeps working and it survives reboots); `ip addr
# replace` applies it now without bouncing the link Vagrant is using.
def static_ip_script(ip)
  <<~SH
    set -eu
    IFACE=$(ip -o -4 route show default | awk '{print $5; exit}')
    ID=$(netplan get ethernets | python3 -c '
    import sys, yaml
    iface, ids = sys.argv[1], list((yaml.safe_load(sys.stdin) or {}).keys())
    print(iface if iface in ids else ids[0] if len(ids) == 1 else "")
    ' "$IFACE")
    if [ -z "$ID" ]; then
      echo "Can't tell which netplan definition owns $IFACE" >&2
      exit 1
    fi
    netplan set "ethernets.$ID.addresses=[#{ip}/#{NODE_IP_PREFIX}]"
    ip addr replace #{ip}/#{NODE_IP_PREFIX} dev "$IFACE"
  SH
end

# Private key is passed inline rather than via a shared folder: VirtualBox
# shared folders on Windows show every file as 0777, which ssh rejects.
def controller_setup_script(private_key)
  <<~SH
    set -eu
    cat > #{CLUSTER_KEY_GUEST_PATH} <<'KEY'
    #{private_key.strip}
    KEY
    chown #{SSH_USER}:#{SSH_USER} #{CLUSTER_KEY_GUEST_PATH}
    chmod 600 #{CLUSTER_KEY_GUEST_PATH}

    # Ubuntu 24.04's `ansible` package bundles community.general and
    # ansible.posix, which the playbook needs.
    if ! command -v ansible-playbook >/dev/null 2>&1; then
      export DEBIAN_FRONTEND=noninteractive
      apt-get update -q
      apt-get install -y -q ansible
    fi

    # Cleared so the upload that follows lands here, not in a subdirectory.
    rm -rf #{ANSIBLE_GUEST_PATH}
  SH
end

Vagrant.configure("2") do |config|

  case PROVIDER
  when "multipass"
    unless command_ok?("multipass", "version")
      raise Vagrant::Errors::VagrantError.new,
        "Multipass isn't installed (or not on PATH). Install it first:\n" \
        "  brew install --cask multipass\n"
    end
    unless Vagrant.has_plugin?("vagrant-multipass")
      raise Vagrant::Errors::VagrantError.new,
        "The vagrant-multipass plugin isn't installed. It isn't published on\n" \
        "RubyGems, so build it from source:\n" \
        "  git clone https://github.com/Fred78290/vagrant-multipass.git\n" \
        "  cd vagrant-multipass && gem build vagrant-multipass.gemspec\n" \
        "  vagrant plugin install ./vagrant-multipass-*.gem\n"
    end
    unless File.file?(SSH_PUBLIC_KEY_PATH)
      raise Vagrant::Errors::VagrantError.new,
        "No SSH keypair found at #{SSH_PRIVATE_KEY_PATH}. Generate one first:\n" \
        "  ssh-keygen -t rsa -b 4096 -f #{SSH_PRIVATE_KEY_PATH}\n"
    end
  when "virtualbox"
    config.vbguest.auto_update = false if Vagrant.has_plugin?("vagrant-vbguest")
  end

  cluster_private_key, cluster_public_key = cluster_keypair
  bridge_iface  = detect_bridge_interface if PROVIDER == "virtualbox"
  hyperv_switch = hyperv_switch_name      if PROVIDER == "hyperv"

  if PROVIDER == "hyperv"
    config.trigger.before :up do |trigger|
      trigger.ruby { |_env, _machine| ensure_hyperv_switch(hyperv_switch) }
    end
  end

  # The playbook is uploaded to controlplane instead (see below).
  config.vm.synced_folder ".", "/vagrant", disabled: true

  NODE_NAMES.each do |name|
    spec = NODE_SPECS[name]
    is_master = name == MASTER_NAME

    config.vm.define name, primary: is_master do |node|
      node.vm.hostname = name

      case PROVIDER
      when "multipass"
        node.vm.box = "multipass"
        node.ssh.username = SSH_USER
        node.ssh.private_key_path = [SSH_PRIVATE_KEY_PATH]

        node.vm.provider "multipass" do |mp|
          mp.image_name = MULTIPASS_IMAGE
          mp.cpu_count  = spec[:cpus]
          mp.memory_mb  = spec[:mem]
          mp.hd_size    = "10G"
        end

      when "hyperv"
        node.vm.box = HYPERV_BOX
        node.vm.network "public_network", bridge: hyperv_switch

        node.vm.provider "hyperv" do |hv|
          hv.vmname  = "k8s-#{name}"
          hv.memory  = spec[:mem]  # no maxmemory -> static memory, as kubelet expects
          hv.cpus    = spec[:cpus]
          hv.linked_clone = true
          hv.enable_checkpoints           = false
          hv.enable_automatic_checkpoints = false
        end

        node.vm.provision "static-ip", type: "shell",
          inline: static_ip_script(NODE_IPS[name])

      when "virtualbox"
        node.vm.box = VBOX_BOX
        ip = NODE_IPS[name]

        if bridge_iface
          node.vm.network "public_network", ip: ip, bridge: bridge_iface
        else
          # Couldn't auto-detect a host interface — Vagrant will prompt
          # you to pick one interactively on `vagrant up`.
          node.vm.network "public_network", ip: ip
        end

        node.vm.provider "virtualbox" do |vb|
          vb.name   = "k8s-#{name}"
          vb.memory = spec[:mem]
          vb.cpus   = spec[:cpus]
          vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
        end
      end

      node.vm.provision "cluster-key", type: "shell",
        inline: authorize_key_script(cluster_public_key)

      next unless is_master

      if USE_MULTIPASS
        # Refresh the inventory with the workers' current NAT IPs before
        # controlplane is provisioned. These are command triggers: they fire
        # per machine as each command reaches it, so on a full `vagrant up`
        # this runs after node01/node02 are already up.
        node.trigger.before [:up, :provision, :reload] do |trigger|
          trigger.name = "Write Ansible inventory"
          trigger.ruby { |_env, _machine| write_multipass_inventory }
        end
      end

      node.vm.provision "ansible-controller", type: "shell",
        inline: controller_setup_script(cluster_private_key)

      node.vm.provision "ansible-files", type: "file",
        source: "ansible", destination: ANSIBLE_GUEST_PATH

      node.vm.provision "cluster", type: "ansible_local" do |ansible|
        ansible.install            = false  # installed by ansible-controller
        ansible.provisioning_path  = ANSIBLE_GUEST_PATH
        ansible.playbook           = "playbook.yml"
        ansible.inventory_path     = "inventory.ini"
        ansible.limit              = "all"
        ansible.compatibility_mode = "2.0"
      end
    end
  end
end
