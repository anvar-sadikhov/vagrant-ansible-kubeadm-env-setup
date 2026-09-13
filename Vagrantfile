# -*- mode: ruby -*-
# vi: set ft=ruby :
#
# 3x Ubuntu 24.04 LTS nodes (controlplane, node01, node02), provisioned into
# a kubeadm + Cilium Kubernetes cluster via Ansible run from the host.
#
#   - Apple Silicon (macOS + arm64)  -> Multipass provider (native ARM64 VMs)
#   - Windows / Linux / Intel macOS  -> VirtualBox provider (x86_64 VMs)
#
# Networking:
#   - VirtualBox path: bridged onto your real LAN at static 192.168.0.x
#     addresses (see NODE_IPS below — adjust if they collide with anything).
#   - Multipass path: NOT bridged. Repeated attempts at real LAN bridging
#     (`multipass launch --network name=bridged,...`) reliably wedged the
#     Multipass daemon on this machine (had to be recovered by killing
#     orphaned qemu processes, restarting the daemon, and eventually a full
#     reboot — and it wedged again on the very next VM afterward). Given
#     that repeated, reproducible instability, this path uses Multipass's
#     default NAT network instead: reliable, but nodes get Multipass's own
#     private IPs, not real 192.168.0.x addresses. The Vagrantfile
#     discovers those IPs after boot and builds the Ansible inventory then.
#
# Requires, for whichever path applies to you:
#   VirtualBox path:  VirtualBox itself (https://www.virtualbox.org/)
#   Multipass path:   Multipass (brew install --cask multipass) AND the
#                      vagrant-multipass plugin, built from source since it
#                      isn't published on RubyGems:
#                        git clone https://github.com/Fred78290/vagrant-multipass.git
#                        cd vagrant-multipass && gem build vagrant-multipass.gemspec
#                        vagrant plugin install ./vagrant-multipass-*.gem
#   Both paths also need Ansible on the HOST: brew/apt install ansible, or
#   pip install ansible.
#
# Both paths need a keypair at ~/.ssh/id_rsa[.pub] (ssh-keygen -t rsa -b 4096
# -f ~/.ssh/id_rsa) — it's injected into every node for Ansible to connect.

require 'json'
require 'rbconfig'

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

USE_MULTIPASS = apple_silicon?

def command_ok?(*cmd)
  system(*cmd, out: File::NULL, err: File::NULL)
rescue StandardError
  false
end

# ---------------------------------------------------------------------------
# Configuration
# ---------------------------------------------------------------------------
NODE_NAMES = ["controlplane", "node01", "node02"]

# Static IPs on your real LAN, used by the VirtualBox (bridged) path only.
# Must be free addresses on your 192.168.0.0/24 network (outside your
# router's DHCP range, or reserved there) or you'll get a collision.
NODE_IPS = {
  "controlplane" => "192.168.0.101",
  "node01"       => "192.168.0.102",
  "node02"       => "192.168.0.103",
}

NODE_SPECS = {
  "controlplane" => { mem: 2048, cpus: 2, role: "master" },
  "node01"       => { mem: 2048, cpus: 2, role: "worker" },
  "node02"       => { mem: 2048, cpus: 2, role: "worker" },
}

VBOX_BOX        = "ubuntu/noble64"  # Ubuntu 24.04 LTS, x86_64
MULTIPASS_IMAGE = "24.04"           # Ubuntu 24.04 LTS, native arm64

VBOX_SSH_USER       = "vagrant"  # Canonical's Vagrant Cloud boxes use this
MULTIPASS_SSH_USER  = "ubuntu"   # Multipass cloud images use this

SSH_PUBLIC_KEY_PATH  = File.join(Dir.home, ".ssh", "id_rsa.pub")
SSH_PRIVATE_KEY_PATH = File.join(Dir.home, ".ssh", "id_rsa")

# ---------------------------------------------------------------------------
# Bridge interface auto-detection for the VirtualBox path (best-effort;
# falls back to letting Vagrant/VirtualBox prompt interactively if we can't
# figure it out).
# ---------------------------------------------------------------------------
def detect_bridge_interface
  if host_os =~ /darwin/
    `route get default 2>/dev/null`[/interface: (\S+)/, 1]
  elsif windows_host?
    ps = 'Get-NetAdapter | Where-Object {$_.Status -eq "Up"} | ' \
         'Sort-Object -Property InterfaceMetric | Select-Object -First 1 -ExpandProperty Name'
    out = `powershell -NoProfile -Command "#{ps}" 2>nul`.strip
    out.empty? ? nil : out
  else
    `ip route show default 2>/dev/null`[/dev (\S+)/, 1]
  end
rescue StandardError
  nil
end

# ---------------------------------------------------------------------------
# Multipass IP discovery (post-boot, since these get dynamic NAT IPs). Retry
# for a bit — Multipass assigns the address via DHCP shortly after boot and
# it isn't always available the instant the VM reports "Running".
# ---------------------------------------------------------------------------
def multipass_ip_for(name, retries: 30, delay: 2)
  retries.times do |attempt|
    raw = `multipass info #{name} --format json`
    data = JSON.parse(raw)
    ip = data["info"][name]["ipv4"].first
    return ip if ip && !ip.empty?
    sleep delay unless attempt == retries - 1
  end
  nil
rescue StandardError
  nil
end

def node_private_key_path(name)
  if USE_MULTIPASS
    SSH_PRIVATE_KEY_PATH
  else
    # Vagrant generates a unique keypair per VM under .vagrant/machines.
    File.join(File.dirname(__FILE__), ".vagrant", "machines", name, "virtualbox", "private_key")
  end
