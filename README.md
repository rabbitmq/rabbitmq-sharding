# RabbitMQ Sharding Plugin #

This plugin introduces the concept of sharded queues for
RabbitMQ. Sharding is done at the exchange level, that is, messages
will be partitioned across queues by one exchange that we should
define as sharded. The machinery used behind the scenes implies
defining an exchange that will partition, or shard messages across
queues. The partitioning will be done automatically for you, i.e: once
you define an exchange as _sharded_, then the supporting queues will
be automatically created and messages will be sharded across them.

The following graphic depicts how the plugin works from the standpoint
of a publisher and a consumer:

![Sharding Overview](http://hg.rabbitmq.com/rabbitmq-sharding/raw-file/6fea09e847d5/docs/sharded_queues.png)

As you can see in the graphic, the producers publishes a series of
messages, those messages get partitioned to different queues, and then
our consumer get messages from one of those queues. Therefore if you
have a partition with 3 queues, then you will need to have at least 3
consumers to get all the messages from those queues.

## Auto-scaling ##

One interesting property of this plugin, is that if you add more nodes
to your RabbitMQ cluster, then the plugin will automatically create
more shards in the new node. Say you had a shard with 4 queues in
`node a` and `node b` just joined the cluster. The plugin will
automatically create 4 queues in `node b` and join them to the shard
partition. Already delivered messages _will not_ be rebalanced, but
newly arriving messages will be partitioned to the new queues.

## Partitioning Messages ##

The exchanges that ship by default with RabbitMQ work in a "all or
nothing" fashion, i.e: if a routing key matches a set of queues bound
to the exchange, then RabbitMQ will route the message to all the
queues in that set. Therefore for this plugin to work, we need to
route messages to an exchange that would partition messages, so they
are routed to _at most_ one queue.

The plugin provides a new exchange type `"x-modulus-hash"` that will use
the traditional hashing technique applying to partition messages
across queues.

The `"x-modulus-hash"` exchange will hash the routing key used to
publish the message and then it will apply a `Hash mod N` to pick the
queue where to route the message, where N is the number of queues
bound to the exchange. This exchange will completely ignore the
binding key used to bind the queue to the exchange.

You could also use other exchanges that have similar behaviour like
the _Consistent Hash Exchange_ or the _Random Exchange_.  The first
one has the advantage of shipping directly with RabbitMQ.

## Consuming from a sharded queue ##

While the plugin creates a bunch of queues behind the scenes, the idea
is that those queues act like a big logical queue where you consume
messages from.

An example should illustrate this better: let's say you declared the
exchange _images_ to be a sharded exchange. Then RabbitMQ created
behind the scenes queues _shard: - nodename images 1_, _shard: -
nodename images 2_, _shard: - nodename images 3_ and _shard: -
nodename images 4_. Of course you don't want to tie your application
to the naming conventions of this plugin. What you would want to do is
to perform a `basic.consume('images')` and let RabbitMQ figure out the
rest. This plugin does exactly that.

TL;DR: if you have a shard called _images_, then you can directly
consume from a queue called _images_.

How does it work? The plugin will chose the queue from the shard with
the _least amount of consumers_, provided the queue contents are local
to the broker you are connected to.

## Installing ##

Install the corresponding .ez files from our
[Community Plugins page](http://www.rabbitmq.com/community-plugins.html).

Then run the following command:

```bash
rabbitmq-plugins enable rabbitmq_sharding
```

You'd probably want to also enable the Consistent Hash Exchange
plugin.

## Usage ##

Once the plugin is installed you can define an exchange as sharded by
setting up a policy that matches the exchange name. For example if we
have the exchange called `shard.images`, we could define the following
policy to shard it:

```bash
$CTL set_policy images-shard "^shard.images$" '{"shards-per-node": 2, "routing-key": "ignored"}'
```

This will create `2` sharded queues per node in the cluster, and will
bind those queus using the `"ignored"` routing key.

## Building the plugin ##

Get the RabbitMQ Public Umbrella ready as explained in the
[RabbitMQ Plugin Development Guide](http://www.rabbitmq.com/plugin-development.html).

Move to the umbrella folder an then run the following commands, to
fetch dependencies:

```bash
make up
cd ../rabbitmq-sharding
make
```

## Plugin Status ##

At the moment the plugin is __experimental__ in order to receive
feedback from the community.

## LICENSE ##

See the LICENSE file.

## Extra information ##

Some information about how the plugin affects message ordering and
some other details can be found in the file README.extra.md
