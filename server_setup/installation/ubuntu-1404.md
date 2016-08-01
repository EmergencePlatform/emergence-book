# Installation Guide for Ubuntu 14.04

## Overview {#overview}### Screencast {#screencast}
This screencast walks through installing Emergence on a fresh [Digital Ocean](https://www.digitalocean.com/?refcode=889859901aab)virtual machine:<iframe width="640" height="480" src="//www.youtube.com/embed/md7_J_ol5TY?rel=0" frameborder="0" allowfullscreen></iframe>

### Quick install {#quickinstall}
If you are starting from a **fresh** Ubuntu 14.04 installation you can use this script to run all the following setup commandsall at once, and then skip down to **[Create a site](#create-a-site) or [Secure your installation](#secure)**.

```language-bash
user@hostname ~ $ wget http://emr.ge/dist/ubuntu/quickinstall-14.04.sh -O - | sudo sh
```

## Ensure trusty-backports are enabled {#enable-backports}
An updated package from trusty-backports will need to be installed later on in this guide. Enable it now before running `apt-get update` in the next step.

```language-bash
user@hostname ~ $ sudo sed -i '/deb .* trusty-backports/s/^#\s*//' /etc/apt/sources.list
```

## Install service binaries {#service-binaries}
These commands will update your system and then install all the packages required for Emergence:

```language-bash
user@hostname ~ $ sudo apt-get update && sudo apt-get upgrade -y
user@hostname ~ $ sudo DEBIAN_FRONTEND=noninteractive apt-get install -y git python-software-properties python g++ make ruby-dev nodejs npm nodejs-legacy nginx php5-fpm php5-cli php5-apcu php5-mysql php5-gd php5-json php5-curl php5-intl php5-imagick mysql-server mysql-client gettext imagemagick postfix
user@hostname ~ $ sudo gem install compass
```

## Install non-broken version of php5-apcu from 14.10 {#patch-apcu}
Ubuntu 14.04 ships with a version of php5-apcu with a fatal bug, run this to install a stable version from trusty-backports:

```language-bash
user@hostname ~ $ sudo apt-get install php5-apcu/trusty-backports
```

## Stop and disable default service instances {#disable-services}
Emergence will be configuring and launching nginx, mysql, and php5-fpm for us, so we need to get the default instances set up by Ubuntu out of the way:

```language-bash
user@hostname ~ $ sudo service nginx stop && sudo update-rc.d -f nginx disable
user@hostname ~ $ sudo service php5-fpm stop && (echo "manual" | sudo tee /etc/init/php5-fpm.override)
user@hostname ~ $ sudo service mysql stop && (echo "manual" | sudo tee /etc/init/mysql.override)
```

## Configure AppArmor to allow Emergence to manage `mysqld` {#apparmor}
AppArmor must be configured to allow MySQL to use Emergence files instead of the defaults:

```language-bash
user@hostname ~ $ echo -e "/emergence/services/etc/my.cnf r,\n/emergence/services/data/mysql/ r,\n/emergence/services/data/mysql/** rwk,\n/emergence/services/run/mysqld/mysqld.sock w,\n/emergence/services/run/mysqld/mysqld.pid rw," | sudo tee -a /etc/apparmor.d/local/usr.sbin.mysqld
user@hostname ~ $ sudo /etc/init.d/apparmor reload
```

## Increase shared memory limit {#shared-memory}
Ubuntu comes with a low limit of 32MB for shared memory. Emergence relies heavily on APC caching and needs kernel.shmmax increased to a more flexible amount. We'll use 128MB:

```language-bash
user@hostname ~ $ echo -e "kernel.shmmax = 268435456\nkernel.shmall = 65536" | sudo tee -a /etc/sysctl.d/60-shmmax.conf
user@hostname ~ $ sudo sysctl -w kernel.shmmax=268435456 kernel.shmall=65536user@hostname ~ $ echo -e "apcu.shm_size=128M\napc.shm_size=128M" | sudo tee -a /etc/php5/mods-available/apcu.ini
```

## Install Emergence from GitHub {#emergence-github}
Clone Emergence into your home directory, then use `npm` to install the package and its dependencies:

```language-bash
user@hostname ~ $ sudo npm install -g git+https://github.com/JarvusInnovations/Emergence
```

## Start Emergence {#start-emergence}
`npm -g` installed the kernel's startup script to `/usr/bin/emergence-kernel`. You can now launch it manually, or install the init script:

```language-bash
user@hostname ~ $ sudo wget http://emr.ge/dist/debian/upstart -O /etc/init/emergence-kernel.conf
user@hostname ~ $ sudo start emergence-kernel
```

## Create a site {#create-a-site}
You can now open your web browser to take control of Emergence and create your first site: [http://127.0.0.1:9083](http://127.0.0.1:9083). If you're installing to a remote machine, replace 127.0.0.1 with your remote IP or hostname.

When prompted, log in with the username and password <kbd>admin</kbd> / <kbd>admin</kbd>.

## Secure your installation {#secure}
Once you confirm that you are able to access the control panel, use the `htpasswd` tool provided in npm to delete the default admin account and create your own. Then restart the Emergence kernel to apply the changes:

```language-bash
user@hostname ~ $ sudo npm install -g htpasswd
user@hostname ~ $ sudo htpasswd -D /emergence/admins.htpasswd admin
user@hostname ~ $ [[[sudo htpasswd -s /emergence/admins.htpasswd ]]]myusername
user@hostname ~ $ sudo restart emergence-kernel
```

## (Optional) install Sencha CMD {#sencha-cmd}
Enter <kbd>/usr/local/bin</kbd> as the install path when prompted by Sencha's CMD installer:

```language-bash
user@hostname ~ $ sudo apt-get install openjdk-7-jre ruby1.9.3 unzip
user@hostname ~ $ wget http://cdn.sencha.com/cmd/3.1.2.342/SenchaCmd-3.1.2.342-linux-x64.run.zip
user@hostname ~ $ unzip SenchaCmd-*-linux-x64.run.zip
user@hostname ~ $ chmod +x SenchaCmd-*-linux-x64.run
user@hostname ~ $ sudo ./SenchaCmd-*-linux-x64.run
user@hostname ~ $ sudo mkdir /usr/local/bin/Sencha/Cmd/repo
user@hostname ~ $ sudo chown www-data:www-data -R /usr/local/bin/Sencha/Cmd/repo
```