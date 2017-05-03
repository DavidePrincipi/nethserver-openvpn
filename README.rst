========
OpenVPN
========

Supported VPN are:

* Client to server (roadwarrior)
* Network to network (net2net)

Certificates
============

All certificates are signed using NethServer default RSA key (``/etc/pki/tls/private/NSRV.key``).

CA environment
--------------

CA configuration is stored inside ``/var/lib/nethserver/`` directory, all certificates are stored inside ``/var/lib/nethserver/certs``. The ``nethserver-openvpn-conf`` action creates:

* ``serial``, ``certindex.attr`` and ``/certindex``: database of valid and revocated certificates
* ``crlnumber`` and ``/etc/openvpn/certs/crl.pem``: certificate revocation list
* ``dh1024.pem``: key for TLS negotation


Certificate creation
--------------------

Certificates in PEM format can be created using the command: ::

  /usr/libexec/nethserver/pki-vpn-gencert <commonName>

The ``commonName`` parameter is an unique name stored inside the certificate. 
The command will generate ``key``, ``crt`` and ``csr`` file.
Each generated certificate is referred with a numeric id and saved inside ``certindex`` database. OpenSSL will also create a certificated called as with the generated id (eg. ``02.pem``). 

Certificate revocation
----------------------

Certificate revocation is done via the command: ::

    /usr/libexec/nethserver/pki-vpn-revoke [-d] <commonName>

The ``commonName`` parameter is an unique name stored inside the certificate. 
If '-d' option is enabled, also delete ``crt``, ``csr``, ``pem`` and key files

Certificate renewal
-------------------

All certificates will expire after ``X`` days, where ``X`` is the value of ``CertificateDuration`` property inside ``pki`` key.
Renew is done via the command: ::

  /usr/libexec/nethserver/pki-vpn-renew <commonName>

The ``commonName`` parameter is an unique name stored inside the certificate. 

Events
======

* ``nethserver-vpn-create``: fired when a new account is created, takes the account name as argument
* ``nethserver-vpn-delete``: fired when a new account is deleted, takes the account name as argument
* ``nethserver-vpn-modify``: fired when a new account is modified, takes the account name as argument
* ``nethserver-vpn-save``: fired when a +client+ is created, modified or deleted


Accounts
========

Accounts are used to identify clients connecting to the server itself. There are two types of account:

* user account: system user with VPN access using username and password
* vpn-only account: simple account with only VPN access

Each account can be used in a roadwarrior connection (host to net). 
If a net to net tunnel is needed, ``VPNRemoteNetwork`` and ``VPNRemoteNetmask`` 
properties must be set to inform the server about remote routes.
When a new account is created, a certificate with same name is generated inside ``/var/lib/nethserver/certs`` directory.

Properties:

* ``VPNRemoteNetwork``: remote network
* ``VPNRemoteNetmask``: remote netmask
* ``OpenVpnIp``: reserved IP for the client

Database reference
------------------

Database: ``vpn``

::

 <name>=vpn
    VPNRemoteNetwork=
    VPNRemoteNetmask=
    OpenVpnIp=

 <name>=vpn-user
    ...
    VPNRemoteNetwork=
    VPNRemoteNetmask=
    OpenVpnIp=

Clients
=======

OpenVPN clients are used to connect the server with other network, typically to create a net2net tunnel. 

Common properties:

* ``AuthMode``: default value is ``certificate``. Possible values:

  * ``certificate``: use x509 certificate. Certificates, including CA and private key, are saved in ``/var/lib/nethserver/certs/clients`` directory in a PEM file named ``key``.pem
  * ``password``: use username and password
  * ``password-certificate``: use username, password and a valid x509 certificate
  * ``psk``: use a pre-shared key
* ``User``: username used for authentication, if ``AuthMode`` is ``password`` or ``password-certificate``
* ``Password``: password used for authentication, if ``AuthMode`` is ``password`` or ``password-certificate``
* ``Psk``: pre-shared key user for authentication, if ``AuthMode`` is ``psk``. Pre-shared key is saved ``/var/lib/nethserver/certs/clients/<name>.key`` 
* ``RemoteHost``: remote server hostname or ip address
* ``RemotePort``: remote host port
* ``VPNRemoteNetwork``: remote network (behind remote firewall), used for a net to net tunnel
* ``VPNRemoteNetmask``: remote netmask, used for a net to net tunnel
* ``Compression``: can be ``enabled`` or ``disabled``, default is ``enabled``. Enable/disable adaptive LZO compression.


