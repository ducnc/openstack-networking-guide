# Scenario: Legacy implementation with Open vSwitch

This scenario describes a legacy (basic) implementation of the
OpenStack Networking service using the ML2 plug-in with Open vSwitch.
The example configuration creates one flat external network and VXLAN
tenant networks. However, this configuration also supports VLAN
external networks, VLAN tenant networks, and GRE tenant networks.

## Requirements

1. One controller node with one network interface: management.

1. One network node with three network interfaces: management, instance
   tunnels, and external (typically the Internet). The Open vSwitch
   bridge `br-ex` must contain a port on the external interface.

1. At least one compute nodes with two network interfaces: management
   and instance tunnels.

![Neutron Legacy Scenario - Hardware Requirements](../common/images/networkguide-neutron-legacy-hw.png "Neutron Legacy Scenario - Hardware Requirements")

![Neutron Legacy Scenario - Network Layout](../common/images/networkguide-neutron-legacy-networks.png "Neutron Legacy Scenario - Network Layout")

**Warning: Proper operation of VLAN requires kernel 3.13 or newer. In
general, only Ubuntu 14.04, Fedora 20, and Fedora 21 meet or exceed this
minimum version requirement when using packages rather than source.

## Prerequisites

1. Controller node

  1. Operational SQL server with `neutron` database and appropriate
     configuration in the neutron-server.conf file.

  1. Operational message queue service with appropriate configuration
     in the neutron-server.conf file.

  1. Operational OpenStack Identity service with appropriate configuration
     in the neutron-server.conf file.

  1. Operational OpenStack Compute controller/management service with
     appropriate configuration to use neutron in the nova.conf file.

  1. Neutron server service, ML2 plug-in, and any dependencies.

1. Network node

  1. Operational OpenStack Identity service with appropriate configuration
     in the neutron-server.conf file.

  1. Open vSwitch service, ML2 plug-in, Open vSwitch agent, L3 agent,
     DHCP agent, metadata agent, and any dependencies including the
     `ipset` utility.

1. Compute nodes

  1. Operational OpenStack Identity service with appropriate configuration
     in the neutron-server.conf file.

  1. Operational OpenStack Compute controller/management service with
     appropriate configuration to use neutron in the nova.conf file.

  1. Open vSwitch service, ML2 plug-in, Open vSwitch agent, and any
     dependencies including the `ipset` utility.

![Neutron Legacy Scenario - Service Layout](../common/images/networkguide-neutron-legacy-services.png "Neutron Legacy Scenario - Service Layout")

## Architecture

### General

The legacy architecture provides basic virtual networking components in
your environment. Routing among tenant and external networks resides
completely on the network node. Although more simple to deploy than
other architectures, performing all functions on the network node
creates a single point of failure and potential performance issues.
Consider deploying DVR or L3 HA architectures in production environments
to provide redundancy and increase performance.

![Neutron Legacy Scenario - Architecture Overview](../common/images/networkguide-neutron-legacy-general.png "Neutron Legacy Scenario - Architecture Overview")

The network node runs the Open vSwitch agent, L3 agent, DHCP agent, and
metadata agent.

![Neutron Legacy Scenario - Network Node Overview](../common/images/networkguide-neutron-legacy-ovs-network1.png "Neutron Legacy Scenario - Network Node Overview")

The compute nodes run the Open vSwitch agent.

![Neutron Legacy Scenario - Compute Node Overview](../common/images/networkguide-neutron-legacy-ovs-compute1.png "Neutron Legacy Scenario - Compute Node Overview")

### Components

The network node contains the following components:

1. Open vSwitch agent managing virtual switches, connectivity among
   them, and interaction via virtual ports with other network components
   such as namespaces, Linux bridges, and underlying interfaces.

1. DHCP agent managing the `qdhcp` namespaces.

  1. The `dhcp` namespaces provide DHCP services for instances using 
     tenant networks.

1. L3 agent managing the `qrouter` namespaces.

  1. The `qrouter` namespaces provide routing between tenant and external
     networks and among tenant networks. They also route metadata traffic
     between instances and the metadata agent.

1. Metadata agent handling metadata operations.

  1. The metadata agent handles metadata operations for instances.

