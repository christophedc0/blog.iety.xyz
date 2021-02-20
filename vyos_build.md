# Vyos Build

I will explain in short, but really very short.. how to build your own vyos image.

## Initial setup

Create a LXC/VM or other environment with Ubuntu/Debian and make sure to have atleast 80GB free disk space.

Run the following:

```
apt install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common
curl -fsSL https://download.docker.com/linux/debian/gpg | apt-key add -
add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
apt update && apt upgrade -y && apt install docker-ce -y
docker pull vyos/vyos-build:current
```

Now go grab a coffee & chill.

## Updating the docker build image

```
docker run --rm -it --privileged -v $(pwd):/vyos -w /vyos vyos/vyos-build:current bash
git clone -b current --single-branch https://github.com/vyos/vyos-build
cd vyos-build
git status
git pull
```

## Configuring your image
I included a few additional packages:
  * nmap
  * htop
  * screen
  * rsync
  * glances
  * prometheus-node-exporter
  * ruby2.5
  * facter
 

```
 ./configure --architecture amd64 --build-by "Christophe DC" --build-type release --version "1.3-CDC-$(date +%Y%m%d)"  --custom-package "nmap htop screen rsync glances prometheus-node-exporter ruby2.5 facter"
```

You can change several things in the above code block.
Things you can change:
  * --architecture
  * --build-by
  * --version
  * --custom-package

## Make the image

```
make iso
```

Grab another coffee & lean back.

## Finished

When it's finished, I like to copy the image straight into my vyos environment.

```
scp build/vyos-1.3-CDC-20210220-amd64.iso vyos@10.32.0.1:/config/vyos-1.3-CDC-20210220-amd64.iso
```
You ~~can &~~ should change several things in the above code block.

## Now what do I do ?

Log in to your vyos environment and install the image as following:
```
add system image /config/vyos-1.3-CDC-20210220-amd64.iso
reboot
```

## Verify what you've done.

```
show version
```
