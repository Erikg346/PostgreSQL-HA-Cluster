Vagrant.configure("2") do |config|
  nodes = {
    "pg01" => "192.168.56.11",
    "pg02" => "192.168.56.12",
    "pg03" => "192.168.56.13"
  }
  nodes.each do |name, ip|
    config.vm.define name do |node|
      node.vm.box = "bento/ubuntu-24.04"
      node.vm.hostname = name
      node.vm.network "private_network", ip: ip
      node.vm.provider "vmware_desktop" do |v|
        v.vmx["memsize"] = "4096"
        v.vmx["numvcpus"] = "2"
        v.gui = false
      end
    end
  end
end