![Neutron Legacy Scenario - Network Node Components](../common/images/networkguide-neutron-legacy-ovs-network2.png "Neutron Legacy Scenario - Network Node Components")

The compute nodes contain the following components:

1. Open vSwitch agent managing virtual switches, connectivity among
   them, and interaction via virtual ports with other network components
   such as namespaces, Linux bridges, and underlying interfaces.

1. Linux bridges handling security groups.

  1. Due to limitations with Open vSwitch and *iptables*, the Networking
     service uses a Linux bridge to manage security groups for
     instances.

![Neutron Legacy Scenario - Compute Node Components](../common/images/networkguide-neutron-legacy-ovs-compute2.png "Neutron Legacy Scenario - Compute Node Components")

## Packet flow

### Case 1: North-south for instances without a floating IP address

For instances without a floating IP address, the network node routes
*north-south* network traffic between tenant and external networks.

Note: The term *north-south* generally defines network traffic that
travels between tenant and external networks (typically the Internet).

#### Example environment configuration

Instance 1 resides on compute node 1 and uses tenant network 1.
The instance sends a packet to a host on the external network.

* External network 1

  * Network 203.0.113.0/24

  * Gateway 203.0.113.1 with MAC address *EG1*

  * Floating IP range 203.0.113.101 to 203.0.113.200

  * Tenant network 1 router interface 203.0.113.101 *TR1*

* Tenant network 1

  * Network 192.168.1.0/24

  * Gateway 192.168.1.1 with MAC address *TG1*

* Compute node 1

  * Instance 1 192.168.1.11 with MAC address *I1*

#### Packet flow

The following steps involve compute node 1. *FIXME*

1. The instance 1 `tap` interface (1) forwards the packet to the Linux
   bridge `qbr`. The packet contains destination MAC address *TG1*
   because the destination resides on another network.

1. Security group rules (2) on the Linux bridge `qbr` handle state tracking
   for the packet.

1. The Linux bridge `qbr` forwards the packet to the Open vSwitch
   integration bridge `br-int`.

1. The Open vSwitch integration bridge `br-int` modifies the packet to
   contain the internal tag for tenant network 1.

1. The Open vSwitch integration bridge `br-int` forwards the packet to
   the Open vSwitch tunnel bridge `br-tun`.

1. The Open vSwitch tunnel bridge `br-tun` replaces the packet source
   MAC address *I1* with *D1*.

1. The Open vSwitch tunnel bridge `br-tun` wraps the packet in a VXLAN
   or GRE tunnel that contains a tag for tenant network 1.

1. The Open vSwitch tunnel bridge `br-tun` forwards the packet to the
   network node via the tunnel interface.

The following steps involve the network node.

1. The tunnel interface forwards the packet to the Open vSwitch tunnel
   bridge `br-tun`.

1. The Open vSwitch tunnel bridge `br-tun` unwraps the packet and adds
   the internal tag for tenant network 1.

1. The Open vSwitch tunnel bridge `br-tun` forwards the packet to the
   Open vSwitch integration bridge `br-int`.

1. The Open vSwitch integration bridge `br-int` replaces the packet
   source MAC address *D1* with *TG1*.

1. The Open vSwitch integration bridge `br-int` forwards the packet to
   the `sg` interface (3) in the SNAT namespace `snat`. The `sg`
   interface contains the tenant network gateway IP address *TG1*.

1. The *iptables* service (4) performs SNAT on the packet using the `qg`
   interface (5) as the source IP address. The `qg` interface contains
   the tenant network 1 router interface IP address *TR1*.

1. The `qg` interface forwards the packet to the Open vSwitch external
   bridge `br-ex`.

1. The Open vSwitch external bridge `br-ex` forwards the packet to the
   external network via the external interface.

Note: Return traffic follows similar steps in reverse.

![Neutron Legacy Scenario - Network Traffic Flow - North/South with Fixed IP Address](../common/images/networkguide-neutron-legacy-ovs-flowns1.png "Neutron Legacy Scenario - Network Traffic Flow - North/South with Fixed IP Address")

### Case 2: North-south for instances with a floating IP address

For instances with a floating IP address, the network node routes
*north-south* network traffic between tenant and external networks.

#### Example environment configuration

