# Creating virtual machines for testing

If you don't have a spare machine available for testing, you can create it locally, using [VirtualBox](https://www.virtualbox.org/) and [Vagrant](https://www.vagrantup.com/). Make sure you have both tools installed by following respective guides:

- [Installing VirtualBox](https://www.virtualbox.org/manual/ch02.html)
- [Installing Vagrant](https://www.vagrantup.com/docs/installation/)

## Single node

If you need just one machine, create file named `Vagrantfile` with the following content:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box       = "flatcar-stable"
  config.vm.box_url   = "https://stable.release.flatcar-linux.net/amd64-usr/current/flatcar_production_vagrant.box"
  config.ssh.username = 'core'
  config.vm.provider :virtualbox do |v|
    v.memory = 1024
  end
end
```

Then, run the following commands to create and connect to the machine:

```sh
vagrant up && vagrant ssh
```

## Multiple nodes

If you need more than one machine, create file named `Vagrantfile` with the following content:

```ruby
Vagrant.configure("2") do |config|
  config.vm.box       = "flatcar-stable"
  config.vm.box_url   = "https://stable.release.flatcar-linux.net/amd64-usr/current/flatcar_production_vagrant.box"
  config.ssh.username = 'core'
  config.vm.provider :virtualbox do |v|
    v.memory = 1024
  end

  config.vm.define "member1" do |config|
    config.vm.hostname = "member1"
    config.vm.network "private_network", ip: "192.168.52.10"
  end

  config.vm.define "member2" do |config|
    config.vm.hostname = "member2"
    config.vm.network "private_network", ip: "192.168.52.11"
  end
end
```

Then, run the following commands to create the machines:

```sh
vagrant up
```
