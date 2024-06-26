Schroot Setup

## Package caching (optional)

Install the caching proxy

```bash
sudo apt-get -y install apt-cacher-ng
```

Add an override for configuration

```bash
sudo tee /etc/apt-cacher-ng/zz_override.conf  >/dev/null <<EOF
Port:3142
BindAddress: 127.0.0.1
PassThroughPattern: .*
ExThreshold: 7
ExStartTradeOff: 1000m
EOF
```

Enable the proxy

```bash
if [ -z ${WSL_DISTRO_NAME} ]; then
    sudo update-rc.d -f apt-cacher-ng defaults
    sudo update-rc.d -f apt-cacher-ng enable
else
    sudo sed -zi '/\[boot\]/!s/$/\n\[boot\]\n/' /etc/wsl.conf
    sudo sed -i '/\[boot\]/acommand=\"/etc/init.d/apt-cacher-ng restart;\"\n' /etc/wsl.conf
fi
```

Start the daemon

```bash
sudo /etc/init.d/apt-cacher-ng start
sudo /etc/init.d/apt-cacher-ng status
```

# Bootstrap filesystem

## Install

```bash
sudo apt-get install debootstrap
```

## Prebuild tasks

Some recent updates have removed numerous links from the deployment package.  We can add those back before we build our environment.

```bash
cd /usr/share/debootstrap/scripts
for DISTRO in artful bionic cosmic disco eoan focal groovy hardy hirsute impish intrepid jammy jaunty karmic kinetic lucid lunar mantic maverick natty noble oneiric precise quantal raring saucy utopic vivid wily xenial yakkety zesty
do
    if [ ! -L ${DISTRO} ]; then
        sudo ln -v -s gutsy ${DISTRO}
    else
        echo "'${DISTRO}' already exists"
    fi
done
for DISTRO in bookworm bullseye buster jessie squeeze stretch trixie trusty wheezy
do
    if [ ! -L ${DISTRO} ]; then
        sudo ln -v -s sid ${DISTRO}
    else
        echo "'${DISTRO}' already exists"
    fi
done
for DISTRO in lenny
do
    if [ ! -L ${DISTRO} ]; then
        sudo ln -v -s etch ${DISTRO}
    else
        echo "'${DISTRO}' already exists"
    fi
done
cd ~
```

## Building filesystem

If you have set up the caching proxy, make sure you have the environment variables set before running the bootstrap.

```bash
export DEBOOTSTRAP_PROXY="http://localhost:3142/"
export http_proxy="http://localhost:3142"
export ftp_proxy="http://localhost:3142"
```

Build the bootstrap file system.  It must be run using with the "-E" parameter to make sure that the proxy variables are passed correctly.  This is not required if it is being run from a script.

```bash
sudo mkdir -pv /chroot/
```

```bash
sudo -E debootstrap \
    --arch=amd64 \
    --variant=minbase \
    --include=ubuntu-minimal,build-essential,git \
    focal \
    /chroot/focal-amd64/
```

The required files should be download and installed in the correct location.

# chroot access

Install the schroot command to help manage access to the bootstrapped environment 

```bash
sudo apt-get -y install schroot
```

Copy an existing profile so we don't have to modify any of the default settings, and add  our sudoers profile into the chroot environment.

```bash
sudo mkdir -pv /etc/schroot/custom/
sudo cp -v /etc/schroot/default/* /etc/schroot/custom/
for SUDO in $(sudo visudo -c | grep "parsed OK" | cut -d ':' -f 1 | egrep -v "(sudoers|README)$"); do
    echo ${SUDO} | sudo tee --append /etc/schroot/custom/copyfiles;
done
```

Create a configuration file for accessing and managing the new chroot environment.  Ensure that profile is set to the name "custom" so that it loads the files we added above.