Instance 1 resides on compute node 1 and uses tenant network 1.
The instance sends a packet to a host on the external network.

* External network 1

  * Network 203.0.113.0/24

  * Gateway 203.0.113.1 with MAC address *EG1*

  * Floating IP range 203.0.113.101 to 203.0.113.200

  * Tenant network 1 router interface 203.0.113.101 *TR1*

* Tenant network 1

  * Network 192.168.1.0/24

  * Gateway 192.168.1.1 with MAC address *TG1*

* Compute node 1

  * Instance 1 192.168.1.11 with MAC address *I1* and floating
   IP address 203.0.113.102 *F1*

#### Packet flow

The following steps involve compute node 1. *FIXME*

1. The instance 1 `tap` interface (1) forwards the packet to the Linux
   bridge `qbr`. The packet contains destination MAC address *TG1*
   because the destination resides on another network.

1. Security group rules (2) on the Linux bridge `qbr` handle state tracking
   for the packet.

1. The Linux bridge `qbr` forwards the packet to the Open vSwitch
   integration bridge `br-int`.

1. The Open vSwitch integration bridge `br-int` modifies the packet to
   contain the internal tag for tenant network 1.

1. The Open vSwitch integration bridge `br-int` forwards the packet to
   the Open vSwitch tunnel bridge `br-tun`.

1. The Open vSwitch tunnel bridge `br-tun` replaces the packet source
   MAC address *I1* with *D1*.

1. The Open vSwitch tunnel bridge `br-tun` wraps the packet in a VXLAN
   or GRE tunnel that contains a tag for tenant network 1.

1. The Open vSwitch tunnel bridge `br-tun` forwards the packet to the
   network node via the tunnel interface.

The following steps involve the network node.

1. The tunnel interface forwards the packet to the Open vSwitch tunnel
   bridge `br-tun`.

1. The Open vSwitch tunnel bridge `br-tun` unwraps the packet and adds
   the internal tag for tenant network 1.

1. The Open vSwitch tunnel bridge `br-tun` forwards the packet to the
   Open vSwitch integration bridge `br-int`.

1. The Open vSwitch integration bridge `br-int` replaces the packet
   source MAC address *D1* with *TG1*.

1. The Open vSwitch integration bridge `br-int` forwards the packet to
   the `sg` interface (3) in the SNAT namespace `snat`. The `sg`
   interface contains the tenant network gateway IP address *TG1*.

1. The *iptables* service (4) performs SNAT on the packet using the `qg`
   interface (5) as the source IP address. The `qg` interface contains
   the tenant network 1 router interface IP address *TR1*.

1. The `qg` interface forwards the packet to the Open vSwitch external
   bridge `br-ex`.

1. The Open vSwitch external bridge `br-ex` forwards the packet to the
   external network via the external interface.

Note: Return traffic follows similar steps in reverse.

![Neutron Legacy Scenario - Network Traffic Flow - North/South with Floating IP Address](../common/images/networkguide-neutron-legacy-ovs-flowns2.png "Neutron Legacy Scenario - Network Traffic Flow - North/South with Floating IP Address")

### Case 3: East-west for instances with or without a floating IP address

For instances with or without a floating IP address, the network node
routes *east-west* network traffic among tenant networks using the
same router.

Note: The term *east-west* generally defines network traffic that
travels within a tenant network or between tenant networks.

#### Example environment configuration

Instance 1 resides on compute node 1 and uses tenant network 1. Instance
2 resides on compute node 2 and uses tenant network 2. Both tenant networks
reside on the same router. Instance 1 sends a packet to instance 2.

* Tenant network 1

  * Network: 192.168.1.0/24

  * Gateway: 192.168.1.1 with MAC address *TG1*

* Tenant network 2

  * Network: 192.168.2.0/24

  * Gateway: 192.168.2.1 with MAC address *TG2*

* Compute node 1

  * Instance 1: 192.168.1.11 with MAC address *I1*

* Compute node 2

  * Instance 2: 192.168.2.11 with MAC address *I2*

#### Packet flow

The following steps involve compute node 1: *FIXME*

1. The instance 1 `tap` interface (1) forwards the packet to the Linux
   bridge `qbr`. The packet contains destination MAC address *TG1*
   because the destination resides on another network.

