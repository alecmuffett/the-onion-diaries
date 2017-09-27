# Building a Basic "Production" Onion Server on Ubuntu

(c) 2016 Alec Muffett - licensed under CC-BY-SA-4.0

#### Goals:
* create a fully up-to-date Ubuntu instance
* with automatic security patches (network access permitting)
* with tested Tor access
* with 4x preconfigured, random onion addresses
* with 4x corresponding fake IPv4 network interfaces to aid application configuration
* with local-only email to satisfy application dependencies
* with a standard hostname which will not resolve in DNS (invalid.invalid)
* with a standard timezone (UTC)
* with a basic firewall of inbound connections
* with openssh for access to local network

#### Optional Goals
* with Tor configuration files under Git revision control

#### Non-Goals:
* we are not forcibly standardising locale
  * not everyone will cope with english language
* we are not building a Docker container, nor Ansible, nor Qubes
  * platforms like those are a barrier to entry because "learning curve" apart from any other reason
  * developers are free to use the content of this document to roll their own platforms
  * doubtless some day this can all be shrinkwrapped into a container, but let's get the process right, first
* we are not forcing outbound TCP connection attempts to go over Tor
  * see future document for options in this space

#### Todo:
* offer optional instruction on standardising locale?

#### Notes:
* text marked *"verbatim"* should be carefully typed/pasted exactly as seen on screen
  * generally it is for purposes of security or for later auto-editing
* code that can be pasted is between `# BEGIN PASTE` and `# END PASTE`
  * if you are (rightly) worried about pasting from a web browser, git-clone the document and use that.

#### Regarding Cut-and-Paste:

* This document has a lot of typing and suggested cut-and-paste
* If your Linux solution does not enable you to paste text into a root shell, you might like to enable `sshd` temporarily (or permanently?) when the machine is first installed with Ubuntu, and then log in *again* using `ssh` so that you can paste command text into the `ssh` terminal window.  That's your call, and it's beyond the scope of this document to describe how to do it on your local system.

## Install Ubuntu Server

Download Ubuntu Server 16.04.1

* 64-bit version: http://www.ubuntu.com/download/server
* 32-bit version: available via Bittorrent "alternative downloads"
  * http://www.ubuntu.com/download/alternative-downloads

Follow the instructions to install Ubuntu Server. 

### Notes

* configure network interfaces carefully
* set the hostname to be `invalid` (*verbatim*)
  * note: configure timezone, keyboards, layout & boot-loader as you see fit
  * note: we shall standardise timezone later
* create an account for the sysadmin
  * note: there probably is not much benefit in encrypting your home directory on a server.
* select: `install security updates automatically`
* install both `standard system utilities` and (optionally) `OpenSSH server`
  * **DO NOT INSTALL ANY OTHER ADDITIONAL PACKAGES (YET)**

## Initial Setup

Log in, and do:

```sh
# THESE MUST BE EXECUTED ONE LINE AT A TIME BECAUSE USER INPUT
sudo -i # to give you a root shell
apt-get install aptitude
aptitude update
aptitude upgrade
```

If you're going to enable `ssh` for remote access from your local network, this is probably the right time to do it.

## Installing Tor

In a browser elsewhere, retreive the instructions for installing Tor from https://www.torproject.org/docs/debian.html.en

* Set the menu options for:
  * run *Ubuntu Xenial Xerus*
  * and want *Tor*
  * and version *stable*
  * and read what is now on the page.
* Configure the APT repositories for Tor
  * I recommend that you add the tor repositories into a new file
  * Use: `/etc/apt/sources.list.d/tor.list` or similar
* Do the gpg thing
* Do the apt update thing
* Do the tor installation thing

## Fake a Fully Qualified Domain Name for Email

do:

```sh
# BEGIN PASTE
perl -pi~ -e 's/invalid/invalid invalid.invalid/' /etc/hosts
# END PASTE
```

The first couple of lines of `/etc/hosts` should probably now look like this:

```
127.0.0.1       localhost
127.0.1.1       invalid invalid.invalid
```

## Install Local-Only Email
### (do it now, because package dependencies will bite you later)

do:

```sh
# THESE MUST BE EXECUTED ONE LINE AT A TIME BECAUSE USER INPUT
aptitude install postfix mailutils
```