```bash
sudo tee /etc/schroot/chroot.d/focal-amd64.conf >/dev/null <<EOF
[focal-amd64]
description=Ubuntu Focal (20.04)
type=directory
directory=/chroot/focal-amd64
users=${USER}
groups=$(groups | tr ' ' '\n' | egrep -v "(${USER}|cdrom|dip|lxd)" | tr '\n' ',' | sed 's/,$//g')
root-users=${USER}
root-groups=$(groups | tr ' ' '\n' | egrep -v "(${USER}|cdrom|dip|lxd)" | tr '\n' ',' | sed 's/,$//g')
profile=custom
personality=linux
message-verbosity=normal
preserve-environment=false
#command-prefix=eatmydata
EOF
sudo vi /etc/schroot/chroot.d/focal-amd64.conf
```

## Post-Build tasks

```bash
schroot -c focal-amd64 -u root -- <<EOF
if [ ! -f /etc/profile.d/05-environment.sh ]; then
cat >> /etc/profile.d/05-environment.sh << SCRIPT
    export http_proxy='http://localhost:3142'
    export HTTP_PROXY='http://localhost:3142'
    export https_proxy='http://localhost:3142'
    export HTTPS_PROXY='http://localhost:3142'
    export ftp_proxy='http://localhost:3142'
    export FTP_PROXY='http://localhost:3142'
    export no_proxy='localhost,127.0.0.1'
    export NO_PROXY='localhost,127.0.0.1'
SCRIPT
fi
EOF
```

Ensure that the system locale, time zones and console encoding are correctly set

```bash
schroot -c focal-amd64 -u root -- <<EOF
    echo "Etc/UTC" > /etc/timezone
    dpkg-reconfigure -f noninteractive tzdata
    sed -i -e 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen
    sed -i -e 's/# nb_NO.UTF-8 UTF-8/nb_NO.UTF-8 UTF-8/' /etc/locale.gen
    dpkg-reconfigure --frontend=noninteractive locales
    update-locale LC_ALL=en_US.UTF-8 LANG=en_US.UTF-8
    locale-gen "en_US.UTF-8"
EOF
```

Enable standard repositories for this distribution

```bash
schroot -c focal-amd64 -u root -- <<EOF
    export DEBIAN_FRONTEND=noninteractive
    sudo apt update
    apt-get -y \
        --no-install-recommends \
        --no-install-suggests \
        install software-properties-common 2>/dev/null
    echo '' > /etc/apt/sources.list
    add-apt-repository --no-update --yes \
        "deb http://gb.archive.ubuntu.com/ubuntu/ \$(lsb_release -sc) \
        main restricted universe multiverse" 2>/dev/null
    add-apt-repository --no-update --yes \
        "deb http://gb.archive.ubuntu.com/ubuntu/ \$(lsb_release -sc)-updates \
        main restricted universe multiverse" 2>/dev/null
    add-apt-repository --no-update --yes \
        "deb http://gb.archive.ubuntu.com/ubuntu/ \$(lsb_release -sc)-backports \
        main restricted universe multiverse" 2>/dev/null
    add-apt-repository --no-update --yes \
        "deb http://gb.archive.ubuntu.com/ubuntu/ \$(lsb_release -sc)-security \
        main restricted universe multiverse" 2>/dev/null
    echo
    cat /etc/apt/sources.list
    echo
    apt-get update
EOF
```

If this chroot is to be used for compiling and packaging then enable the source repositories is recommended

```bash
schroot -c focal-amd64 -u root -- <<EOF
    sed -i '/deb-src/s/^# //' /etc/apt/sources.list
    echo
    cat /etc/apt/sources.list
    echo
    apt-get update
EOF
```

Install any missing, required or preferred packages.

```bash
schroot -c focal-amd64 -u root -- <<EOF
    apt-get update 
    apt-get -y install vim ssh wget curl 2>/dev/null
    apt-get -y --fix-broken install
EOF
```

Ensure the latest packages are installed.

```bash
schroot -c focal-amd64 -u root -- <<EOF
    export DEBIAN_FRONTEND=noninteractive
    apt-get clean
    apt-get update
    apt-get -y -o Dpkg::Options::="--force-confnew" -fuy dist-upgrade 2>/dev/null
    apt-get -y purge
    apt-get -y --purge autoremove
EOF
```

More stuff goes here....