1. Security group rules (2) on the Linux bridge `qbr` handle state tracking
   for the packet.

1. The Linux bridge `qbr` forwards the packet to the Open vSwitch
   integration bridge `br-int`.

1. The Open vSwitch integration bridge `br-int` forwards the packet to
   the tenant network 1 interface (3) in the distributed router namespace
   `qrouter`.

1. The distributed router namespace `qrouter` routes the packet to
   tenant network 2.

1. The tenant network 2 interface (4) in the distributed router namespace
   `qrouter` namespace forwards the packet to the Open vSwitch
   integration bridge `br-int`.

1. The Open vSwitch integration bridge `br-int` modifies the packet
   to contain the internal tag for tenant network 2.

1. The Open vSwitch integration bridge `br-int` forwards the packet to
   the Open vSwitch tunnel bridge `br-tun`.

1. The Open vSwitch tunnel bridge `br-tun` replaces the packet source
   MAC address *I1* with *D1*.

1. The Open vSwitch tunnel bridge `br-tun` wraps the packet in a VXLAN
   or GRE tunnel that contains a tag for tenant network 2.

1. The Open vSwitch tunnel bridge `br-tun` forwards the packet to
   compute node 2 via the tunnel interface.

The following steps involve compute node 2.

1. The tunnel interface forwards the packet to the Open vSwitch tunnel
   bridge `br-tun`.

1. The Open vSwitch tunnel bridge `br-tun` unwraps the packet.

1. The Open vSwitch tunnel bridge `br-tun` forwards the packet to the
   Open vSwitch integration bridge `br-int`.

1. The Open vSwitch integration bridge `br-int` replaces the packet
   source MAC address *D1* with *TG2*.

1. The Open vSwitch integration bridge `br-int` forwards the packet to
   the Linux bridge `qbr`.

1. Security group rules (7) on the Linux bridge `qbr` handle firewalling
   and state tracking for the packet.

1. The Linux bridge `qbr` forwards the packet to the instance 2 `tap`
   interface (8).

Note: Packets arriving from compute node 1 do not traverse the tenant
network interfaces (5,6) in the `qrouter` namespace on compute node 2.
However, return traffic traverses them.

![Neutron Legacy Scenario - Network Traffic Flow - East/West](../common/images/networkguide-neutron-legacy-ovs-flowew1.png "Neutron Legacy Scenario - Network Traffic Flow - East/West")

## Configuration

### Controller node (controller)

The controller node provides the neutron API and manages services on the
other nodes.

1. Configure base options.

   1. Edit the /etc/neutron/neutron.conf file.

    ```
    [DEFAULT]
    verbose = True
    core_plugin = ml2
    service_plugins = router
    allow_overlapping_ips = True

    notify_nova_on_port_status_changes = True
    notify_nova_on_port_data_changes = True
    nova_url = http://controller:8774/v2
    nova_region_name = regionOne
    nova_admin_username = NOVA_ADMIN_USERNAME
    nova_admin_tenant_id = NOVA_ADMIN_TENANT_ID
    nova_admin_password =  NOVA_ADMIN_PASSWORD
    nova_admin_auth_url = http://controller:35357/v2.0
    ```

  Note: Replace NOVA_ADMIN_USERNAME, NOVA_ADMIN_TENANT_ID, and
  NOVA_ADMIN_PASSWORD with suitable values for your environment.

1. Configure the ML2 plug-in.

  1. Edit the /etc/neutron/plugins/ml2/ml2_conf.ini file.

    ```
    [ml2]
    type_drivers = flat,vlan,gre,vxlan
    tenant_network_types = vlan,vxlan,gre
    mechanism_drivers = openvswitch,l2population

    [ml2_type_vlan]
    network_vlan_ranges = external:1:1000

    [ml2_type_gre]
    tunnel_id_ranges = 1:1000

    [ml2_type_vxlan]
    vni_ranges = 1:1000
    vxlan_group = 239.1.1.1

    [securitygroup]
    enable_security_group = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    enable_ipset = True
    ```

  Note: Adjust the VLAN, GRE tunnel, and VXLAN tunnel ID ranges for your
  environment.

1. Start the following services:

  * Server

### Network node (network1)

The network node provides DHCP and NAT services to all instances.