Database reference
------------------

Database: ``vpn``

::

 t1=tunnel
    Mode=routed
    AuthMode=certificate
    Crt=
    Psk=
    Password=
    RemoteHost=1.2.3.4
    RemotePort=1234
    User=
    VPNRemoteNetmask=255.255.255.0
    VPNRemoteNetwork=192.168.4.0
    Compression=enabled


Roadwarrior server
==================

Client configuration is generated using ``/usr/libexec/nethserver/openvpn-local-client`` command. 
The file will contain the CA certificate inside the <ca>.

Example: ::

  /usr/libexec/nethserver/openvpn-local-client myuser

The OpenVPN server listens on a management socket: ``/var/spool/openvpn/host-to-net``.
It's possible to retrieve server status and execute commands using the socket.

Available scripts:

* ``/usr/libexec/nethserver/openvpn-status``: retrieve status of connected clients and return result in JSON format
* ``/usr/libexec/nethserver/openvpn-kill``: kill a connected client, exits 0 on success, 1 otherwise

Example with netcat: ::

  >INFO:OpenVPN Management Interface Version 1 -- type 'help' for more info
  status
  OpenVPN CLIENT LIST
  Updated,Thu Jan 23 09:22:24 2014
  Common Name,Real Address,Bytes Received,Bytes Sent,Connected Since
  ROUTING TABLE
  Virtual Address,Common Name,Real Address,Last Ref
  GLOBAL STATS
  Max bcast/mcast queue length,0
  END

See more on management option: http://openvpn.net/index.php/open-source/documentation/miscellaneous/79-management-interface.html

Log files
---------

Host to net status: ``/var/log/openvpn/host-to-net-status.log``.
Server and client output: ``/var/log/messages``.

Configuration database
----------------------

Properties:

* ``status``: enable or disabled OpenVPN server _and_ cleints,  can be ``enabled`` or ``disabled``, default is ``enabled``
* ``ServerStatus``: enable or disabled the OpenVPN server, can be ``enabled`` or ``disabled``, default is ``disabled``
* ``AuthMode``: authentication mode, can be ``password``, ``certificate`` or ``password-certificate``. Default is ``password``
* ``UDPPort``: server listen port, default is ``1194``
* ``Mode``: network mode, can be ``routed`` or ``bridged``. Default is ``routed``.
* ``ClientToClient``: can be ``enabled`` or ``disabled``, default is ``disabled``. When enabled, traffic between VPN clients is allowed
* ``Compression``: can be ``enabled`` or ``disabled``, default is ``disabled``. When enabled, adaptive LZO compression is used
* ``Remote``: comma-separated list of IPs or host names, it's used as multiple *remote* option inside client configuration generation script

If mode is ``bridged``:

* ``BridgeEndIP``: first client IP pool, must be inside the LAN range and outside DHCP range
* ``BridgeStartIP``: last client IP pool, must be inside the LAN range and outside DHCP range
* ``BridgeName``: name of the bridge, default is ``br0``
* ``TapInterface``: name of bridged tap interface, default is ``tap0``

If mode is ``routed``:

* ``Network``: network of VPN clients, eg. 192.168.6.0
* ``Netmask``: netmask of VPN clients, eg. 255.255.255.0
* ``RouteToVPN``: can be ``enabled`` or ``disabled``, default is ``disabled``. When enabled, all traffic from client will be routed via VPN tunnel


Reference
^^^^^^^^^

Example: ::

 openvpn=service
    ServerStatus=enabled
    AuthMode=password
    BridgeEndIP=192.168.1.122
    BridgeName=br0
    BridgeStartIP=192.168.1.121
    ClientToClient=disabled
    Mode=routed
    Netmask=255.255.255.0
    Network=192.168.6.0
    RouteToVPN=disabled
    TapInterfaces=tap0
    UDPPort=1194
    access=green,red
    status=enabled

