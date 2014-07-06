title: 'How to Create a Data Cluster and Live Through It'
id: 106
categories:
  - Node.js
date: 2014-06-13 18:17:01
tags:
---

This post originally posted at [BigPanda's Blog](http://bit.ly/1v4ycZY "BigPanda"), BigPanda is the cool start-up company that I currently work at, you can find more about how it manages alerts and fights the alert fatigue [here](http://bigpanda.io "BigPanda").

As the world of software development is moving to [BigPanda](http://bigpanda.io "BigPanda"), ahmm... sorry, I meant Big Data, we need to pay attention to synchronization, and it's becoming one of the most difficult challenges we face.

Here at BigPanda, we receive a lot of alerts from [many monitoring systems](http://bigpanda.io/integrations/), and naturally we want to keep the data synchronized and consistent. One issue we face is when a server is reported by a monitoring system as **DOWN**, and then, one moment later is reported as **UP**. We need the two events to be processed in relation to the same server and in the correct order, to ensure the status does not end up being **DOWN**.

<!--more-->Our data cluster was written because we wanted to synchronize the data in our back-end applications, and not in the database itself. We believe this is a better approach for several reasons:

*   _Vendor independence_ - Today we use [MongoDB](http://www.mongodb.org "mongodb"), but our needs and the database may change in the future.
*   _Control_ - Sharding complex data entities across servers is difficult without understanding the application-level entities. We know the entities, but MongoDB, [Cassandra](http://cassandra.apache.org "Cassanda") or [HBase](http://hbase.apache.org "HBase") don't.
*   _Synchronization_ - As described above, we need to process correctly two updates for the same entity that occur at the same time.

#### Our Solution: Frodo.io

To address this challenge we developed Frodo.io (Github: [https://github.com/bigpandaio/frodo](https://github.com/bigpandaio/frodo)). Frodo.io exposes an easy to use API that allows you to easily synchronize distributed backends. It returns the server by entity details, and ensures it will always return the same server for a given ID. Frodo orchestrates **[etcd](https://github.com/coreos/etcd "etcd")** and **[hashring](https://github.com/3rd-Eden/node-hashring "hashring").**

**etcd** - is _a highly-available key value store for shared configuration and service discovery_ (from [etcd's README](https://github.com/coreos/etcd/blob/master/README.md "etcd"))

**hashring** - is a consistent hashing algorithm implementation. The hashing algorithm is optimized so that when the hash table is resized, the number of keys that it needs to migrate is relatively small.

Let's look at an example in which we're initializing frodo, adding 3 servers to the ring, retrieving the server that should be in use for this id and making a REST request to the server:

``` javascript
  var Frodo = require('frodoio');
  var frodo = new Frodo({ context: 'appname', etcdServers: [ { host: '127.0.0.1', port: 4001 } ]});
  Q.all(frodo.addServer('dataacess1:3030'),
    frodo.addServer('dataacess2:3030'),
    frodo.addServer('dataacess3:3030')).then(function () {
      frodo.serverById('someId', function (server)
        request.put('http://' + server + '/entity');
      });
    });
```

_Note: don't forget to _`npm install frodoio`_ and _`npm install q`_ in order to run this code snippet_

No matter how many times you will make a call to a server, as long as the server list is the same, you'll consistently get the same server for the same id!

In the following, more comprehensive, example, we separate the server code from the client code. Every server in the data cluster registers itself in frodo.io. In the client we're requesting a server to handle a specific entity and then calling a url on that server.

##### Server:

``` javascript
  var Frodo = require('frodoio');
  var frodo = new Frodo({ context: 'appname', etcdServers: [ { host: '127.0.0.1', port: 4001 } ]});

  // http://stackoverflow.com/questions/3653065/get-local-ip-address-in-node-js
  var ip = _.chain(require('os').networkInterfaces()).flatten().filter(function(val){ return (val.family == 'IPv4' &amp;&amp; val.internal == false) }).pluck('address').first().value();
  var me = ip + ':3030';

  function start(done) {
    frodo.addServer(me).then(done);
  }

  function stop(done) {
    frodo.removeServer(me).then(function () {
      frodo.close().then(done);
    });
  }
```

In the first two lines we are initializing frodo with the etcd config and a `context` for the application (usually just your app name). Then we fetch the IP of the current server.
The `start` and `stop` functions are "listeners" and your application should call them when it starts or stops.

##### Client:

``` javascript
  var Frodo = require('frodoio');
  var frodo = new Frodo({ context: 'appname', etcdServers: [ { host: '127.0.0.1', port: 4001 } ]});

  frodo.serverById(id).then(function (server) {
    request.put('http://' + server + '/entity');
  });
```

After initialization, we request the server by an `id`, just like in the previous example.

Try to run at least two instances of the server, and then run the clients repeatedly with the same id to get the same server and with different ids to get different servers. The entities are evenly distributed with **hashring**.

For more information about Frodo's capabilities and the API, please go to [frodo's github homepage](https://github.com/bigpandaio/frodo "frodo.io").