* Under *Postfix Configuration*, select `Local only` as *General type of mail configuration*
* set the email hostname `invalid.invalid` (*verbatim*) to match the above FQDN hack
  * you will be adding the extra `.invalid` top level domain to the existing name

## Optional: Make Git shut up about Email addresses

do:

```sh
# THESE MUST BE EXECUTED ONE LINE AT A TIME BECAUSE USER INPUT
env EDITOR=vi git config --global --edit
```

...and either uncomment the relevant lines or fix it properly

## Optional: Put the Tor configuration under revision control

Because we all can make mistakes:

```sh
# BEGIN PASTE
cd /etc/tor
git init
git add .
git commit -m initial
# END PASTE
```

## Standardise on UTC timezone

do:

```sh
# BEGIN PASTE
timedatectl set-timezone Etc/UTC
# END PASTE
```

## Add Virtual Network Addresses to /etc/hosts
### (we create 4 as an example)

Notes:

- these are addresses in separate "/30" subnets of the DHCP address space
  - the DHCP address space is not routable in the same way as RFC1918 but is unlikely to clash with extant subnets
  - if this really upsets you, replace `169.254.255` with whatever, throughout the rest of this process
    - we're just betting that your local DHCP administrator would rather lose a tooth than use `255` in a netaddr
- we use the first usable address in each of separate "/30"-type subnets to inhibit routing and cross-contamination.
- because of what we are trying to achieve we could perhaps try using "/31" pairs and treat them as point-to-point, but that would be complex and contentious, whereas this is vanilla networking.

```sh
# BEGIN PASTE
cat >>/etc/hosts <<EOT
# 'shadow' onion ip-addresses
169.254.255.253 osite0.onion
169.254.255.249 osite1.onion
169.254.255.245 osite2.onion
169.254.255.241 osite3.onion
EOT
# END PASTE
```

## Disable IP Forwarding and Multihoming

Manual work! Edit: `/etc/sysctl.conf` - and uncomment and set to 0 the following:

```
net.ipv4.ip_forward=0
net.ipv6.conf.all.forwarding=0
```
...and also uncomment and set to 1 the following...

```
net.ipv4.conf.default.rp_filter=1
net.ipv4.conf.all.rp_filter=1
```


## Backup the Tor config file for reference, and nuke the original

It's basically all commented out anyway, so let's have a clean slate.

do:

```sh
# BEGIN PASTE
cp -p /etc/tor/torrc /etc/tor/torrc.orig
cp /dev/null /etc/tor/torrc
# END PASTE
```

## Constrain Tor SOCKS access to literally 127.0.0.1

do:

```sh
# BEGIN PASTE
cat >>/etc/tor/torrc <<EOT
SOCKSPolicy accept 127.0.0.1
SOCKSPolicy reject *
#
EOT
# END PASTE
```

## Create Onion Addresses
### (we create 4 as an example)

Annoyingly, tor will not create a HiddenServiceDir unless a corresponding HiddenServicePort is defined, so we will have to define them temporarily and then switch them off later. This should be safe since nothing is listening to port 80 on the machine at the moment.

do:

```sh
# BEGIN PASTE
cat >>/etc/tor/torrc <<EOT
# ---- section for osite0.onion ----
HiddenServiceDir /var/lib/tor/osite0/
HiddenServicePort 80 osite0.onion:80
#
# ---- section for osite1.onion ----
HiddenServiceDir /var/lib/tor/osite1/
HiddenServicePort 80 osite1.onion:80
#
# ---- section for osite2.onion ----
HiddenServiceDir /var/lib/tor/osite2/
HiddenServicePort 80 osite2.onion:80
#
# ---- section for osite3.onion ----
HiddenServiceDir /var/lib/tor/osite3/
HiddenServicePort 80 osite3.onion:80
#
EOT
# END PASTE
```

...this should be safe since we're not actually running anything on port 80 yet.

## Restart Tor

do:

```sh
# BEGIN PASTE
/etc/init.d/tor restart
# END PASTE
```

This will create the hidden service directories cited above, etc

## Check Tor Connectivity

Wait 30+ seconds, and then do:

```sh
# BEGIN PASTE
torsocks curl https://www.facebook.com/si/proxy/ ; echo ""
# END PASTE
```

