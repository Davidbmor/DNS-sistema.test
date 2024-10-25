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


Configura los archivos de zona directa 

sudo cp /etc/bind/db.local /etc/bind/db.sistema.test

sudo nano /etc/bind/db.sistema.test



```plaintext
$TTL    604800
@       IN      SOA     ns1.sistema.test. root.sistema.test. (
                         2         ; Serial
                    604800         ; Refresh
                     86400         ; Retry
                   2419200         ; Expire
                    7200 )       ; Negative Cache TTL
;
@       IN      NS      ns1.sistema.test.
@       IN      NS      ns2.sistema.test.

ns1     IN      A       192.168.57.103
ns2     IN      A       192.168.57.102



```

Configura los archivos de zona inversa  

sudo cp /etc/bind/db.127 /etc/bind/db.192

sudo nano /etc/bind/db.192


```plaintext
$TTL    604800
@       IN      SOA     ns1.sistema.test. root.sistema.test. (
                         2         ; Serial
                    604800         ; Refresh
                     86400         ; Retry
                   2419200         ; Expire
                    7200 )       ; Negative Cache TTL
;
@       IN      NS      ns1.sistema.test.
@       IN      NS      ns2.sistema.test.

103     IN      PTR     ns1.sistema.test.
102     IN      PTR     ns2.sistema.test.





```


Reiniciar BIND9

sudo systemctl restart bind9


### 3. Configuración del Servidor Esclavo (Venus)

Accede a la máquina Venus:

vagrant ssh venus

sudo nano /etc/bind/named.conf.local

```plaintext
zone "sistema.test" {
    type slave;
    file "/var/cache/bind/db.sistema.test";
    masters { 192.168.57.103; };
};

zone "57.168.192.in-addr.arpa" {
    type slave;
    file "/var/cache/bind/db.192";
    masters { 192.168.57.103; };
};


```

Reiniciar BIND9

sudo systemctl restart bind9

### 4. Verificacion 

Resolución de nombres y direcciones IP

Registros A:

Comprueba los registros A en Tierra y Venus:

dig @192.168.57.103 ns1.sistema.test  # Servidor maestro



```plaintext

; <<>> DiG 9.16.50-Debian <<>> @192.168.57.103 ns1.sistema.test
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 48107
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 4d90b55a47cd82a101000000671bf038b9eafb1449cb3f0b (good)
;; QUESTION SECTION:
;ns1.sistema.test.              IN      A

;; ANSWER SECTION:
ns1.sistema.test.       604800  IN      A       192.168.57.103

;; Query time: 4 msec
;; SERVER: 192.168.57.103#53(192.168.57.103)
;; WHEN: Fri Oct 25 19:23:36 UTC 2024
;; MSG SIZE  rcvd: 89

```


dig @192.168.57.102 ns2.sistema.test      # Servidor esclavo


```plaintext

; <<>> DiG 9.16.50-Debian <<>> @192.168.57.102 ns2.sistema.test
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 31636
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 862d42615fe9338d01000000671bf0ae153eecb7a5ae34e0 (good)
;; QUESTION SECTION:
;ns2.sistema.test.              IN      A

;; ANSWER SECTION:
ns2.sistema.test.       604800  IN      A       192.168.57.102

;; Query time: 8 msec
;; SERVER: 192.168.57.102#53(192.168.57.102)
;; WHEN: Fri Oct 25 19:25:34 UTC 2024
;; MSG SIZE  rcvd: 89

```

Resolución inversa:

dig @192.168.57.103 -x 192.168.57.103


```plaintext

; <<>> DiG 9.16.50-Debian <<>> @192.168.57.103 -x 192.168.57.103
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 29147
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 5dbc1fcf9ccf80d601000000671bf1a2a51f47841ab44ebf (good)
;; QUESTION SECTION:
;103.57.168.192.in-addr.arpa.   IN      PTR

;; ANSWER SECTION:
103.57.168.192.in-addr.arpa. 604800 IN  PTR     ns1.sistema.test.

;; Query time: 4 msec
;; SERVER: 192.168.57.103#53(192.168.57.103)
;; WHEN: Fri Oct 25 19:29:38 UTC 2024
;; MSG SIZE  rcvd: 114

```

dig @192.168.57.103 -x 192.168.57.102


```plaintext

; <<>> DiG 9.16.50-Debian <<>> @192.168.57.103 -x 192.168.57.102
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 39691
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: bd441c503c3e7cb701000000671bf20519473019d15d210d (good)
;; QUESTION SECTION:
;102.57.168.192.in-addr.arpa.   IN      PTR

;; ANSWER SECTION:
102.57.168.192.in-addr.arpa. 604800 IN  PTR     ns2.sistema.test.

;; Query time: 4 msec
;; SERVER: 192.168.57.103#53(192.168.57.103)
;; WHEN: Fri Oct 25 19:31:17 UTC 2024
;; MSG SIZE  rcvd: 114

```

Alias:

dig @192.168.57.103 ns1.sistema.test



```plaintext

; <<>> DiG 9.16.50-Debian <<>> @192.168.57.103 ns1.sistema.test
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 62678
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: 3a50b55e6cb91d4901000000671bf2d174b32afd229b23d5 (good)
;; QUESTION SECTION:
;ns1.sistema.test.              IN      A

;; ANSWER SECTION:
ns1.sistema.test.       604800  IN      A       192.168.57.103

;; Query time: 0 msec
;; SERVER: 192.168.57.103#53(192.168.57.103)
;; WHEN: Fri Oct 25 19:34:41 UTC 2024
;; MSG SIZE  rcvd: 89

```

dig @192.168.57.102 ns2.sistema.test

```plaintext

; <<>> DiG 9.16.50-Debian <<>> @192.168.57.102 ns2.sistema.test
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 48784
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: eb98847b8289992d01000000671bf32b2836080dd5e3b97e (good)
;; QUESTION SECTION:
;ns2.sistema.test.              IN      A

;; ANSWER SECTION:
ns2.sistema.test.       604800  IN      A       192.168.57.102

;; Query time: 4 msec
;; SERVER: 192.168.57.102#53(192.168.57.102)
;; WHEN: Fri Oct 25 19:36:11 UTC 2024
;; MSG SIZE  rcvd: 89

```