1. Configure base options.

  1. Edit the /etc/neutron/neutron.conf file.

    ```
    [DEFAULT]
    verbose = True
    core_plugin = ml2
    service_plugins = router
    allow_overlapping_ips = True
    ```

1. Configure the ML2 plug-in.

  1. Edit the /etc/neutron/plugins/ml2/ml2_conf.ini file.

    ```
    [ml2]
    type_drivers = flat,vlan,gre,vxlan
    tenant_network_types = vlan,vxlan,gre
    mechanism_drivers = openvswitch,l2population

    [ml2_type_flat]
    flat_networks = external

    [ml2_type_vlan]
    network_vlan_ranges = external:1:1000

    [ml2_type_gre]
    tunnel_id_ranges = 1:1000

    [ml2_type_vxlan]
    vni_ranges = 1:1000
    vxlan_group = 239.1.1.1

    [securitygroup]
    enable_security_group = True
    enable_ipset = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

    [ovs]
    local_ip = TENANT_NETWORK_TUNNELS_INTERFACE_IP_ADDRESS
    enable_tunneling = True
    bridge_mappings = external:br-ex

    [agent]
    l2population = True
    tunnel_types = gre,vxlan
    ```

  Note: Replace TENANT_NETWORK_TUNNELS_INTERFACE_IP_ADDRESS with the IP
  address of the tenant network tunnels interface.

  Note: Adjust the VLAN, GRE tunnel, and VXLAN tunnel ID ranges for your
  environment.

1. Configure the L3 agent.

  1. Edit the /etc/neutron/l3_agent.ini file.

    ```
    [DEFAULT]
    verbose = True
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    use_namespaces = True
    external_network_bridge = br-ex
    router_delete_namespaces = True
    ```

1. Configure the DHCP agent.

  1. Edit the /etc/neutron/dhcp_agent.ini file.

    ```
    verbose = True
    interface_driver = neutron.agent.linux.interface.OVSInterfaceDriver
    dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
    use_namespaces = True
    dhcp_delete_namespaces = True
    ```

  1. (Optional) Reduce MTU for VXLAN tenant networks.

    1. Edit the /etc/neutron/dhcp_agent.ini file.

    ```
    dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
    ```

    1. Edit the /etc/neutron/dnsmasq-neutron.conf file.

    ```
    dhcp-option-force=26,1450
    ```

1. Configure the metadata agent.

  1. Edit the /etc/neutron/metadata_agent.ini file.

    ```
    [DEFAULT]
    verbose = True
    auth_url = http://controller:5000/v2.0
    auth_region = regionOne
    admin_tenant_name = ADMIN_TENANT_NAME
    admin_user = ADMIN_USER
    admin_password = ADMIN_PASSWORD
    nova_metadata_ip = controller
    metadata_proxy_shared_secret = METADATA_SECRET
    ```

  Note: Replace ADMIN_TENANT_NAME, ADMIN_USER, ADMIN_PASSWORD, and
  METADATA_SECRET with suitable values for your environment.

1. Start the following services:

  * Open vSwitch
  * Open vSwitch agent
  * L3 agent
  * DHCP agent
  * Metadata agent

### Compute nodes (compute1 and compute2)

The compute nodes provide switching services and handle security groups
for instances.

1. Configure base options.

  1. Edit the /etc/neutron/neutron.conf file.

    ```
    [DEFAULT]
    verbose = True
    core_plugin = ml2
    service_plugins = router
    allow_overlapping_ips = True
    ```

1. Configure the ML2 plug-in.

  1. Edit the /etc/neutron/plugins/ml2/ml2_conf.ini file.

    ```
    [ml2]
    type_drivers = flat,vlan,gre,vxlan
    tenant_network_types = vlan,gre,vxlan
    mechanism_drivers = openvswitch,l2population

    [ml2_type_vlan]
    network_vlan_ranges = external:1:1000

    [ml2_type_gre]
    tunnel_id_ranges = 1:1000

    [ml2_type_vxlan]
    vni_ranges = 1:1000
    vxlan_group = 239.1.1.1

    [securitygroup]
    enable_security_group = True
    enable_ipset = True
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver

    [ovs]
    local_ip = TENANT_NETWORK_TUNNELS_INTERFACE_IP_ADDRESS
    enable_tunneling = True
    bridge_mappings = external:br-ex

    [agent]
    l2population = True
    tunnel_types = gre,vxlan
    ```

  Note: Replace TENANT_NETWORK_TUNNELS_INTERFACE_IP_ADDRESS with the IP
  address of the tenant network tunnels interface.

  Note: Adjust the VLAN, GRE tunnel, and VXLAN tunnel ID ranges for your
  environment.