...this should print: `tor`

Next, do:

```sh
# BEGIN PASTE
torsocks curl https://www.facebookcorewwwi.onion/si/proxy/ ; echo ""
# END PASTE
```

...this should print: `onion`

The tests should **not** print: `normal` - if they do, it's a burp/error.

If your Tor daemon is slow to connect to the Tor network, you might want to wait a bit longer and try again.

## Configure Virtual IP interfaces that will map to the Onions
### (this is where the constraint of 4 example addresses is hardcoded)

do:

```sh
# BEGIN PASTE
# settings
NUMDUMMIES=4 # you can make this bigger if you want
# for now
modprobe dummy numdummies=$NUMDUMMIES
# for reboot
echo dummy >> /etc/modules-load.d/dummy.conf
echo options dummy numdummies=$NUMDUMMIES >> /etc/modprobe.d/dummy.conf
chmod 644 /etc/modules-load.d/dummy.conf /etc/modprobe.d/dummy.conf

# for config
cat >>/etc/network/interfaces <<EOT
auto dummy0
iface dummy0 inet static
  address 169.254.255.253
  netmask 255.255.255.252
  broadcast 169.254.255.255

auto dummy1
iface dummy1 inet static
  address 169.254.255.249
  netmask 255.255.255.252
  broadcast 169.254.255.251

auto dummy2
iface dummy2 inet static
  address 169.254.255.245
  netmask 255.255.255.252
  broadcast 169.254.255.247

auto dummy3
iface dummy3 inet static
  address 169.254.255.241
  netmask 255.255.255.252
  broadcast 169.254.255.243
EOT
# END PASTE
```

## Create the Virtual IP interfaces

do:

```sh
# BEGIN PASTE
ifup -a
ifconfig -a
# END PASTE
```

...and you should see the four new network interfaces

## Install a Firewall to block incoming network connections

You've done the work above in order to create onion-network-addresses and create easy ways to configure applications that can talk to them consistently, with a reasonable minimum of useful metadata that could be used to identify the machine's location or "true" IP address which would open it up to (eg:) DDoS attack.

The next logical step for the attacker would be to scan networks looking for machines named `invalid.invalid` and attack them anyway, so it's wise for this server to be very limited in terms of to whom it will listen for incoming IP connections.

Therefore we install a firewall and default-deny all incoming connection attempts:

```sh
# THESE MUST BE EXECUTED ONE LINE AT A TIME BECAUSE USER INPUT
ufw enable
ufw status verbose
# consider very carefully before adding any stuff like this:
# okay:     ufw allow from $SPECIFIC_ADDRESS to $MY_IP_ADDRESS port 22
# bad:      ufw allow from $SPECIFIC_ADDRESS to any port 22
# terrible: ufw allow from any to any port 22
# watch GitHub for future documents expanding on these matters
```

## Tor Finalisation - **THE GRAND RENAMING** -

do:

```sh
# BEGIN PASTE
cp /etc/hosts /etc/hosts.orig
for odir in /var/lib/tor/osite?/ ; do
oname=`basename $odir`
oaddr=`cat $odir/hostname`
perl -pi~ -e "s/$oname.onion/$oaddr/" /etc/hosts
perl -pi~ -e "s/$oname.onion/$oaddr/" /etc/tor/torrc
done
# END PASTE
```

Then test the resolution of IPv4-equivalent onion names:

```sh
# BEGIN PASTE
for odir in /var/lib/tor/osite?/ ; do
oaddr=`cat $odir/hostname`
ping -q -c 1 $oaddr
done
# END PASTE
```

For each address you should see something like this:

```
root@invalid:~# ping -q -c 1 zxd674r63j44zfj7.onion
PING zxd674r63j44zfj7.onion (169.254.255.253) 56(84) bytes of data.

--- zxd674r63j44zfj7.onion ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.096/0.096/0.096/0.000 ms
```

...demonstrating that `zxd674r63j44zfj7.onion` is a name that is bound to a virtual interface on your machine, and to which `torrc` is now configured forward connections on port 80:

```
root@invalid:~# grep zxd674r63j44zfj7.onion /etc/tor/torrc
HiddenServicePort 80 zxd674r63j44zfj7.onion:80
```

## Disable the temporary HiddenServicePorts

