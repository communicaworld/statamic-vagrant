# -*- mode: ruby -*-
# vi: set ft=ruby :

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.

Vagrant.configure("2") do |config|

  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  config.vm.box = "ubuntu/bionic64"

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine and only allow access
  # via 127.0.0.1 to disable public access
  
  config.vm.network "forwarded_port", guest: 80, host: 8001, host_ip: "127.0.0.1"

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.

  # config.vm.synced_folder "../data", "/vagrant_data"

  # Enable provisioning with a shell script. Additional provisioners such as
  # Ansible, Chef, Docker, Puppet and Salt are also available. Please see the
  # documentation for more information about their specific syntax and use.

  # Add files for us to use in the shell script.

  config.vm.provision "file", source: "config/vagrant/.env", destination: ".env"
  config.vm.provision "file", source: "config/statamic/statamic.exp", destination: "statamic.exp"
  config.vm.provision "file", source: "config/apache/apache2.conf", destination: "apache2.conf"
  config.vm.provision "file", source: "config/apache/000-default.conf", destination: "000-default.conf"

  config.vm.provision "shell", inline: <<-SHELL

    #Add environment variables
    
    export $(egrep -v '^#' .env | xargs)
    rm .env

    # Suppress interactive frontend.

    export DEBIAN_FRONTEND=noninteractive

    # Bringing some friends to the party.

    apt-get update
    apt-get -y install \
      apache2 \
      curl \
      git \
      unzip \
      openssl \
      imagemagick \
      expect \
      php \
      php-cli \
      php-common \
      php-bcmath \
      php-json \
      php-mbstring \
      php-mysql \
      php-xml \
      php-gd 

    # Add our Apache configurations.

    mv -f apache2.conf /etc/apache2/apache2.conf
    mv -f 000-default.conf /etc/apache2/sites-available/000-default.conf

    # Remove the DocumentRoot only if Laravel is not detected. We'll be putting it back together later.

    if [ ! -f /var/www/artisan ]; then
      rm -rf /var/www/*
    fi

    # Install Composer

    curl -sS https://getcomposer.org/installer -o composer-setup.php
    php composer-setup.php --install-dir=/usr/local/bin --filename=composer
    rm composer-setup.php
    export COMPOSER_ALLOW_SUPERUSER=1
    composer config -g github-oauth.github.com $GITHUB_AUTH_TOKEN

    # Make a swap file to avoid Composer failing.
    # https://getcomposer.org/doc/articles/troubleshooting.md#proc-open-fork-failed-errors

    if [ ! -f "/var/swap.1" ]; then

      echo "Swap does not exist. Creating swap."
      /bin/dd if=/dev/zero of=/var/swap.1 bs=1M count=1024
      chmod 0600 /var/swap.1
      /sbin/mkswap /var/swap.1
      /sbin/swapon /var/swap.1

    else

      echo "Swap exists. Good to go!"

    fi

    # Statamic is a go.

    if [ ! -d "/var/www/html" ]; then

      # We previously removed the DocumentRoot above. This installs Statamic and recreates the DocumentRoot.

      echo "DocumentRoot does not exist. Begin Statamic installation."
      composer create-project statamic/statamic /var/www/statamic --prefer-dist --stability=dev

      # Move files above the webroot, rename `public` to `html` to reflect DocumentRoot in Apache virtual host.

      mv /var/www/statamic/* /var/www
      mv /var/www/statamic/.env /var/www
      rm -r /var/www/statamic

    else

      # If the DocumentRoot exists, we know we've already installed Statamic.

      echo "DocumentRoot exists. Statamic is already installed."

    fi

    expect statamic.exp
    sleep 6

    # Hand everything back to Apache and enable mod_rewrite.
    a2enmod rewrite
    systemctl restart apache2
    chown -R www-data:www-data /var/www/*

    echo "Ready to go!"

  SHELL

  config.vm.synced_folder ".www", "/var/www", 
    create: true,
    owner: "www-data", 
    group: "www-data"
end
