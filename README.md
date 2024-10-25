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

En el directorio del proyecto, crea el archivo `Vagrantfile` y define las 2 máquinas:

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

```

### 2. Configuración del Servidor Maestro (Tierra)

Accede a la máquina Tierra:

vagrant ssh tierra


Configura el archivo de opciones de BIND9

```plaintext
options {
    directory "/var/cache/bind";
    forwarders {
        208.67.222.222;  # OpenDNS
    };
    dnssec-validation yes;
    listen-on { any; };
    allow-query { localhost; 192.168.57.0/24; };
    recursion yes;

};

```

Configura las zonas en named.conf.local

sudo nano /etc/bind/named.conf.local


```plaintext
zone "sistema.test" {
    type master;
    file "/etc/bind/db.sistema.test";
    allow-transfer { 192.168.57.102; };  # Transferencia al esclavo
};

zone "57.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192";
    allow-transfer { 192.168.57.102; };
};


```