1. Start the following services:

  * Open vSwitch
  * Open vSwitch agent

### Verify service operation

1. Source the administrative tenant credentials.

1. Verify presence and operation of the agents. *FIXME*

  ```
  $ neutron agent-list
  +--------------------------------------+--------------------+----------+-------+----------------+---------------------------+
  | id                                   | agent_type         | host     | alive | admin_state_up | binary                    |
  +--------------------------------------+--------------------+----------+-------+----------------+---------------------------+
  | 10b084e5-4ab8-43d6-9b04-6d56f27f9cd4 | Metadata agent     | network1 | :-)   | True           | neutron-metadata-agent    |
  | 2f90ef81-3eed-4ecf-b6b9-2d2c21dda85c | Open vSwitch agent | compute2 | :-)   | True           | neutron-openvswitch-agent |
  | 319563ac-88f9-4352-b63e-e55beb673372 | DHCP agent         | network1 | :-)   | True           | neutron-dhcp-agent        |
  | 3345723e-16e8-4b74-9d15-d7f1f977a3bd | Open vSwitch agent | compute1 | :-)   | True           | neutron-openvswitch-agent |
  | 4643c811-a54a-41da-91a8-c2328bcaeea3 | Open vSwitch agent | network1 | :-)   | True           | neutron-openvswitch-agent |
  | 5ad81671-efc3-4acc-9d5d-030a1c4f6a25 | L3 agent           | compute1 | :-)   | True           | neutron-l3-agent          |
  | 641337fa-99c2-468d-8d7e-89277d6ba144 | Metadata agent     | compute1 | :-)   | True           | neutron-metadata-agent    |
  | 9372e008-bd29-4436-8e01-8ddfd50d2b74 | L3 agent           | network1 | :-)   | True           | neutron-l3-agent          |
  | af9d1169-1012-4440-9de2-778c8fce21b9 | L3 agent           | compute2 | :-)   | True           | neutron-l3-agent          |
  | ee59e3ba-ee3c-4621-b3d5-c9d8123b6cc5 | Metadata agent     | compute2 | :-)   | True           | neutron-metadata-agent    |
  +--------------------------------------+--------------------+----------+-------+----------------+---------------------------+
  ```

## Create initial networks

### External (flat) network

1. Source the administrative tenant credentials.

1. Create the external network. *FIXME*

  ```
  $ neutron net-create ext-net --router:external True \
    --provider:physical_network external --provider:network_type flat
  Created a new network:
  +---------------------------+--------------------------------------+
  | Field                     | Value                                |
  +---------------------------+--------------------------------------+
  | admin_state_up            | True                                 |
  | id                        | 893aebb9-1c1e-48be-8908-6b947f3237b3 |
  | name                      | ext-net                              |
  | provider:network_type     | flat                                 |
  | provider:physical_network | external                             |
  | provider:segmentation_id  |                                      |
  | router:external           | True                                 |
  | shared                    | True                                 |
  | status                    | ACTIVE                               |
  | subnets                   |                                      |
  | tenant_id                 | 54cd044c64d5408b83f843d63624e0d8     |
  +---------------------------+--------------------------------------+
  ```

