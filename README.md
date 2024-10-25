# Proyecto de DNS Maestro-Esclavo con Vagrant

Este proyecto configura un sistema DNS con servidores maestro y esclavo usando **Vagrant** y **BIND9** en máquinas virtuales Debian. La configuración incluye 4 máquinas (2 reales y 2 imaginarias) en una red privada.


## Requisitos previos

- **Vagrant** y **VirtualBox** instalados.


## Infraestructura del Proyecto

Se crean 2 máquinas en la red `192.168.57.0/24`:
| Máquina                | FQDN                | IP              | Descripción                     |
|------------------------|---------------------|-----------------|---------------------------------|
| Venus (esclavo)         | venus.sistema.test  | 192.168.57.102  | Servidor DNS esclavo (Debian)   |
| Tierra (maestro)        | tierra.sistema.test | 192.168.57.103  | Servidor DNS maestro (Debian)   |

---


### 1. Creación del `Vagrantfile`

En el directorio del proyecto, crea el archivo `Vagrantfile` y define las 4 máquinas:

```ruby
Vagrant.configure("2") do |config|

  # Máquina Venus (esclavo)
  config.vm.define "venus" do |venus|
    venus.vm.box = "debian/bullseye64"
    venus.vm.network "private_network", ip: "192.168.57.102"
    venus.vm.hostname = "venus.sistema.test"
    venus.vm.provision "shell", inline: <<-SHELL
      apt-get update
      apt-get install -y bind9 bind9utils bind9-doc
    SHELL
  end

 # Máquina Tierra (maestro)
  config.vm.define "tierra" do |tierra|
    tierra.vm.box = "debian/bullseye64"
    tierra.vm.network "private_network", ip: "192.168.57.103"
    tierra.vm.hostname = "tierra.sistema.test"
    tierra.vm.provision "shell", inline: <<-SHELL
      apt-get update
      apt-get install -y bind9 bind9utils bind9-doc
    SHELL
  end

end