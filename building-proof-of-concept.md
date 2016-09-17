# Building a Proof of Concept Onion Site for a Normal, Cleartext Web Site

# WORK IN PROGRESS DO NOT USE YET

## Goal

Let's build an Onion site which serves proxied-and-rewritten content of the BBC website.

The BBC is (currently) suitable for exemplar experimentation because it does not use HTTPS to protect its content, plus it's unlike to complain from a corporate standpoint and has no concept of logged-in users to be interfered with.

## You Will Need

* Basic Linux sysadmin skills
* A fresh Ubuntu Server 16.04.1 instance, or similar
  * I use VirtualBox on a laptop for convenience
  * A virtual host or AWS instance would also be okay
* Outbound network access
* Maybe a couple of hours
* Coffee

## Notes

### Networking

Don't fret about firewalls or lack of inbound internet access; one of the joys of Onion sites is that they only use *outbound* network access.

Tor onions work because any hypothetically "inbound" traffic is received by coming backwards up a connection which was originally established *outbound*.

This also has the nice side effect that you can lock your onions in a DMZ "enclave", restricted to permit outbound internet access only, and limiting any other connectivity.

### Production

I recommend not using this process for anything other that Proof of Concept of your own website.

Certainly I consider the *rewrite HTML on the fly* approach which we a re describing here, to be greatly inferior to fixing your CMS to manage access from multiple and separate domainnames elegantly.

However, this is a nice, simple taster of *how easy it is to onionify a site*.

## Steps

This is probably not as optimal as it could be, but it is tested and works.

### Build your System

Install Ubuntu with "standard system utilities" and apply all patches; we will do everything else manually.

You might want to install sshd so that you have terminal cut-and-paste; I shall leave that undocumented, but please maintain security appropriately.

### Build `mitmproxy`

There is a version of mitmproxy available in the Ubuntu APT repositories, but it is old, out of date, and wrong with respect to the documentation that you will find on the web.

So we shall build it manually.  The commands I'm typing are as follows:

```sh
sudo -i # become root

apt-get install aptitude # if you can do without this, you're a hero

aptitude install libffi-dev libjpeg-dev libssl-dev libxml2-dev libxslt1-dev libyaml-dev python-dev python-pip zlib1g-dev

pip install --upgrade pip

pip install mitmproxy

mitmproxy --version
```

This should give you `mitmproxy 0.17` if everything works okay.

### Install Tor

### Set up an Onion Site

### Enable `mitmproxy`

### Connect to the Onion Site

### Iterate and Improve

