===============
Architecture
===============

.. image:: images/nm-diagram-2.jpg
   :width: 45%
   :alt: Netmaker Architecture Diagram
   :align: center
    

*Pictured Above: A diagram of Netmaker's Architecture.*


Core Concepts
==============

Familiarity with several core concepts will help when you encounter them later on in the documentation.

WireGuard
----------

WireGuard is a relatively new but very important technology which was recently added to the Linux kernel. WireGuard creates very fast but simple encrypted tunnels between devices. From the `WireGuard <https://www.wireguard.com/>`_ website, "it might be regarded as the most secure, easiest to use, and simplest VPN solution in the industry."

Previous solutions like OpenVPN and IPSec are considerably more heavy and complex, while being less performant. All existing VPN tunnelling solutions will cause a significant increase in your network latency. WireGuard is the first to achieve near over-the-line network speeds, meaning you see no signigifant performance impact.  With the release of WireGuard, there is little reason to use any other existing tunnel encryption technology.

Mesh Network
-------------

When we refer to a mesh network in these documents we are typically referring to a "full mesh."

.. image:: images/mesh.png
   :width: 33%
   :alt: Full Mesh Network Diagram
   :align: center


A full `mesh network <https://www.bbc.co.uk/bitesize/guides/zr3yb82/revision/2>`_ exists where each machine is able to directly talk to every other machine on the network. For example, on your home network, behind your router, all the computers are likely given private addresses and can reach each other directly.

This is in contrast to a hub-and-spoke network, where each machine must first pass its traffic through a relay server before it can reach other machines.

In certain situations you may either want or need a *partial mesh* network, where only some devices can reach each other directly, and other devices must route their traffic through a relay/gateway. Netmaker can use this model in some use cases where it makes sense. In the diagram at the top of this page, the setup is a partial mesh, because the servers (nodes A-D) are meshed, but then external clients come in via a gateway, and are not meshed.

Mesh networks are generally faster than other topologies, but are also more complicated to set up. WireGuard on its own gives you the means to create encrypted tunnels between devices, but it does not provide a method for setting up a full network. This is where Netmaker comes in.

Netmaker
---------

Netmaker is a platform built off of WireGuard which enables users to create mesh networks between their devices. Netmaker can create both full and partial mesh networks depending on the use case.

When we refer to Netmaker in aggregate, we are typically referring to Netmaker and the netclient, as well as other supporting services such as CoreDNS, MongoDB, and UI webserver.

From an end user perspective, they typically interact with the Netmaker UI, or even just run the install script for the netclient on their devices. The other components run in the background invisibly. 

Netmaker does a lot of work to set configurations for you, so that you don't have to. This includes things like WireGuard ports, endpoints, public IPs, keys, and peers. Netmaker works to abstract away as much of the network management as possible, so that you can just click to create a network, and click to add a machine to a network. That said, every machine (node) is different, and may require special configuration. That is why, while Netmaker sets practical default settings, everything within Netmaker is fully configurable.

Node
------

A machine in a Netmaker network, which is managed by the Netclient, is referred to as a Node, as you will see in the UI. A Node can be a VM, a bare metal server, a desktop computer, an IoT device, or any other number of internet-connected machines on which the netclient is installed. A node is simply an endpoint in the network, which can send traffic to all the other nodes, and recieve traffic from all of the other nodes.

SystemD
-------

SystemD is a system service manager for a wide array of Linux operating systems. Not all Linux distributions have adopted systemd, but, for better or worse, it has become a fairly common standard in the Linux world. That said, any non-Linux operating system will not have systemd, and many Linux/Unix distributionshave alternative system service managers.

Netmaker's netclient, the agent which controls networking on all nodes, relies heavily on systemd as of version 0.3. This reliance is being reduced but is currently a core dependency, causing most of the limitations and incompatibilities. As Netmaker evolves, systemd will become just one of the possible service management options, allowing the netclient to be run on a wider array of devices.


Components
===========

Netmaker consists of several core components, which are explained in high-level technical detail below.

Netmaker Server
------------------

The Netmaker server is, at its core, a golang binary. Source code can be found `on GitHub <https://github.com/gravitl/netmaker>`_. The binary, by itself can be compiled for most systems. If you need to run the Netmaker server on a particular system, it likely can be made to work. In typical deployments, it is run as a Docker container. It can also be run as a systemd service as outlined in the non-docker install guide.

The Netmaker server acts as an API to the front end, and as a GRPC server to the machines in the network. GRPC is much faster and more efficient than standard API calls, which increases the speed of transactions. For this reason, the Netmaker server exposes two ports: The default for the API is 8081, and the default for GRPC is 50051. Either the API or the GRPC server can be disabled on any given Netmaker instance can be disabled, allowing you to deploy two different servers for managing the API (which is largely for the admin's use) and GRPC (which is largely for the nodes' use).

Most server settings are configurable via a config file, or by environment variables (which take precedence). If the server finds neither of these, it sets sensible defaults, including things like the server's reachable IP, ports, and which "modes" to run in.

These modes include client mode and dns mode. Either of these can be disabled but are enabled by default. Client mode allows you to treat the Netmaker host machine (operating system) as a network Node, installing the netclient and controlling the host network. DNS mode has the server write config settings for CoreDNS, a separate component and nameserver, which picks up the config settings to manage node DNS.

The Netmaker server interacts with (as of v0.3) a MongoDB instance, which holds information about nodes, networks, users, and other important data. This data is configuration data. For the most part, Netmaker serves configuration data to Nodes, telling them how they should configure themselves. The Netclient is the agent that actually does that configuration.


