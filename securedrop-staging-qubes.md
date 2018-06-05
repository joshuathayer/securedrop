
## start with a debian based "work" vm.

You'll be building the SD debs here, and configuring it as a development machine.

## download Ubuntu Trusty server ISO

http://releases.ubuntu.com/14.04/ubuntu-14.04.5-server-amd64.iso

## make a new VM

    qvm-create sd-server --class StandaloneVM --property virt_mode=hvm --label green
    qvm-volume extend sd-server:root 20g

Using the GUI, set the new VM's kernel to "None", and give it at least a GB of ram.

## boot

    qvm-start sd-server --cdrom=<download-vm>:/path/to/ubuntu-18.04-live-server-amd64.iso


Start configuration. Note you'll need to manually set up the interface: check the machine's IP via the Qubes GUI. That GUI gives a /32 network address: I reduced it to a /24 (10.137.0.0/24) in order for it to work.

You may need to jump through some hoops to make sure the installation uses the entire volume you've given it. XXX

Once installation is done, let the machine shut down and then restart it with

    qvm-start sd-server

you should get a login prompt. Yay.

## networking

Following https://www.qubes-os.org/doc/firewall/#enabling-networking-between-two-qubes.

We want to be able to ssh from a normal vm to this new standalone vm. In order to do so, we have to adjust the firewall on sys-firewall.

First decide from which VM you'll be ssh'ing to your server. On dom0, do

    qvm-ls -n

to see the addresses of all the VMs. Note the address of your "source" VM and of your new sd-server VM.

Get a shell on sys-firewall. Enter the following

    sudo iptables -I FORWARD 2 -s <source address>  -d <dest address> -j ACCEPT

Now, log in to sd-server on the console you got from running `qvm-start`. Once you have a shell, enter:

    sudo iptables -I INPUT -s <source address> -j ACCEPT

Now, from your source vm, you should be able to do

    ssh <user you created>@<ip of sd-server>

and log in using the password you created.

If everything worked as expected, make the changes persist over reboots: on the firewall VM, create or edit `/rw/config/qubes-firewall-user-script` to include the command you ran (without the `sudo`). On `sd-server`, create/edit `/etc/rc.local` and add the `iptables` command you ran there (again without the `sudo`).

## SD Installation

Unless otherwise noted, all the following happens in a shell in sd-server. Ideally, ssh there from a Qube so you can copy/paste, etc.

We're going to follow https://docs.securedrop.org/en/latest/development/setup_development.html

### baby steps

  sudo apt update
  sudo apt install -y make git

### Docker

Following the Docker docs:


    sudo apt install \
      apt-transport-https \
      ca-certificates \
      curl \
      software-properties-common

    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

    sudo apt-key fingerprint 0EBFCD88
    # confirm figerprint 9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88

    sudo add-apt-repository \
     "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
     $(lsb_release -cs) \
     edge"

Note that Ubuntu Bionic does not have `stable` docker-ce packages yet, so we're using `edge` in the above command.

    sudo apt update
    sudo apt install docker-ce

Test that we're working:

    sudo docker run hello-world

Hurrah. Post-installation things:

    sudo groupadd docker
    sudo usermod -aG docker $USER

Exit your ssh session, then ssh back in. Confirm you can run containers as your normal user:

    docker run hello-world

should work without error.

### basic requirements

    sudo apt-get install -y build-essential libssl-dev libffi-dev python-dev \
        dpkg-dev git linux-headers-$(uname -r) virtualbox

Now install vagrant... on Ubuntu Bionic the packaged version of Vagrant is new enough, so that's as simple as

    sudo apt install vagrant

If you're not on Bionic you may need to install Vagrant manually. See https://docs.securedrop.org/en/latest/development/setup_development.html#id1

### clone the repo

If you're going to be developing sd-server code, you'll want to clone your own fork of the securedrop project. I'll leave it as an exercise to the reader to work through that, but you'll need fork the project (https://docs.securedrop.org/en/latest/development/setup_development.html#fork-clone-the-repository), and either copy an SSH keypair known to github onto your new VM, or create a new keypair there and tell github about it.

Decide where you want to clone source code to on sd-server. `cd` to that directory, and do:

    git clone git@github.com:<your github user>/securedrop.git

### ansible, pip, virtualenv...

    sudo apt install python-pip virtualenvwrapper
    source /usr/share/virtualenvwrapper/virtualenvwrapper.sh
    mkvirtualenv -p /usr/bin/python2 securedrop
    workon securedrop

Add `source /usr/share/virtualenvwrapper/virtualenvwrapper.sh` to your `~/.bashrc`. Now, when you log back in, you can

    workon securedrop

to enable your new python virtual environment.

Install development requirements:

    pip install -r securedrop/requirements/develop-requirements.txt

## build

Now we can build the .debs for the server!

    make build-debs

## Start up staging instance

    vagrant up /staging/