end

def write_inventory(multipass_ips = nil)
  ssh_user = USE_MULTIPASS ? MULTIPASS_SSH_USER : VBOX_SSH_USER
  ip_for = USE_MULTIPASS ? ->(n) { multipass_ips[n] } : ->(n) { NODE_IPS[n] }

  lines = []
  lines << "; AUTO-GENERATED by the Vagrantfile — do not hand-edit."
  lines << ""
  lines << "[master]"
  lines << "controlplane ansible_host=#{ip_for.call('controlplane')} ansible_ssh_private_key_file=#{node_private_key_path('controlplane')}"
  lines << ""
  lines << "[workers]"
  lines << "node01 ansible_host=#{ip_for.call('node01')} ansible_ssh_private_key_file=#{node_private_key_path('node01')}"
  lines << "node02 ansible_host=#{ip_for.call('node02')} ansible_ssh_private_key_file=#{node_private_key_path('node02')}"
  lines << ""
  lines << "[all:children]"
  lines << "master"
  lines << "workers"
  lines << ""
  lines << "[all:vars]"
  lines << "ansible_user=#{ssh_user}"
  lines << "ansible_ssh_common_args='-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null'"
  lines << "ansible_python_interpreter=/usr/bin/python3"

  File.write(File.join(File.dirname(__FILE__), "ansible", "inventory.ini"), lines.join("\n") + "\n")
end

# Look up all 3 nodes' real IPs, write the inventory, and run Ansible from
# the host. Shared by the global :up and :provision triggers below so that
# both `vagrant up` and `vagrant provision` work on the Multipass path.
#
# These triggers fire once per machine (not once per command), so a single
# `vagrant up`/`vagrant provision` spanning all 3 nodes would otherwise run
# the whole playbook up to 3 times back-to-back. Guard against that with a
# process-wide flag — one Ansible run per Vagrant invocation is enough.
$multipass_ansible_already_ran = false

def run_multipass_ansible
  return if $multipass_ansible_already_ran

  ips = {}
  NODE_NAMES.each { |name| ips[name] = multipass_ip_for(name) }

  unless ips.values.all?
    # e.g. `vagrant up controlplane` on its own — not all nodes are up.
    puts "==> Skipping Ansible run — not all nodes are up yet (#{ips.inspect})"
    return
  end

  $multipass_ansible_already_ran = true
  write_inventory(ips)

  project_dir = File.dirname(__FILE__)
  inventory   = File.join(project_dir, "ansible", "inventory.ini")
  playbook    = File.join(project_dir, "ansible", "playbook.yml")

  puts "==> Running Ansible against all #{NODE_NAMES.length} nodes..."
  unless system("ansible-playbook", "-i", inventory, playbook)
    raise Vagrant::Errors::VagrantError.new,
      "Ansible provisioning failed — see output above."
  end
end

# VirtualBox path uses fully static IPs, so the inventory can be written
# once, up front — no post-boot discovery needed.
write_inventory unless USE_MULTIPASS

Vagrant.configure("2") do |config|

  if USE_MULTIPASS
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
  else
    unless command_ok?("VBoxManage", "--version")
      raise Vagrant::Errors::VagrantError.new,
        "VirtualBox isn't installed (or VBoxManage is broken/not on PATH).\n" \
        "Install it from https://www.virtualbox.org/wiki/Downloads\n"
    end
    if Vagrant.has_plugin?("vagrant-vbguest")
      config.vbguest.auto_update = false
    end
  end

  bridge_iface = detect_bridge_interface unless USE_MULTIPASS

  NODE_NAMES.each do |name|
    spec = NODE_SPECS[name]

    config.vm.define name do |node|
      node.vm.hostname = name

      if USE_MULTIPASS
        node.vm.box = "multipass"
        node.ssh.username = MULTIPASS_SSH_USER
        node.ssh.private_key_path = [SSH_PRIVATE_KEY_PATH]
        node.vm.synced_folder ".", "/vagrant", type: "rsync"

        node.vm.provider "multipass" do |mp|
          mp.image_name = MULTIPASS_IMAGE
          mp.cpu_count  = spec[:cpus]
          mp.memory_mb  = spec[:mem]
          mp.hd_size    = "10G"
        end

      else
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

      # VirtualBox only: provision once, after the LAST node is up, so the
      # whole cluster is targeted in a single Ansible run against the
      # static inventory written above. The Multipass path is provisioned
      # separately below via global triggers, since its IPs aren't known
      # until each node has actually booted.
      if !USE_MULTIPASS && name == NODE_NAMES.last
        node.vm.provision "ansible" do |ansible|
          ansible.playbook = "ansible/playbook.yml"
          ansible.inventory_path = "ansible/inventory.ini"
          ansible.limit = "all"
          ansible.compatibility_mode = "2.0"
        end
      end
    end
  end

  if USE_MULTIPASS
    # Global (config-level) triggers, not per-node ones: the vagrant-multipass
    # provider doesn't run per-node provisioning as a distinct ":provision"
    # action the way VirtualBox does, so a `node.trigger.before :provision`
    # on the last node never reliably fires. Global "after" triggers fire
    # once the `vagrant up` / `vagrant provision` command itself finishes
    # (across whichever machines it targeted), which every provider reaches.
    config.trigger.after :up do |trigger|
      trigger.ruby { |_env, _machine| run_multipass_ansible }
    end

    config.trigger.after :provision do |trigger|
      trigger.ruby { |_env, _machine| run_multipass_ansible }
    end
  end
end
