# openvpn-ubiquity

## Purpose

This package auto-generates SSL certificates, and helps configure OpenVPN on
Ubiquity EdgeOS routers as well as OpenVPN clients.  The cert generation
features of this package may be used to automatically create a basic CA along
with server, client, and peer certificates for any use... not just OpenVPN.

This package:

* Create CA keys and certificates
* Generates Diffie Hellman params
* Generates Server Certs
* Generates Client Cert
* Generates OpenVPN Peer Certs
  * Creates OpenVPN config files with certs inline for easy distribute

CA files will output to ./ca, and other certificate files to ./certs.

Furthermore, instructions for configuring OpenVPN on Ubiquity EdgeOS
Routers and OpenVPN clients are also included.

This guide assumes you have setup your Ubiquity router with eth0 and eth1 as
different vlans using the WAN+2LAN2 wizard (using the EdgeRouter Lite v1.9.1).

## Install the PreReqs

The pre-reqs are openssl, openvpn, jq, j2cli, and cfssl.

* If on Linux, `make hostdeps`.
* Or if on OSX, you can probably install most of this with https://brew.sh/

## How to use it

* Edit the Makefile to set the quantities of each cert to generate
* Edit the existing ./config.yaml (linked to ./conf/config.yaml.example)
* Generate all templates and certificates with `make`.

## Ubiquiti EdgeOS OpenVPN Configuration

To configure the Router:

* customize the commands below
* ssh to the router
* paste the commands into the router

Please note that the current 1.9.1 version of Ubiquity EdgeOS, and thus
we are limited to non-current ciphers.

### Copy the certificates to the Router

```
# CUT AND PASTABLE

# Configure some vars
export USER=ubnt
export UBIQUITY=192.168.1.1

# Constants
export CA_DIR=ca
export CERTS_DIR=certs
export MACHINE="${USER}@${UBIQUITY}"
export KEY_DIR=/config/auth/keys

# Copy the Certificates to the router
ssh $MACHINE mkdir -p $KEY_DIR
scp $CA_DIR/ca.pem $MACHINE:$KEY_DIR/ca.crt
scp $CERTS_DIR/server01-key.pem $MACHINE:$KEY_DIR/server.key
scp $CERTS_DIR/server01.pem $MACHINE:$KEY_DIR/server.crt
scp $CA_DIR/dh4096.pem $MACHINE:$KEY_DIR/dh4096.pem
scp $CA_DIR/ta.key $MACHINE:$KEY_DIR/ta.key
```

### Configure OpenVPN on Ubiquiti