1. Create a subnet on the external network. *FIXME*

  ```
  $ neutron subnet-create ext-net --name ext-subnet --allocation-pool \
    start=203.0.113.101,end=203.0.113.200 --disable-dhcp \
    --gateway 203.0.113.1 203.0.113.0/24
  Created a new subnet:
  +-------------------+------------------------------------------------------+
  | Field             | Value                                                |
  +-------------------+------------------------------------------------------+
  | allocation_pools  | {"start": "203.0.113.101", "end": "203.0.113.200"}   |
  | cidr              | 203.0.113.0/24                                       |
  | dns_nameservers   |                                                      |
  | enable_dhcp       | False                                                |
  | gateway_ip        | 203.0.113.1                                          |
  | host_routes       |                                                      |
  | id                | 9159f0dc-2b63-41cf-bd7a-289309da1391                 |
  | ip_version        | 4                                                    |
  | ipv6_address_mode |                                                      |
  | ipv6_ra_mode      |                                                      |
  | name              | ext-subnet                                           |
  | network_id        | 893aebb9-1c1e-48be-8908-6b947f3237b3                 |
  | tenant_id         | 54cd044c64d5408b83f843d63624e0d8                     |
  +-------------------+------------------------------------------------------+
  ```

### Tenant (VXLAN) network

1. Source the regular tenant credentials.

1. Create the tenant network. *FIXME*

  ```
  $ neutron net-create demo-net
  Created a new network:
  +-----------------+--------------------------------------+
  | Field           | Value                                |
  +-----------------+--------------------------------------+
  | admin_state_up  | True                                 |
  | id              | ac108952-6096-4243-adf4-bb6615b3de28 |
  | name            | demo-net                             |
  | router:external | False                                |
  | shared          | False                                |
  | status          | ACTIVE                               |
  | subnets         |                                      |
  | tenant_id       | cdef0071a0194d19ac6bb63802dc9bae     |
  +-----------------+--------------------------------------+
  ```

1. Create a subnet on the tenant network. *FIXME*

  ```
  $ neutron subnet-create demo-net --name demo-subnet --gateway 192.168.1.1 192.168.1.0/24
  Created a new subnet:
  +-------------------+------------------------------------------------------+
  | Field             | Value                                                |
  +-------------------+------------------------------------------------------+
  | allocation_pools  | {"start": "192.168.1.2", "end": "192.168.1.254"}     |
  | cidr              | 192.168.1.0/24                                       |
  | dns_nameservers   |                                                      |
  | enable_dhcp       | True                                                 |
  | gateway_ip        | 192.168.1.1                                          |
  | host_routes       |                                                      |
  | id                | 69d38773-794a-4e49-b887-6de6734e792d                 |
  | ip_version        | 4                                                    |
  | ipv6_address_mode |                                                      |
  | ipv6_ra_mode      |                                                      |
  | name              | demo-subnet                                          |
  | network_id        | ac108952-6096-4243-adf4-bb6615b3de28                 |
  | tenant_id         | cdef0071a0194d19ac6bb63802dc9bae                     |
  +-------------------+------------------------------------------------------+
  ```

1. Create a tenant network router. *FIXME*

  ```
  $ neutron router-create demo-router
  Created a new router:
  +-----------------------+--------------------------------------+
  | Field                 | Value                                |
  +-----------------------+--------------------------------------+
  | admin_state_up        | True                                 |
  | distributed           | True                                 |
  | external_gateway_info |                                      |
  | ha                    | False                                |
  | id                    | 635660ae-a254-4feb-8993-295aa9ec6418 |
  | name                  | demo-router                          |
  | routes                |                                      |
  | status                | ACTIVE                               |
  | tenant_id             | cdef0071a0194d19ac6bb63802dc9bae     |
  +-----------------------+--------------------------------------+
  ```

1. Add a tenant subnet interface on the router. *FIXME*

  ```
  $ neutron router-interface-add demo-router demo-subnet
  Added interface b1a894fd-aee8-475c-9262-4342afdc1b58 to router demo-router.
  ```

1. Add a gateway to the external network on the router. *FIXME*

  ```
  $ neutron router-gateway-set demo-router ext-net
  Set gateway for router demo-router
  ```

## Verify operation

1. On the network node, verify creation of the 'qrouter' and 'qdhcp'
   namespaces. *FIXME*

  ```
  # ip netns
  qrouter-4d7928a0-4a3c-4b99-b01b-97da2f97e279
  qdhcp-353f5937-a2d3-41ba-8225-fa1af2538141
  ```

