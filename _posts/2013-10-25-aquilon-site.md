---
layout: article
title: Starting a site with Aquilon
category: documentation
modified: 2013-11-16
author: Luis Fernando Muñoz Mejías
---

**THIS IS WORK IN PROGRESS**

## Introduction

After
[installing Aquilon](/documentation/2012/10/31/install-aquilon.html), we
want to add hosts to our instance, and produce some useful
configuration.

We'll explain in this document the steps to set up the clusters
for the Daily Planet, Superman's employer.

### Aquilon terminology

These are the basic terms in Aquilon operations:

* `archetype`: The highest possible grouping of hosts into distinctly
    seperate types, analogous to a QWG site. Archetypes are a bundle
    that expresses how to build something. It defines what set of
    templates to use (for example, what operating systems are
    available, etc).  Hosts therefore require an archetype to define
    how they are compiled.
* `broker`: The backend which the aq client communicates with and the
    owner of all object and production templates.  We'll refer to it
    also as `aqd`.
* `cluster`: A group of hosts related in some way, different to an
    archetype in that hosts may or may not be in a cluster. When
    grouping hosts into a cluster, an object profile is also built for
    the cluster, as well as the hosts. Clusters go through a
    completely different schema and build process to how hosts are
    built and therefore have a different archetypes.
* `domain`: A set of shared Quattor templates. Entities to be
  configured live in exactly one domain. A domain is currently
  implemented as a branch if the template-king git repository.
* `feature`: A re-useable configuration template that configures a
  specific thing but does not in itself describe a complete
  system. (So similar to a chef recipe.) A feature may be included in
  a personality or anywhere else that PAN template code can be
  included.
* `personality`: Analogous to QWG machine types, describes the
    services required but not the instance (selected using plenary
    template information).
* `plenary`: Template generated on the fly from an external
    source.
* `sandbox`: A development area for making changes to (Pan)
  templates. Each sandbox must be started from a domain and hosts or
  clusters may be managed into the sandbox to allow for pre-deployment
  testing. Once the template code is fully tested, it must be
  published back to the broker and then deployed to the shared domain.
* `service`: A network-based service such as DNS, DHCP, NTP, etc. Each
 service can have multiple instances. Service instances must be mapped
 which allows the administrator to model client hosts connecting to
 different instances based on location and optionally personality.

### Before proceeding

We'll interact with the broker using the `aq` command, which has lots
of sub-commands.  Each sub-command has a detailed help, and man pages
can be found in the appendix of the [Aquilon book](http://FIXME).

In this stage we will be adding a site, so we will use `add_*`
commands.  Most of the commands we'll use have `show_`, `update_` and
`del_` counterparts.

## Archetypes, domains...

You need to start by declaring archetypes and the basic domains for
your Aquilon instance.  Let's imagine we have two archetypes, one for
Linux hosts under Quattor control and another one for IPMI interfaces

```bash
aq add_archetype --archetype 'linux' --compilable
aq add_archetype --archetype 'bmc' --nocompilable
```

Next come the domains, which are just Git branches with some metadata.
We'll define a `prod`uction and a `test`ing domains.

```bash
aq add_domain --domain 'prod'
aq add_domain --domain 'test'
```

## Storing your inventory

The Aquilon database contains the entire inventory of your
infrastructure.  You may use it as your asset management database, or
feed it from your existing one.  You will have to store all your
hardware, its characteristics and locations, which may be an annoying
process.

### Geographical locations

Before starting, you have to add the geographical locations of your
infrastructure.  The meanings of the commands to run should be
obvious.  We list here an example:

```bash
aq add_organization --organization 'daily_planet'
aq add_hub --hub 'pacific' --organization 'daily_planet'
aq add_continent --continent 'america' --hub 'pacific'
aq add_country --country 'us' --continent 'america'
aq add_city --city 'metropolis' --fullname 'Metropolis' --country 'us' --timezone 'dct'
aq add_building --building 'hq' --city 'metropolis' --address '355, 1000 Broadway'
aq add_room --room 'supersecret' --building 'hq'
```

List your own addresses, rooms...  but remember that each step depends
on the previous ones.  If you want to add European countries, you have
to insert Europe as a continent first!

### Your hardware inventory

Next comes your hardware.  At the very least, your racks, chassis and
servers will be stored here.  It is a good place to store other
hardware such as switches.

For racks, you'll have to specify on which row and column inside that
row they are.

```bash
aq add_rack --rackid 'forlexluthor' --row 'R' --column 'C' --building 'hq'
```

Next come servers, switches, routers... we'll go for servers,
_machine_s in Aquilon terms.  Aquilon has to know the exact vendor and
model for each server, so here we go:

```bash
aq add_vendor --vendor 'luthorindustries' --comments "Bad guys make good sells"
aq add_model --model 'spyserver' --vendor 'luthorindustries' --type 'rackmount'
aq add_machine --machine 'illegaltapper' --model 'spyserver' --rack 'forlexluthor'
```

#### Network interfaces

Each machine has a number of network interfaces, with their MAC
addresses.  Don't forget to add them!

```bash
aq add_interface --interface eth0 --machine 'illegaltapper' --mac 'AA:BB:CC:DD:EE:FF'
```

## Declaring networks

Once we have the hardware in place, it's time to declare the networks
our hosts live in.

We'll use the `add_network` command:

```bash
aq add_network --network 'leaks' --ip '192.168.1.0' --netmask '255.255.255.0' --city 'metropolis'
aq add_network --network 'reporters' --ip '192.168.100.0' --netmask '255.255.255.0' --city 'metropolis'
```

### Wait, how do I define the routers for my networks?

When we declare our networks, Aquilon assumes the first IP address is
reserved for the router.  If this isn't correct, you have to modify
the configuration for the broker.  Create a `network_` section, and
declare the default gateway offset.  For instance, let's assume that
the gateway for `leaks` network above is on 192.168.1.30.  We edit `/etc/aqd.conf`:

```ini
[network_leaks]
default_gateway_offset=30
```

And finally we restart the broker:

```bash
service aqd restart
```

### DNS domains

Our network has some DNS domains.  We have to add them:

```bash
aq add_dns_domain --dns_domain 'dailyplanet.com'
```

## Declaring your first host

Now we can declare a host on our `illegaltapper` machine.

To do so, we have already:

* Stored all the relevant hardware in the database
* Declared all the network interfaces, with their MAC addresses
* Declared all the networks it will be connected to
* Declared all the DNS domains our host will live in

Now, we use all that information to add a `tapping` host:

```bash
aq add_host --hostname 'tapping.dailyplanet.com' --machine 'illegaltapper' --ip '192.168.1.3'
```

Congratulations!  You have your first host!  Run `aq show_host --all`
to see it.

But we aren't done yet.  This host will do nothing.  It's useless.  We
need to bind it to some personality, and add features to that personality.

### Personalities and features

## Importing your first templates

## Declaring your first sandbox

## Compiling the first host

## Deploying the first configuration

## Conclusion