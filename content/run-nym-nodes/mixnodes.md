---
title: "Mixnodes"
weight: 30
description: "Mixnodes accept Sphinx packets, shuffle packets together, and forward them onwards, providing strong anonymity for internet users."
---


{{% notice info %}}
The Nym mixnode was built in the [building nym](/docs/run-nym-nodes/build-nym/) section. If you haven't yet built Nym and want to run the code, go there first.
{{% /notice %}}

To join the Nym testnet as a mixnode, copy the `nym-mixnode` binary from the `target/release` directory up to your server (or compile it on the server).

### Upgrading from an earlier version

If you have already been running a node on the Nym network v0.8.1, you can use the `upgrade` command to upgrade your configs in place. 

```shell
./nym-mixnode upgrade --id your-node-id --current-version 0.8.1
```

If you are participating in the Nym incentives program, you can enter your Liquid address to receive your NYMPH tokens during `upgrade` by using the `--incentives flag`:

```shell
./nym-mixnode upgrade --id your-node-id --current-version 0.8.1 --incentives-address YOURADDRESSHERE
```


### Initialize a mixnode

If you are new to Nym, here's how you initialize a mixnode:

```shell
./nym-mixnode init --id winston-smithnode --host $(curl ifconfig.me) --location YourCity
```

To participate in the Nym testnet, `--host` must be publicly routable on the internet. It can be either an Ipv4 or IPv6 address. Your node *must* be able to send TCP data using *both* IPv4 and IPv6 (as other nodes you talk to may use either protocol). The above command gets your IP automatically using an external service `$(curl ifconfig.me)`. Enter it manually if you don't have `curl` installed.

The `--location` flag is optional but helps us debug the testnet. 

You can pick any `--id` you want.

When you run `init`, configuration files are created at `~/.nym/mixnodes/<your-id>/`. 

The `init` command will refuse to destroy existing mixnode keys.

If you are participating in the Nym incentives program, you can enter your Liquid address to receive your NYMPH tokens during `init` by using the `--incentives--address` flag:

```shell
./nym-mixnode init --id winston-smithnode --host $(curl ifconfig.me) --location YourCity --incentives-address YOURADDRESSHERE
```

### Run the mixnode

`./nym-mixnode run --id winston-smithnode`


You should see a nice clean startup: 

```
     | '_ \| | | | '_ \ _ \
     | | | | |_| | | | | | |
     |_| |_|\__, |_| |_| |_|
            |___/

             (mixnode - version 0.9.2)

    
Starting mixnode winston-smithnode...

Directory server [presence]: https://testnet-validator1.nymtech.net
Directory server [metrics]: https://metrics.nymtech.net
Listening for incoming packets on 167.70.75.75:1789
Announcing the following socket address: 167.70.75.75:1789
Public key: HHWAJ1zwpbb1uPLCvoTCUrtyUEuW9KKbUUnz3EUF1Xd9

 2020-05-05T16:01:07.802 INFO  nym_mixnode::node > Starting nym mixnode
 2020-05-05T16:01:08.135 INFO  nym_mixnode::node > Starting packet forwarder...
 2020-05-05T16:01:08.136 INFO  nym_mixnode::node > Starting metrics reporter...
 2020-05-05T16:01:08.136 INFO  nym_mixnode::node > Starting socket listener...
 2020-05-05T16:01:08.136 INFO  nym_mixnode::node > Starting presence notifier...
 2020-05-05T16:01:08.136 INFO  nym_mixnode::node > Finished nym mixnode startup procedure - it should now be able to receive mix traffic!
```

If everything worked, you'll see your node running at https://testnet-explorer.nymtech.net. 

Note that your node's public key is displayed during startup, you can use it to identify your node in the list.

Keep reading to find our more about configuration options or troubleshooting if you're having issues. There are also some tips for running on AWS and other cloud providers, some of which require minor additional setup.

{{% notice info %}}
If you run into trouble, please ask for help in the channel **nymtech.friends#general** on [KeyBase](https://keybase.io).
{{% /notice %}}

Have a look at the saved configuration files to see more configuration options.

### Set the ulimit

You **must** set your ulimit well above 1024 or your node won't work properly in the testnet. To test the `ulimit` of your mixnode:

```
cat /proc/$(pidof nym-mixnode)/limits | grep "Max open files" 
```

You'll get back the hard and soft limits, something like this: 

```Max open files            1024               1024               files```

If either value is 1024, you must raise the limit. To do so, execute this as root: 

```
echo "DefaultLimitNOFILE=65535" >> /etc/systemd/system.conf
```

Reboot your machine and restart your node. When it comes back, do `cat /proc/$(pidof nym-mixnode)/limits | grep "Max open files"`  again to make sure the limit has changed to 65535.

Changing the `DefaultLimitNOFILE` and rebooting should be all you need to do. But if you want to know what it is that you just did, read on.

Linux machines limit how many open files a user is allowed to have. This is called a `ulimit`.

`ulimit` is 1024 by default on most systems. It needs to be set higher, because mixnodes make and receive a lot of connections to other nodes.

#### Symptoms of ulimit problems

If you see any references to `Too many open files` in your logs:

```
Failed to accept incoming connection - Os { code: 24, kind: Other, message: "Too many open files" }
```

This means that the operating system is preventing network connections from being made. Raise your `ulimit`.


### Making a systemd startup script

Although it's not totally necessary, it's useful to have the mixnode automatically start at system boot time. Here's a systemd service file to do that:

```
[Unit]
Description=Nym Mixnode (0.9.2)

[Service]
User=nym
ExecStart=/home/nym/nym-mixnode run --id mix090
KillSignal=SIGINT # gracefully kill the process when stopping the service. Allows node to unregister cleanly.
Restart=on-failure
RestartSec=30
StartLimitInterval=350
StartLimitBurst=10

[Install]
WantedBy=multi-user.target
```

Put the above file onto your system at `/etc/systemd/system/nym-mixnode.service`. 

Change the path in `ExecStart` to point at your mixnode binary (`nym-mixnode`), and the `User` so it is the user you are running as.

If you have built nym on your server, and your username is `jetpanther`, then the start command might look like this: 

`ExecStart=/home/jetpanther/nym/target/release/nym-mixnode run --id your-id`. Basically, you want the full `/path/to/nym-mixnode run --id whatever-your-node-id-is`

Then run:

```
systemctl enable nym-mixnode.service
```

Start your node: 

```
service nym-mixnode start
```

This will cause your node to start at system boot time. If you restart your machine, the node will come back up automatically. 

You can also do `service nym-mixnode stop` or `service nym-mixnode restart`. 

Note: if you make any changes to your systemd script after you've enabled it, you will need to run: 

```
systemctl daemon-reload
```

This lets your operating system know it's ok to reload the service configuration.


### Checking that your node is mixing correctly

Once you've started your mixnode and it connects to the testnet validator, your node will automatically show up in the [Nym testnet explorer](https://testnet-explorer.nymtech.net).

The Nym network will periodically send two test packets through your node (one IPv4, one IPv6), to ensure that it's up and mixing. In the current version, this determines your node reputation over time (and if you're participating in the incentives program, it will set your node's reputation score). 

If your node is not mixing correctly, you will notice that its status is not green. Ensure that your node handles both IPv4 and IPv6 traffic, and that its public `--host` is set correctly. If you're running on cloud infrastructure, you may need to explicitly set the `--announce-host` (see below).

Nodes join the active mixing set once they have achieved a reputation score of 100 or above. 

### Viewing command help

See all available options by running:

```
./nym-mixnode --help
```

Subcommand help is also available, e.g.:

```
./nym-mixnode upgrade --help
```


### Node registration and de-registration

When your node starts up, it notifies the rest of the Nym network that it is up and ready to mix traffic. Token rewards will start as soon as registration has taken place. But if your node isn't mixing properly, you'll start to incur reputation penalties. 

If you run your node in a console window, it will register when it starts up, and un-register automatically when you hit `ctrl-c` to stop it. When your node is unregistered, it will not gain reputation, but it won't lose any either. 

If you kill it, though (`kill <process-number>`), the un-registration will not happen and you will incur reputation penalties. So, use `ctrl-c` instead.

Most people will want to run their mixnodes as a systemd service instead of in a console. Systemd scripts by default send `KillSignal=SIGTERM`, which kills the process non-gracefully, so that the un-registration doesn't happen. 

You **must** use `KillSignal=SIGINT` in your systemd scripts, under the `[Service]` block. This allows the un-registration code to run whenever your service is stopped. 

#### Manual unregister

Sometimes it's useful to move your node between servers. But the network won't allow you to start 2 nodes with the same keys in different locations.

If it's set up properly, your node should automatically unregister when you stop it. But in case it doesn't, you can unregister it manually:

```
./nym-mixnode unregister --id mix090  # substitute your node id here.
```

This takes your node out of the network. Reputation monitoring stops. You can then move your node between servers and restart it. Registration will happen automatically when you run it again.

### Virtual IPs, Google, AWS, and all that

On some services (e.g. AWS, Google), the machine's available bind address is not the same as the public IP address. In this case, bind `--host` to the local machine address returned by `ifconfig`, but also specify `--announce-host` with the public IP. Please make sure that you pass the correct, routable `--announce-host`.

For example, on a Google machine, you may see the following output from the `ifconfig` command:

```
ens4: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1460
        inet 10.126.5.7  netmask 255.255.255.255  broadcast 0.0.0.0
        ...
```

The `ens4` interface has the IP `10.126.5.7`. But this isn't the public IP of the machine, it's the IP of the machine on Google's internal network. Google uses virtual routing, so the public IP of this machine is something else, maybe `36.68.243.18`.

`nym-mixnode init --host 10.126.5.7`, inits the mixnode, but no packets will be routed because `10.126.5.7` is not on the public internet.

Trying `nym-mixnode init --host 36.68.243.18`, you'll get back a startup error saying `AddrNotAvailable`. This is because the mixnode doesn't know how to bind to a host that's not in the output of `ifconfig`.

The right thing to do in this situation is `nym-mixnode init --host 10.126.5.7 --announce-host 36.68.243.18`.

This will bind the mixnode to the available host `10.126.5.7`, but announce the mixnode's public IP to the directory server as `36.68.243.18`. It's up to you as a node operator to ensure that your public and private IPs match up properly.



### Mixnode Hardware Specs

For the moment, we haven't put a great amount of effort into optimizing concurrency to increase throughput. So don't bother provisioning a beastly server with many cores. 

* Processors: 2 cores are fine. Get the fastest CPUs you can afford. 
* RAM: Memory requirements are very low - typically a mixnode may use only a few hundred MB of RAM. 
* Disks: The mixnodes require no disk space beyond a few bytes for the configuration files

This will change when we get a chance to start doing performance optimizations in a more serious way. Sphinx packet decryption is CPU-bound, so once we optimise, more fast cores will be better.
