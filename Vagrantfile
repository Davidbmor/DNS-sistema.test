Vagrant.configure("2") do |config|
  
  # Configuración de máquina Mercurio (Linux gráfico - imaginaria)
  config.vm.define "mercurio" do |mercurio|
    mercurio.vm.box = "ubuntu/bionic64"
    mercurio.vm.hostname = "mercurio.sistema.test"
    mercurio.vm.network "private_network", ip: "192.168.57.101"
    mercurio.vm.provision "shell", inline: <<-SHELL
      echo "Configurando Mercurio (imaginario)"
    SHELL
  end