Netclient
----------------

The netclient is, at its core, a golang binary. Source code can be found in the netclient folder of the Netmaker `GitHub Repository <https://github.com/gravitl/netmaker/tree/master/netclient>`_. The binary, by itself, can be compiled for most systems. However, this binary is designed to manage a certain number of Operating Systems. As of version 0.3, it requires systemd in order to manage the host system appropriately. It may be installable, and it may even make the machine a part of the mesh network, but it will not function in entirely (see Compatible Systems for more info) without systemd.

The netclient is installed via a simple bash script, which pulls the latest binary and runs install command.

The install command registers the machine with the Netmaker server using sensible defaults, which can be overridden with a config file or environment variables. Assuming the netclient has a valid key (or the network allows manual node signup), it will be registered in the Netmaker network, which will return configuration details about how to set up the local network. 

The netclient then sets itself up in systemd, and configures WireGuard. At this point it should be part of the network.

On a periodic basis (systemd timer), the netclient performs a "check in." It will authenticate with the server, and check to see if anything has changed in the network. It will also post changes about its own local configuration if there. If there has been a change, the server will return new configurations and the netclient will reconfigure the network.

The check in process is what allows Netmaker to create dynamic mesh networks. As nodes are added to, removed from, and modified on the network, other nodes are notified, and make appropriate changes.


MongoDB
--------

As of v0.3, Netmaker uses MongoDB as its database, and interacts with a MongoDB instance to store and retrieve information about nodes, networks, and users. Netmaker is rapidly evolving, and MongoDB provides a flexible database structure that accelerates development. However, MongoDB is also the heaviest component of Netmaker (high cpu/memory consumption), and is set to be replaced by a lighter-weight, SQL-based database in the future.

Netmaker UI
---------------

The Netmaker UI is a ReactJS-based static website which can be run on top of standard webservers such as Apache and Nginx. Source code can be found `here <https://github.com/gravitl/netmaker-ui>`_. In a typical configuration, the Netmaker UI is run on Nginx as a Docker container.

Netmaker can be used in its entirety without the UI, but the UI makes things a lot easier for most users. It has a sensible flow and layout for managing Networks, Nodes, Access Keys, and DNS.


CoreDNS
--------

v0.3 introduced the concept of private DNS management for nodes. This requires a nameserver, and CoreDNS is the chosen nameserver. CoreDNS is lightweight and extensible. CoreDNS loads dns settings from a simple file, managed by Netmaker, and serves out DNS info for managed nodes. DNS can be tricky, and DNS management is currently only supported on a small set of devices, specifically those running systemd-resolved. However, the Netmaker CoreDNS instance can be added manually as a nameserver to other devices. DNS mode can also be turned off.

Worth considering is that CoreDNS requires port 53 on the Netmaker host system, which may cause conflicts depending on your operating system. This is explained in the :doc:`Server Installation <./server-installation>` guide.


Technical Process
====================

Below is a high level, step-by-step overview of the flow of communications within Netmaker (assuming Netmaker has already been installed):

1. Admin creates a new network with a subnet, for instance 10.10.10.0/24
2. Admin creates an access key for signing up new nodes
3. Both of the above requests are routed to the server via an API call from the front end
4. Admin runs the netclient install script on any given node (machine).
5. Netclient decodes key, which contains the GRPC server location and port
6. Netclient retrieves/sets local information, including open ports for WireGuard, public IP, and generating key pairs for peers
7. Netclient reaches out to GRPC server with this information, authenticating via access key.
8. Netmaker server verifies information and creates the node, setting default values for any missing information. 
9. Timestamp is set for the network (see #16). 
10. Netmaker returns settings as response to netclient. Some settings may be added or modified based on the network.
11. Netclient recieves response. If successful, it takes any additional info returned from Netmaker and configures the local system/WireGuard
12. Netclient sends another request to Netmaker's GRPC server, this time to retrieve the peers list (all other clients in the network).
13. Netmaker sends back peers list, including current known configurations of all nodes in network.
14. Netclient configures WireGuard with this information. At this point, the node is fully configured as a part of the network and should be able to reach the other nodes via private address.
15. Netclient begins daemon (system timer) to run check in's with the server. It awaits changes, reporting local changes, and retrieving changes from any other nodes in the network.
16. Other netclients on the network, upon checking in with the Netmaker server, will see that the timestamp has updated, and they will retrieve a new peers list, completing the update cycle.


Compatible Systems for Netclient
==================================

To manage a node automatically, the Netmaker client (netclient) requires **systemd-based linux.** Compatible systems include:
        - Fedora
        - Ubuntu
        - Debian
        - Mint
        - SUSE
        - RHEL
        - Raspian.
        - Arch
        - CentOS
        - CoreOS
      
To manage DNS (optional), the node must have systemd-resolved. Systems that have this enabled include:
        - Arch
        - Debian
        - Ubuntu
        - SUSE

Limitations
===========

Install limitations mostly include platform-specific limitations, such as needing systemd or systemd-resolved (see above). In addition the Netmaker platform has some additional limitations:

- **Double NAT**: Netmaker is currently unable to route traffic for devices behind a "double NAT".
- **CGNAT**: Netmaker is currently unable to route traffic for for devices behind a "carrier-grade NAT".
- **Windows/iPhone/Android**: To reiterate the systemd limitation, Netmaker is not currently configured to support "end user" devices such as Windows desktops and phones generally. In v0.4, Netmaker will introduce external device gateways to allow this traffic (and many other sorts of devices).
