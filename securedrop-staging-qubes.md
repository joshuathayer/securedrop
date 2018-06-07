## Deploying SecureDrop staging instance on Qubes

This assumes you have an up-to-date Qubes installation on a compatible laptop with at least 16GB RAM and 60GB free disk space.

### Overview

We're going to create three new standalone Qubes VMs:

- the securedrop application VM
- the monitoring VM
- a VM to provision the app and monitoring VMs, using existing Ansible playbooks

## download Ubuntu Trusty server ISO

On any exising Qube, download the Ubutnu Trusty server ISO, from

    http://releases.ubuntu.com/14.04/ubuntu-14.04.5-server-amd64.iso

## create the "build" VM

We're going to build a single, minimally configured Ubuntu VM. Once it's bootable, we'll clone it for the application and monitoring VMs.

In `dom0`, do the following:

    qvm-create sd-build --class StandaloneVM --property virt_mode=hvm --label green
    qvm-volume extend sd-app:root 20g

Using the Qubes Settings interface (Q menu -> sd-build -> Qubes Settings), set the VM's kernel to "None", and give it at least 2 GB of RAM.

While you're in the settings interface, note down the IP address Qubes gave the new VM and the configured gateway IP address.

### boot into installation medium

In dom0:

    qvm-start sd-build --cdrom=<download-vm>:/path/to/ubuntu-14.04.5-server-amd64.iso

where `download-vm` is the name of the VM to which you downloaded the ISO.

Start configuration.

At some point you'll need to manually set up the network interface, after DHCP failss. If you didn't mark it down down earlier, you can check the machine's IP and gateway via the Qubes GUI. When prompted, use enter that IP address, with a `/24` netmask (for example: `10.137.0.16/24`. Use Qubes' internal resolvers as DNS servers: `10.139.1.1` and `10.139.1.2`. Use the gateway address indicated in the Qubes Settings UI.

Give the new VM the hostname `sd-build`.

You'll be prompted to add a "regular" user for the VM: this is the user you'll be using later to SSH into the VM, so give it a username and password you're comfortable with.

During partitionaing, ensure the installer uses the full 20GB volume you've given it. There's no need to encrypt the filesystem.

During software installation, make sure you install the SSH server. You don't need to install anything else.

The installer will prompt about where to install GRUB: choose the default (MBR).

Once installation is done, let the machine shut down and then restart it with

    qvm-start sd-build

you should get a login prompt. Yay.

### Initial VM configuration

Before cloning this machine, we'll add some software we might want on all the staging VMs.

In the new VM's console, do:

    sudo apt update
    sudo apt dist-upgrade
    sudo apt install vim

Feel free to add anything else you need to make your console life happy.

Before we continue, let's allow your user to `sudo` without their password. Edit `/etc/sudoers` using `visudo` to make the sudo group line look like

    %sudo    ALL=(ALL) NOPASSWD: ALL

When initial configuration is done, `halt` the `sd-build` VM.

## clone app vm as monitor vm

In dom0:

    qvm-clone sd-build sd-app
    qvm-clone sd-build sd-mon

Try it:

    qvm-start sd-mon

On the console which eventually appears, should be able to log in as the user you created. Also start `sd-app` and log in to get a console there.

### Configure cloned VMs

We'll need to fix each machine's idea of its own IP. In `dom0`, use `qvm-ls -n` to discover the IP addresses of `sd-app` and `sd-mon`. In the console for each machine, edit `/etc/network/interfaces` to update the `address` line with the machine's IP.

`/etc/hosts` on each host needs to be modified to include all of these new VMs. On each host, add the IP and the hostname of each VM (`sd-build`, `sd-app`, `sd-mon`).

Finally, on each host edit `/etc/hostname` to reflect the machine's name.

Halt each machine, then restart each from `dom0`. The prompt in each console should reflect the correct name of the VM. You should be able to ping IPs on the internet.

### inter-VM networking