do:

```sh
# BEGIN PASTE
perl -pi~ -e 's/^HiddenServicePort/#HiddenServicePort/' /etc/tor/torrc
# END PASTE
```

## Reboot to apply all changes and new executables

do:

```sh
# BEGIN PASTE
shutdown -r now
# END PASTE
```

If you are unwilling to do this, at least restart Tor with: `/etc/init.d/tor restart` - otherwise you will not pick up the changes that we just made to the `torrc` file.  But it's better to reboot and test everything

## After reboot, Re-Check Outbound Tor Connectivity

See the **Check Tor Connectivity** section above. Do that again.

## Check For Promiscuous Network Listeners

Do:

```sh
# THESE MUST BE EXECUTED ONE LINE AT A TIME BECAUSE USER INPUT
sudo netstat -a --inet --program  | awk '$6=="LISTEN" && $4~/^\*/'
```
This will print a list of network sockets which are listening to all local network interfaces simultaneously; in theory you will only see `sshd` and it will look something like:

```
tcp  0  0 *:ssh  *:*  LISTEN  1276/sshd
```

This tells you that process ID `1276` is an instance of `sshd`  which is listening to the `ssh` port on all network interfaces.  You may want to consider the security risks, and perhaps reconfigure this (and other) programs to listen only to specific, especially non-onion, network interfaces.

If you unexpectedly see more than `sshd`, ask in the forums what it is.

## ---- Finish ----

You now have a server which is configured with (up to) four onion addresses. You may disable any onion addresses that you are not using (by editing `/etc/tor/torrc`) or you may have avoided creating them in the first place.

The reason for creating IPv4 addresses to act as "shadows" for the onion addresses is one of Unix access control - eg: Apache "Listen" and VirtualHost directives can be configured clearly and unambiguously with the given Onion name, which will resolve also locally and (hopefully) avoid complaints.

Also: applications which enforce access-control on the basis of source IP address can be configured inherit and resolve the name of the Onion address through which the traffic arrived, by virtue of the Tor daemon connecting to the correspondingly-named IP address.

There is a small risk here that bad system administrators will permit the contents of (eg:) /var/lib/tor/oside0/hostname to get out of sync with either/both of `/etc/hosts` or `/etc/tor/torrc`.  So don't let that happen.

### How This Works

On an less "advanced" onion server, the `torrc` would probably contain configurations like:

```
HiddenServicePort 80 localhost:80
```

This has two downsides:

1. connections to the Onion Address on port 80 will appear to come from `localhost` which may falsely convey extra trustworthiness or special/privileged access to whatever process is listening on port 80
1. (restating the above) the process listening on port 80 cannot distinguish whether the connection actually came from `localhost` or from an onion address, and if the latter obviously cannot distinguish from *which* onion address

But there's a feature in Unix/Linux which we can use, that when a TCP connection is made to an internal network interface on a system, both the source and destination IP addresses of that connection are set to match the IP address of the interface. This allows us to disambiguate onion connections from each-other and from `localhost`. 

So with the design specified in this document the system administrator may run multiple onion addresses (eg: `a1a1a1a1a1a1a1a1.onion` and `b2b2b2b2b2b2b2b2.onion`) on one system and configure Tor thusly:

```
# ---- section for a1a1a1a1a1a1a1a1.onion ----
HiddenServiceDir /var/lib/tor/osite1/
HiddenServicePort 80 a1a1a1a1a1a1a1a1.onion:80
# ---- section for b2b2b2b2b2b2b2b2.onion ----
HiddenServiceDir /var/lib/tor/osite2/
HiddenServicePort 80 b2b2b2b2b2b2b2b2.onion:80
```

...and then may define (say) separate Apache daemons with:

```
Listen a1a1a1a1a1a1a1a1.onion:80 # or equivalent IP address
#...and separately...
Listen b2b2b2b2b2b2b2b2.onion:80 # or equivalent IP address
```

...and the processes will retain separation and may even **enforce access control** - that requests to them must come from/via onion A1, or onion B2, or tuples thereof, respectively.

### Next Steps

You should be good to install actual programs (eg: webservers) on the server:

* configure them to listen on the (fake IP address) *named-onion-addresses*
* amend `/etc/tor/torrc` to expose the relevant *named-onion-address* port-numbers to onionspace
  * example: `HiddenServicePort 80 zxd674r63j44zfj7.onion:80 # in the appropriate section`
  * remember: it is up to *you* to keep these in synchronisation
* restart the tor service to kick them into life

----

# Notes

## Putting 4 Onion Addresses on a Server? OMGWTFBBQ?

Yeah, well, whatever. Someone is gonna want more than one, so I might as well do the subnet math for them now.

Yes there is advice to not run more than 1 onion address per machine that is attached to the internet.

Anyone who is *actually* worried about being deanonymised can disable the extra addresses in `torrc` and install more systems if they actually need more than 1 onion address.

## Those DHCP IP Addresses? OMGWTFBBQ?

Basically we have created four little subnets, each of which can hold a maximum of two computers, and in each of those subnets we are using only one address - see `*-addr-1`, below.

Why do this? For convenience we want virtual IPv4 addresses which exist only on the server and which are not routable across the internet; we will then map those 1-to-1 against Onion addresses, and use them systematically in the `torrc` file so that processes (Apache, sshd, etc...) that do `gethostbyname()` on their connection's source address will see (eg:) `zxd674r63j44zfj7.onion` rather than localhost.

Quite a lot of processes would give `localhost` special privileged access, so marking inbound Onion connections as coming from somewhere other than `localhost` is a good idea.

It just seems sane, therefore to use the onion address as a hostname, instead,

The subnets are computed like this, using a special script which does a bunch of simple math and spits out the configuration:

```
$ mknetmask 169.254.255.254/30
# 169.254.255.252/30: is a class B network and supports 2 hosts
# 169.254.255.252/30: netaddr 169.254.255.252 netmask 255.255.255.252 broadcast 169.254.255.255
# 169.254.255.252/30: netaddr 0xa9fefffc netmask 0xfffffffc broadcast 0xa9feffff
169.254.255.252 net-169-254-255-252-slash-30-netaddr
169.254.255.253 net-169-254-255-252-slash-30-addr-1
169.254.255.254 net-169-254-255-252-slash-30-addr-2
169.254.255.255 net-169-254-255-252-slash-30-broadcast

$ mknetmask 169.254.255.249/30
# 169.254.255.248/30: is a class B network and supports 2 hosts
# 169.254.255.248/30: netaddr 169.254.255.248 netmask 255.255.255.252 broadcast 169.254.255.251
# 169.254.255.248/30: netaddr 0xa9fefff8 netmask 0xfffffffc broadcast 0xa9fefffb
169.254.255.248 net-169-254-255-248-slash-30-netaddr
169.254.255.249 net-169-254-255-248-slash-30-addr-1
169.254.255.250 net-169-254-255-248-slash-30-addr-2
169.254.255.251 net-169-254-255-248-slash-30-broadcast

$ mknetmask 169.254.255.245/30
# 169.254.255.244/30: is a class B network and supports 2 hosts
# 169.254.255.244/30: netaddr 169.254.255.244 netmask 255.255.255.252 broadcast 169.254.255.247
# 169.254.255.244/30: netaddr 0xa9fefff4 netmask 0xfffffffc broadcast 0xa9fefff7
169.254.255.244 net-169-254-255-244-slash-30-netaddr
169.254.255.245 net-169-254-255-244-slash-30-addr-1
169.254.255.246 net-169-254-255-244-slash-30-addr-2
169.254.255.247 net-169-254-255-244-slash-30-broadcast

$ mknetmask 169.254.255.241/30
# 169.254.255.240/30: is a class B network and supports 2 hosts
# 169.254.255.240/30: netaddr 169.254.255.240 netmask 255.255.255.252 broadcast 169.254.255.243
# 169.254.255.240/30: netaddr 0xa9fefff0 netmask 0xfffffffc broadcast 0xa9fefff3
169.254.255.240 net-169-254-255-240-slash-30-netaddr
169.254.255.241 net-169-254-255-240-slash-30-addr-1
169.254.255.242 net-169-254-255-240-slash-30-addr-2
169.254.255.243 net-169-254-255-240-slash-30-broadcast
```

Aside for fellow network programmers: yes I could screw this down even tighter and use a "/31" but why mess around with contentious netmasks that are meant for point-to-point links?