The nicely documented settings have been taken from
[GainfulShrimp](https://community.ubnt.com/t5/EdgeMAX/Secure-OpenVPN-server-setup-with-multi-factor-authentication/td-p/1240405) and ccustomized.

The following configs assume your Ubiquity has the following networks:

* 192.168.1.0 255.255.255.0
* 192.168.2.0 255.255.255.0

You may have to modify code and instructions to be specific to your setup.

SSH to the router first, then paste these commands

```

# Start the configuration shell
configure

# The EdgeOS UI listens on port 443 TCP by default, which would clash with our
# OpenVPN server if it also tried to listen on port 443 using TCP, so we'll
# listen on port 1194 and use port-forwarding later so that we can reach our
# server from the outside
set interfaces openvpn vtun0 openvpn-option "--port 1194"

# By default, OpenVPN is peer-to-peer, but we want to put our instance in
# 'multi-user server' mode
set interfaces openvpn vtun0 mode server

# Enable TLS and assume server role during TLS handshake
set interfaces openvpn vtun0 openvpn-option --tls-server

# This line enables adaptive fast LZO compression. You must enable it on both
# server and client(s).
set interfaces openvpn vtun0 openvpn-option --comp-lzo

# The following line instructs OpenVPN to drop root privileges following
# initialisation - this is an important option for security
set interfaces openvpn vtun0 openvpn-option '--user nobody --group nogroup'

# We need to tell OpenVPN to remember the key after first reading it. Normally
# if you drop root privileges in OpenVPN, the daemon cannot be restarted (e.g.
# on SIGUSR1 signal), since it will now be unable to re-read protected key
# files.
# This option solves the problem by persisting keys across SIGUSR1 resets, so
# they don't need to be re-read.
set interfaces openvpn vtun0 openvpn-option --persist-key

# Keep the tunnel up across SIGUSR1 restarts. Also keep the same IP addresses.
set interfaces openvpn vtun0 openvpn-option --persist-tun
set interfaces openvpn vtun0 openvpn-option --persist-local-ip
set interfaces openvpn vtun0 openvpn-option --persist-remote-ip

# Check that the client is still reachable, every 8 seconds (unless a packet
# has been received from the client in the meantime). If no pings received from
# the client for 60 seconds, assume that the connection needs restarting.
set interfaces openvpn vtun0 openvpn-option '--keepalive 8 60'

# Set the verbosity level for the logs to a medium-low level. If you're having
# trouble getting things working, you will want to set this to a value between
# 6-11, where higher numbers give more debugging information
set interfaces openvpn vtun0 openvpn-option '--verb 4'

# We trust all of our clients, so will allow them to communicate with each
# other while connected via the VPN
set interfaces openvpn vtun0 openvpn-option --client-to-client

# Write a file with details of clients and the virtual IP address assigned to
# them, so that they hopefully always get the same address (this helps clients
# using persist-tun option)
set interfaces openvpn vtun0 openvpn-option '--ifconfig-pool-persist /config/auth/openvpn/vtun0-ipp.txt'

# The following lines instruct clients to direct all of their traffic (as much
# as possible) via the VPN. You should replace 192.168.2.1 with the address of
# your Edgerouter on your LAN (assuming that your Edgerouter is providing DNS
# for your LAN, that is).
# set interfaces openvpn vtun0 openvpn-option '--push redirect-gateway def1'
set interfaces openvpn vtun0 openvpn-option "--push dhcp-option DNS 192.168.1.1"

# If you just want to make your LAN devices accessible from wherever you are
# but don't want to route internet traffic via your VPN tunnel, then leave out
# the following two lines and instead use something like (but replace the
# subnet and mask with your actual subnet and mask):
#set interfaces openvpn vtun0 openvpn-option "--push route 192.168.2.0 255.255.255.0"
set interfaces openvpn vtun0 openvpn-option "--push route 192.168.1.0 255.255.255.0"
set interfaces openvpn vtun0 openvpn-option "--push route 192.168.2.0 255.255.255.0"

# Enable HMAC authentication of the TLS control channel, using the key we
# generated earlier. The zero at the end is important, because it specifies the
# direction of the negotiation.
# With the following option enabled (and the corresponding option enabled on
# the clients), unauthorised clients are dropped before they can even attempt a
# TLS handshake.
set interfaces openvpn vtun0 openvpn-option '--tls-auth /config/auth/keys/ta.key 0'

# Enable our 'openvpn' PAM config for authentication of users using either TOTP
# or both password+TOTP (according to how you set up the PAM config earlier),
# on top of the standard PKI-based authentication:
# set interfaces openvpn vtun0 openvpn-option '--plugin /usr/lib/openvpn/openvpn-auth-pam.so openvpn'
# set interfaces openvpn vtun0 openvpn-option "--client-cert-not-required --username-as-common-name"

# Use 256bit AES-CBC cipher for main encryption. AES ciphers are considered
# strong and they are also well suited to ARM-powered clients such as iPhones.
# You can amend this to AES-128-CBC to trade off some security for slightly
# increased performance
set interfaces openvpn vtun0 openvpn-option '--cipher AES-256-CBC'

# All the advice online seems to recommend using the snappily-named cipher
# suite called TLS-DHE-RSA-WITH-AES-256-CBC-SHA for the control channel.
# However, if you use that name in the following line, you'll receive an error.
# That's because our OpenVPN server uses OpenSSL, which uses a different name
# for the same cipher suite:
set interfaces openvpn vtun0 openvpn-option '--tls-cipher DHE-RSA-AES256-SHA'

# The float option means that our clients can remain connected even if they
# move IP addresses after they first authenticate (so long as the packets still
# pass verification etc, of course). We want this because our clients might
# switch from wifi to cellular networks and back again, and we want to maximise
# the chances of our tunnel remaining usable during such events.
set interfaces openvpn vtun0 openvpn-option --float

# Use TCP protocol for the tunnel (for compatibility with restrictive firewalls
# remember), and passively accept connections from clients
set interfaces openvpn vtun0 openvpn-option "--proto tcp"
# set interfaces openvpn vtun0 protocol tcp-passive

# This is a virtual subnet used just for OpenVPN clients. I've chosen a
# relatively obscure subnet here, to minimise the chance of it clashing with
# whatever local subnet the client is connected to at the time.
set interfaces openvpn vtun0 server subnet 192.168.100.0/24

# Use the certificates (public keys) and private key that we generated earlier.
# If you chose to use 1024 bit strength when editing the 'vars' file, you'll
# need to specify "dh1024.pem" for the Diffie Hellman parameter file in the
# third line below:
set interfaces openvpn vtun0 tls ca-cert-file /config/auth/keys/ca.crt
set interfaces openvpn vtun0 tls cert-file /config/auth/keys/server.crt
set interfaces openvpn vtun0 tls key-file /config/auth/keys/server.key
set interfaces openvpn vtun0 tls dh-file /config/auth/keys/dh4096.pem

# Save the configuration
commit
save

```

### Configure Port Forwarding on Ubiquiti

```
#####################################################################
# Port Forwarding

configure

set port-forward wan-interface eth0
set port-forward lan-interface eth1

# Rule numbers are arbitrary
set port-forward rule 50 description 'OpenVPN TCP'
set port-forward rule 50 forward-to address 192.168.1.1
set port-forward rule 50 forward-to port 1194
set port-forward rule 50 original-port 1194
set port-forward rule 50 protocol tcp

# Skip the following for UDP which we are not enabling
#set port-forward rule 51 description 'OpenVPN UDP'
#set port-forward rule 51 forward-to address 192.168.1.1
#set port-forward rule 51 forward-to port 1194
#set port-forward rule 51 original-port 1194
#set port-forward rule 51 protocol udp

# Only enable listening on vtun0
set service dns forwarding listen-on vtun0
# Udp would have been vtun1, but we are not setting that up
#set service dns forwarding listen-on vtun1

set firewall name WAN_LOCAL rule 50 action accept
set firewall name WAN_LOCAL rule 50 description 'Allow OpenVPN'
set firewall name WAN_LOCAL rule 50 destination port 1194
set firewall name WAN_LOCAL rule 50 log disable
set firewall name WAN_LOCAL rule 50 protocol tcp
#set firewall name WAN_LOCAL rule 50 protocol tcp_udp

commit
save

```

## Configuring Clients

### Installing on a Mac Client

* brew install Caskroom/cask/tunnelblick
* Copy a clientXX.ovpn file to the machine
* Double click on it
* Start the VPN connection from TunnelBlick

### Installing on an iOS Client

* Download the app "OpenVPN Connect"
* Connect your phone to your computer
* Open iTunes
* Select the phone view
* Find the OpenVPN Connect App
* Drag one of the clientXX.ovpn files onto the OpenVPN Connect File Area
* Go to iPhone Settings -> OpenVPN
* Enable "Force AES-CBC ciphersuites"
* Start the OpenVPN Connect app
* Click on the newly recognized profile to install it.
* From there on, you can start VPN from the OpenVPN connect app or from iOS settings

## Relevant Documentation

OpenVpn Documentation

* https://openvpn.net/index.php/open-source/documentation/howto.html

OpenVpn on Ubiquiti

* https://community.ubnt.com/t5/EdgeMAX/Secure-OpenVPN-server-setup-with-multi-factor-authentication/td-p/1240405

Key Usage / Purposes

* https://tools.ietf.org/html/rfc5280#section-4.2.1.3

Cfssl Docs

* https://github.com/cloudflare/cfssl/blob/master/doc/cmd/cfssl.txt#L72

OpenVpn Hardening Guide

* https://community.openvpn.net/openvpn/wiki/Hardening