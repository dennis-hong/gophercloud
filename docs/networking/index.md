---
layout: docpage
title: Getting Started with Networking v2
---

* [Setup](#setup)
* [Networks](#networks)
	* [Create a network](#create-network)
	* [List networks](#list-networks)
	* [Get network details](#get-network)
	* [Update a network](#update-network)
	* [Delete a network](#delete-network)
* [Subnets](#subnets)
	* [Create a subnet](#create-subnet)
	* [List subnets](#list-subnets)
	* [Get subnet details](#get-subnet)
	* [Update a subnet](#update-subnet)
	* [Delete a subnet](#delete-subnet)
* [Ports](#ports)
	* [Create a port](#create-port)
	* [List ports](#list-ports)
	* [Get port details](#get-port)
	* [Update a port](#update-port)
	* [Delete a port](#delete-port)

## <a name="setup"></a>Setup

In order to interact with OpenStack APIs, you must first pass in your auth
credentials to a `Provider` struct. Once you have this, you then retrieve
whichever service struct you're interested in - so in our case, we invoke the
`NewNetworkV2` method:

{% highlight go %}
import "github.com/rackspace/gophercloud/openstack"

authOpts, err := utils.AuthOptions()

provider, err := openstack.AuthenticatedClient(authOpts)

client, err := openstack.NewNetworkV2(provider, gophercloud.EndpointOpts{
	Name:   "neutron",
	Region: "RegionOne",
})
{% endhighlight %}

If you're unsure about how to retrieve credentials, please read our [introductory
guide](/docs) which outlines the steps you need to take.

## <a name="networks"></a>Networks

A network is the central resource of the OpenStack Neutron API. If you were to
compare it to physical networking, a Neutron network would be analagous to a
VLAN - which is an isolated [broadcast domain](http://en.wikipedia.org/wiki/Broadcast_domain)
inside a larger [layer-2 network](http://en.wikipedia.org/wiki/Data_link_layer).
Because of this virtualized partitioning, a virtual network can only share
packets with other networks through one or more routers.

### <a name="create-network"></a>Create a network

{% highlight go %}
import "github.com/rackspace/gophercloud/openstack/networking/v2/networks"

// We specify a name and that it should forward packets
opts := networks.CreateOpts{Name: "main_network", AdminStateUp: networks.Up}

// Execute the operation and get back a networks.Network struct
network, err := networks.Create(client, opts).Extract()
{% endhighlight %}

### <a name="list-networks"></a>List networks

{% highlight go %}
import "github.com/rackspace/gophercloud/pagination"

// We have the option of filtering the network list. If we want the full
// collection, leave it as an empty struct
opts := networks.ListOpts{Shared: false}

// Retrieve a pager (i.e. a paginated collection)
pager := networks.List(client, opts)

// Define an anonymous function to be executed on each page's iteration
err := pager.EachPage(func(page pagination.Page) (bool, error) {
	networkList, err := networks.ExtractNetworks(page)

	for _, n := range networkList {
		// "n" will be a networks.Network
	}
})
{% endhighlight %}

### <a name="get-network"></a>Get details for an existing network

{% highlight go %}
// We need to know what the UUID of our network is and pass it in as a string
network, err := networks.Get(client, "id").Extract()
{% endhighlight %}

### <a name="update-network"></a>Update an existing network

You can update a network's name, along with its "shared" or "admin" status:

{% highlight go %}
opts := networks.UpdateOpts{Name: "new_name", Shared: true}

// Like Get(), we need the UUID in string form
network, err := networks.Update(client, "id", opts)
{% endhighlight %}

### <a name="delete-network"></a>Delete a network

{% highlight go %}
result := networks.Delete(client, "id")
{% endhighlight %}

## <a name="subnets"></a>Subnets

A subnet is a block of IP addresses (either version 4 or 6) that are assigned
to devices in a particular network. A device in the context of Neutron
specifically means a virtual machine (Compute instance). For this reason, each
subnet must have a [CIDR](http://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing)
and be associated with a network.

### <a name="create-subnet"></a>Create a subnet

{% highlight go %}
import "github.com/rackspace/gophercloud/openstack/networking/v2/subnets"

// You must associate a new subnet with an existing network - to do this you
// need its UUID. You must also provide a well-formed CIDR value.
opts := subnets.CreateOpts{
	NetworkID:  "network_id",
	CIDR:       "192.168.199.0/24",
	IPVersion:  subnets.IPv4,
	Name:       "my_subnet",
}

// Execute the operation and get back a subnets.Subnet struct
subnet, err := subnets.Create(client, opts).Extract()
{% endhighlight %}

### <a name="list-subnets"></a>List all subnets

{% highlight go %}
import "github.com/rackspace/gophercloud/pagination"

// We have the option of filtering subnets. For example, we may want to return
// every subnet that belongs to a specific network. Or filter again by name.
opts := subnets.ListOpts{NetworkID: "some_uuid"}

// Retrieve a pager (i.e. a paginated collection)
pager := subnets.List(client, opts)

// Define an anonymous function to be executed on each page's iteration
err := pager.EachPage(func(page pagination.Page) (bool, error) {
	subnetList, err := subnets.ExtractSubnets(page)

	for _, s := range subnetList {
		// "s" will be a subnets.Subnet
	}
})
{% endhighlight %}

### <a name="get-subnet"></a>Get details for an existing subnet

{% highlight go %}
// We need to know what the UUID of our subnet is and pass it in as a string
subnet, err := subnets.Get(client, "id").Extract()
{% endhighlight %}

### <a name="update-subnet"></a>Update an existing subnet

You can edit the name, gateway IP address, DNS nameservers, host routes
and "enable DHCP" status.

{% highlight go %}
opts := subnets.UpdateOpts{Name: "new_subnet_name"}
subnet, err = subnets.Update(client, "id", opts).Extract()
{% endhighlight %}

### <a name="delete-subnet"></a>Delete a subnet

{% highlight go %}
result := subnets.Delete(client, "id")
{% endhighlight %}

## <a name="ports"></a>Ports

Before talking about what ports are, an important concept to define first are
network switches (both the virtual and physical kind). A network switch connects
different network segments together, and a port is the location where devices
connect to the switch. A device in our case is usually a virtual machine. For
more information about these terms, read this [related article](http://www.wisegeek.com/what-is-a-switch-port.htm).

### <a name="create-port"></a>Create a port

{% highlight go %}
import "github.com/rackspace/gophercloud/openstack/networking/v2/ports"

// You must associate a new port with an existing network - to do this you
// need its UUID. Also notice the "FixedIPs" field; this allows you to specify
// either a specific IP to use for this port, or the subnet ID from which a
// random free IP is selected.
opts := ports.CreateOpts{
	NetworkID:    "network_id",
	Name:         "my_port",
	AdminStateUp: ports.Up,
	FixedIPs:     []ports.IP{ports.IP{SubnetID: "subnet_id"}},
}

// Execute the operation and get back a subnets.Subnet struct
port, err := ports.Create(client, opts).Extract()
{% endhighlight %}

### <a name="list-ports"></a>List all ports

{% highlight go %}
import "github.com/rackspace/gophercloud/pagination"

// We have the option of filtering ports. For example, we may want to return
// every port that belongs to a specific network. Or filter again by MAC address.
opts := ports.ListOpts{NetworkID: "some_uuid", MACAddress: "some_addr"}

// Retrieve a pager (i.e. a paginated collection)
pager := ports.List(client, opts)

// Define an anonymous function to be executed on each page's iteration
err := pager.EachPage(func(page pagination.Page) (bool, error) {
	portList, err := ports.ExtractPorts(page)

	for _, s := range portList {
		// "p" will be a ports.Port
	}
})
{% endhighlight %}

### <a name="get-port"></a>Get details for an existing port

{% highlight go %}
// We need to know what the UUID of our port is and pass it in as a string
port, err := ports.Get(client, "id").Extract()
{% endhighlight %}

### <a name="update-port"></a>Update an existing port

You can edit the name, admin state, fixed IPs, device ID, device owner and
security groups.

{% highlight go %}
opts := ports.UpdateOpts{Name: "new_port_name"}
port, err = ports.Update(client, "id", opts).Extract()
{% endhighlight %}

### <a name="delete-port"></a>Delete a port

{% highlight go %}
result := ports.Delete(client, "id")
{% endhighlight %}