1. On the controller node, ping the tenant router gateway IP address,
   typically the lowest IP address in the external network subnet
   allocation range.

  ```
  # ping -c 4 203.0.113.101
  PING 203.0.113.101 (203.0.113.101) 56(84) bytes of data.
  64 bytes from 203.0.113.101: icmp_req=1 ttl=64 time=0.619 ms
  64 bytes from 203.0.113.101: icmp_req=2 ttl=64 time=0.189 ms
  64 bytes from 203.0.113.101: icmp_req=3 ttl=64 time=0.165 ms
  64 bytes from 203.0.113.101: icmp_req=4 ttl=64 time=0.216 ms

  --- 203.0.113.101 ping statistics ---
  4 packets transmitted, 4 received, 0% packet loss, time 2999ms
  rtt min/avg/max/mdev = 0.165/0.297/0.619/0.187 ms
  ```

1. Source the regular tenant credentials.

1. Launch an instance with an interface on the tenant network.

1. Obtain console access to the instance.

  1. Test connectivity to the tenant network router.

    ```
    $ ping -c 4 192.168.1.1
    PING 192.168.1.1 (192.168.1.1) 56(84) bytes of data.
    64 bytes from 192.168.1.1: icmp_req=1 ttl=64 time=0.357 ms
    64 bytes from 192.168.1.1: icmp_req=2 ttl=64 time=0.473 ms
    64 bytes from 192.168.1.1: icmp_req=3 ttl=64 time=0.504 ms
    64 bytes from 192.168.1.1: icmp_req=4 ttl=64 time=0.470 ms

    --- 192.168.1.1 ping statistics ---
    4 packets transmitted, 4 received, 0% packet loss, time 2998ms
    rtt min/avg/max/mdev = 0.357/0.451/0.504/0.055 ms
    ```

  1. Test connectivity to the Internet.

    ```
    $ ping -c 4 openstack.org
    PING openstack.org (174.143.194.225) 56(84) bytes of data.
    64 bytes from 174.143.194.225: icmp_req=1 ttl=53 time=17.4 ms
    64 bytes from 174.143.194.225: icmp_req=2 ttl=53 time=17.5 ms
    64 bytes from 174.143.194.225: icmp_req=3 ttl=53 time=17.7 ms
    64 bytes from 174.143.194.225: icmp_req=4 ttl=53 time=17.5 ms

    --- openstack.org ping statistics ---
    4 packets transmitted, 4 received, 0% packet loss, time 3003ms
    rtt min/avg/max/mdev = 17.431/17.575/17.734/0.143 ms
    ```

1. Create the appropriate security group rules to allow ping and SSH access
   to the instance.

1. Create a floating IP address. *FIXME*

  ```
  $ neutron floatingip-create ext-net
  Created a new floatingip:
  +---------------------+--------------------------------------+
  | Field               | Value                                |
  +---------------------+--------------------------------------+
  | fixed_ip_address    |                                      |
  | floating_ip_address | 203.0.113.102                        |
  | floating_network_id | 9bce64a3-a963-4c05-bfcd-161f708042d1 |
  | id                  | 05e36754-e7f3-46bb-9eaa-3521623b3722 |
  | port_id             |                                      |
  | router_id           |                                      |
  | status              | DOWN                                 |
  | tenant_id           | 7cf50047f8df4824bc76c2fdf66d11ec     |
  +---------------------+--------------------------------------+
  ```

1. Associate the floating IP address with the instance.

  ```
  $ nova floating-ip-associate demo-instance1 203.0.113.102
  ```

1. On the controller node, ping the floating IP address associated with
   the instance.

  ```
  $ ping -c 4 203.0.113.102
  PING 203.0.113.102 (203.0.113.112) 56(84) bytes of data.
  64 bytes from 203.0.113.102: icmp_req=1 ttl=63 time=3.18 ms
  64 bytes from 203.0.113.102: icmp_req=2 ttl=63 time=0.981 ms
  64 bytes from 203.0.113.102: icmp_req=3 ttl=63 time=1.06 ms
  64 bytes from 203.0.113.102: icmp_req=4 ttl=63 time=0.929 ms

  --- 203.0.113.102 ping statistics ---
  4 packets transmitted, 4 received, 0% packet loss, time 3002ms
  rtt min/avg/max/mdev = 0.929/1.539/3.183/0.951 ms
  ```

This work is licensed under the Creative Commons Attribution 4.0
International License. To view a copy of this license, visit
http://creativecommons.org/licenses/by/4.0/.