(Following https://www.qubes-os.org/doc/firewall/#enabling-networking-between-two-qubes)

We want to be able to ssh from a normal Qubes VM to these new standalone VMs, and the `sd-build` VM must be able to SSH to the `sd-app` and `sd-mon` VMs. In order to do so, we have to adjust the firewall on `sys-firewall`.

First decide which VM you'll be using to SSH to `sd-build`. Generally this will be something like your "work" VM. On dom0, do

    qvm-ls -n

to see the addresses of all the VMs. Note the address of your "source" VM ("work", eg) and of `sd-build`, `sd-mon`, and `sd-app` (the destinations).

Get a shell on `sys-firewall`. Enter the following

    sudo iptables -I FORWARD 2 -s <work-vm-address> -d <sd-build-addr> -j ACCEPT
    sudo iptables -I FORWARD 2 -s <sd-build-addr> -d <sd-app-addr> -j ACCEPT
    sudo iptables -I FORWARD 2 -s <sd-build-addr> -d <sd-mon-addr> -j ACCEPT

Now from your source vm, you should be able to do

    ssh <user-you-created>@<ip-of-sd-build>

and log in using the password you created.

(If you were unable to connect, try adding the following on each VM):

    sudo iptables -I INPUT -s <source address> -j ACCEPT

If everything worked as expected, make the changes persist over reboots: on the firewall VM, create or edit `/rw/config/qubes-firewall-user-script` to include the commands you ran (without the `sudo`).

(If required, on each destination VM, create/edit `/etc/rc.local` and add the `iptables` command you ran there (again without the `sudo`).

### ssh using keys

Later we'll be using Ansible to provision the application VMs, so we should make sure we can ssh between those machines without needing to type a password. On `sd-build`:

    ssh-keygen
    ssh-copy-id <user>@sd-app
    ssh-copy-id <user>@sd-mon

Confirm that you're able to ssh from `sd-build` to `sd-mon` and `sd-app` without being prompted for a password.

## SecureDrop Installation

If you don't already have a shell, SSH to `sd-build`, where we'll run all of the following.

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

(Debian:)

    sudo add-apt-repository \
      "deb [arch=amd64] https://download.docker.com/linux/debian \
      $(lsb_release -cs) \
      stable"

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

### clone the repo

Clone the securedrop server repo: follow instructions at https://docs.securedrop.org/en/latest/development/setup_development.html#fork-clone-the-repository), and either copy an SSH keypair known to github onto your new VM, or create a new keypair there and tell github about it.

Decide where you want to clone source code to on sd-server. `cd` to that directory, and do:

    git clone git@github.com:<your github user>/securedrop.git

### ansible, pip, virtualenv...

We need some software on `sd-build` in order to build securedrop. In the root of the your checked out code on `sd-build`, do:

    sudo apt install python-pip virtualenvwrapper
    source /usr/share/virtualenvwrapper/virtualenvwrapper.sh
    mkvirtualenv -p /usr/bin/python2 securedrop
    workon securedrop

Add `source /usr/share/virtualenvwrapper/virtualenvwrapper.sh` to your `~/.bashrc`. Now, when you log back in, you can

    workon securedrop

to enable your new python virtual environment.

Install development requirements:

    pip install -r securedrop/requirements/develop-requirements.txt

### build

Now we can build the .debs for the server!

    make build-debs

This will take some time.

### configuration: keys

Before we can provision our application, we need to create two GPG keys: one for our document submissions, and one for OSSEC. Use

    gpg --gen-key

to create each key. Note that the document key should be 4096 bits. To make further handling easier, when prompted use the name "SecureDrop" for the application key, and "OSS" for the OSS key. Otherwise the defaults are fine.

After keys are created, export both:

    gpg --export "SecureDrop" > SecureDrop.asc
    gpg --export "OSS" > OSS.asc

Copy each exported key to the `install_files/ansible-base/` directory.

Now run `gpg --fingerprint` to show the fingerprints of both the above keys. You'll need those for the next step.

### configuration

We're going to use `securedrop-setup` to configure our staging instance before deployment.

`ssh` to sd-build in a new shell. `cd` to your checked-out code. `securedrop-setup` builds its own virtual environment separate from the one we created for building, but I think it fails the first time through. In any case, run

    securedrop-admin setup

Wait for it to fail. Then

    source .venv/bin/activate
    pip install --upgrade pip

Then run the admin script again

    securedrop-admin setup

Then

    securedrop-admin sdconfig

This will prompt you for a number of details: the username for accessing the application hosts (use your own username), the IP addresses and hostnames for the app and monitoring machines, the paths to the keys you created in the previous steps, and the fingerprints of the those keys (use the output from the `gpg --fingerprint` command. You can use dummy information for the questions about sending OSSEC email, since our system won't actually be attempting to send mail. You shouldn't need to enable SSH over Tor.

When the script finishes it will leave site-specific configuration files in various places. We're ready to provision the VMs now.

Exit this shell!

## Provisioning

Back in the shell where you run `make build-debs`, we can finally use Ansible to configure the VMs we created. Run the following:

    ansible-playbook -i install_files/ansible-base/inventory-staging install_files/ansible-base/securedrop-staging.yml

The process involves a number of reboots. On Qubes, the VMs will simply halt, and we'll have to restart them by hand with

    qvm-start sd-app

and/or

    qvm-start sd-mon

on dom0.

If all goes well after some time the `sd-app` and `sd-mon` VMs will be automatically configured to run the securedrop staging environment.

## post-provisioning

### create admin user

SSH to `sd-app`. You should be able to do

    sudo su
    cd /var/www/securedrop
    ./manage.py add-admin

### tor service addresses

We want the Tor service addresses, of course.

On `sd-app`, do:

    sudo cat /var/lib/tor/services/journalist/hostname

to get the journalist interface and key. You'll want this in order to configure the workstation.

Get the source address with:

    sudo cat /var/lib/tor/services/source/hostname

Use Tor Browser in your `anon-whonix` Qube to try connecting to the source interface... make yourself a source and leave a submission!
