Title: Deploying applications
TODO: Add 'centos' and 'windows' stuff to series talk
      Review whether Juju should go to the store when pointing to a local dir
        with non-existant charm. It did not for me but the old version of this
        doc said it should.
      Review required. Channnels especially
      This page is too long. It should contain just basic stuff and link to sub-pages.
      Hardcoded: Ubuntu codenames

# Deploying applications

The fundamental point of Juju is that you can use it to deploy applications
through the use of charms (the magic bits of code that make things just work).
These charms can exist in the [Charm Store](https://jujucharms.com/store) or on
the file system (previously downloaded from the store or written locally).

Charms use the concept of *series* analogous as to how Juju does with Ubuntu
series ('Trusty', 'Xenial', etc). For the most part, this is transparent as
Juju will use the most relevant charm to ensure things "just work". This makes
deploying applications with Juju fun and easy.

## Deploying from the charm store

Typically, applications are deployed using the online charms. This ensures that
you get the latest version of the charm. To deploy in this way:

```bash
juju deploy mysql
```

This will create a machine and use the latest online MySQL charm (for your
default series) to deploy a MySQL application.

!!! Note:
    The default series can be configured at a model level, see
    [Configuring models][models] for further details. In the absence of this
    setting, the default is to use the series specified by the charm.

Assuming that the Xenial series charm exists and was used above, an equivalent
command is:

```bash
juju deploy cs:xenial/mysql
```

Where 'cs' denotes the charm store.

!!! Note:
    A used charm gets cached on the controller's database to minimize network
    traffic for subsequent uses.

### Channels

The charm store offers charms in different stages of development. Such stages
are called *channels*.

Channels offer a way for charm developers, and the users of charms, to manage
and offer charms at various stages of development. Some users may want the very
latest features, or be part of a beta test; others may want to only install the
most reliable software. The channels are:

 - **stable**: (default) This is the latest, tested, working stable version of the charm.
 - **candidate**: A release candidate. There is high confidence this will work
   fine, but there may be minor bugs.
 - **beta**: A beta testing milestone release.
 - **edge**: The very latest version - expect bugs!

As each new version of a charm is automatically versioned, these channels serve
as pointers to a specific version number. It may be that after time a beta
version becomes a candidate, or a candidate (hopefully) becomes the new stable
version.

By default you will get the 'stable' channel, but you can specify a channel
when using the `deploy` command:

```bash
juju deploy mysql --channel <channel_name>
```

In the case of there being no version of the charm specified for that channel,
Juju will fall back to the next 'most stable'; e.g. if you were to specify the
'beta' channel, but no charm version is set for that channel, Juju will try to
deploy from the 'candidate' channel instead, and so on. This means that
whenever you specify a channel, you will always end up with something that best
approximates your choice if it is not available.

See [Upgrading applications][charms-upgrading] for how charm upgrades work.

## Deploying a multi-series charm

Charms can be created that support more than one release of a given operating
system distro, such as the multiple Ubuntu releases shown below. It is not
possible to create a charm to support multiple distros, such as one charm for
both Ubuntu and CentOS. Supported series are added to the charm metadata like
this:

```
name: mycharm
summary: "Great software"
description: It works
maintainer: Some One <some.one@example.com>
categories:
   - databases
series:
   - trusty
   - xenial
provides:
   db:
     interface: pgsql
requires:
   syslog:
     interface: syslog
```

The default series for the charm is the first one listed. So, in this example,
to deploy `mycharm` on `trusty`, all you need is:

```bash
juju deploy mycharm
```

You can specify a different series using the `--series` flag:

```bash
juju deploy mycharm --series xenial
```

You can force the charm to deploy using an unsupported series using the
`--force` flag:

```bash
juju deploy mycharm --series yakkety --force
```

Here is a more complete example showing a new machine being added that uses
a different series than is supported by our `mycharm` example and then forcing
the charm to install:

```bash
juju add-machine --series yakkety
Machine 1 added.
juju deploy mycharm --to 1 --series yakkety --force
```

It may be required to use `--force-series` when upgrading charms. For example,
in a case where an application is initially deployed using a charm that
supports `precise` and `trusty`. If a new version of the charm is released that
only supports `trusty` and `xenial` then it will be allowed to upgrade
applications deployed on `precise`, but only using `--force-series`, like this:

```bash
juju upgrade-charm mycharm --force-series
```

## Deploying from a local charm

It is possible to deploy applications using local charms. See
[Deploying charms offline][charms-offline-deploying] for further guidance.

## Configuring at deployment time

Deployed applications usually start with a sane default configuration. However,
for some applications it may be desirable (and quicker) to configure them at
deployment time. This can be done whether a charm is deployed from the Charm
Store or from a local charm.

See [Application configuration](../charms-config.html) for more on this.

## Deploying to specific machines and containers

It is possible to specify which machine or container an application is to be
deployed to. One notable reason is to reduce costs when using a public cloud;
applications can be consolidated instead of dedicating a machine per application
unit.

Below, the `--constraints` option is used to create an LXD controller with
enough memory for other applications to run. The `--to` option is used to
specify a machine:

```bash
juju bootstrap --constraints="mem=4G" lxd lxd-controller
juju deploy mysql
juju deploy --to 0 rabbitmq-server
```

Here, MySQL is deployed as the first unit (in the 'default' model) and so ends
up on machine '0'. Then Rabbitmq gets deployed to machine '0' as well.

Applications can also be deployed to containers:

```bash
juju deploy mysql --to 24/lxd/3
juju deploy mysql --to lxd:25
```

Above, MySQL is deployed to existing container '3' on machine '24'. Afterwards,
a MySQL application is deployed to a new container on machine '25'.

The above examples show how to deploy to a machine where you know the machine's
identifier. The output to `juju status` will provide this information.

It is also possible to deploy units using placement directives as `--to`
arguments. Placement directives are provider specific. For example:

```bash
juju deploy mysql --to zone=us-east-1a
juju deploy mysql --to host.mass
```

The first example deploys to a specified zone for AWS. The second example
deploys to a named machine in MAAS.

The `add-unit` command also supports the `--to` option, so it's now possible to
specifically target machines when expanding application capacity:

```bash
juju deploy --constraints="mem=4G" openstack-dashboard
juju add-unit --to 1 rabbitmq-server
```

There should now be a second machine running both the openstack-dashboard
application and a second unit of the rabbitmq-server application. The
`juju status` command will show this.

These two features make it much easier to deploy complex applications such as
OpenStack which use a large number of charms on a limited number of physical
servers.

As with deploy, the --to option used with `add-unit` also supports placement
directives. A comma separated list of directives can be provided to cater for
the case where more than one unit is being added.

```bash
juju add-unit rabbitmq-server -n 4 --to zone=us-west-1a,zone=us-east-1b
juju add-unit rabbitmq-server -n 4 --to host1,host2,host3,host4
```

Any extra placement directives are ignored. If not enough placement directives
are supplied, then the remaining units will be assigned as normal to a new,
clean machine.

## Deploying to spaces

More complex networks can be configured using spaces. Spaces group one or more
routable subnets with common ingress and egress rules to give the operator much
better and finer-grained control over all networking aspects of a model and its
application deployments.

See the [How to configure more complex networks using spaces][spaces] for
details on creating and listing spaces.

When deploying a charm or a bundle, you can specify a space using the `--bind`
argument following the `juju deploy` command.

When deploying an application to a target with multiple spaces, the operator
must specify which space to use because ambiguous bindings will result in a
provisioning failure. For example, the following will deploy the 'mysql'
application to the 'db-space' space:

```bash
juju deploy mysql --bind db-space
```

For finer control, the `--bind` argument can also be used to specify how
specific charm-defined endpoints are connected to specific spaces, including a
default option for any interfaces not specified:

```bash
juju deploy --bind "default-space db=db-space db-admin=admin-space" mysql
```

For information on building bundles with bindings, see
[Using and Creating Bundles][creatingbundles].

Both the `add-machine` and `deploy` commands allow the specification of a
spaces constraint using the `--constraints` argument:

```bash
juju add-machine --constraints spaces=db-space
```

The spaces constraint allows you to select an instance for the new machine or
unit, connected to one or more existing spaces. Both positive and negative
entries are accepted, the latter prefixed by "^", in a comma-delimited list.
For example, given the following:

```
--constraints spaces=db-space,^storage,^dmz,internal
```

Juju will provision instances connected to (with IP addresses on) one of the
subnets of both db-space and internal spaces, and NOT connected to either the
storage or dmz spaces.

See [Constraints][constraints] for more general information regarding
constraints. To learn about `extra-bindings`, which provide a way to declare an
extra bindable endpoint that is not a relation, see [Charm metadata][metadata].

!!! Note:
    Juju's knowledge of the available spaces on the underlying cloud may lag
    behind recent changes. To force Juju to re-examine the spaces it can use,
    you should run the command `juju reload-spaces`.

## Juju retry-provisioning

You can use the `retry-provisioning` command in cases where deploying
applications, adding units, or adding machines fails. It allows you to specify
machines which should be retried to resolve errors reported with `juju status`.

For example, after having deployed 100 units and machines, status reports that
machines '3', '27' and '57' could not be provisioned because of a 'rate limit
exceeded' error. You can ask Juju to retry:

```bash
juju retry-provisioning 3 27 57
```

## Considerations

Although we are working to have each application co-locatable without the
danger of conflicting configuration files and network configurations this work
is not yet complete.

While the `add-unit` command supports the `--to` option, you can elect not use
`--to` when doing an "add-unit" to scale out the application on its own node.

```bash
juju add-unit rabbitmq-server
```

This will allow you to save money when you need it by using `--to`, but also
horizontally scale out on dedicated machines when you need to.


<!-- LINKS -->

[models]: ./models-config.html
[spaces]: ./network-spaces.html
[creatingbundles]: ./charms-bundles.html#binding-endpoints-of-applications-within-a-bundle
[metadata]: ./authors-charm-metadata.html
[constraints]: ./charms-constraints.html
[charms-upgrading]: ./charms-upgrading.html
[charms-offline-deploying]: ./charms-offline-deploying.html
