# P4Runtime

Find documentation under [docs](docs/).

## Tentative gNMI support with sysrepo

We are working on supporting gNMI and OpenConfig YANG models as part of the P4
Runtime server. We are using [sysrepo](https://github.com/sysrepo/sysrepo) as
our YANG configuration data store and operational state manager. If you want to
experiment with the gNMI support, you will need to install sysrepo and its
dependencies. We currently recommend using [version 0.7.1 of
sysrepo](https://github.com/sysrepo/sysrepo/releases/tag/v0.7.1), which depends
on [version 0.13-rc2 of
libyang](https://github.com/CESNET/libyang/releases/tag/v0.13-r2).

Please make sure you install all the dependencies for libyang and sysrepo. If
you are using a Debian system, we recommend that you install the following
packages:

    build-essential cmake libpcre3-dev libavl-dev libev-dev libprotobuf-c-dev protobuf-c-compiler

Then install libyang:

    git clone https://github.com/CESNET/libyang.git
    cd libyang
    git checkout v0.13-r2
    mkdir build
    cd build
    cmake ..
    make
    [sudo] make install

Finally, install sysrepo

    git clone https://github.com/sysrepo/sysrepo.git
    cd sysrepo
    git checkout v0.7.1
    mkdir build
    cd build
    cmake -DBUILD_EXAMPLES=Off -DCALL_TARGET_BINS_DIRECTLY=Off ..
    make
    [sudo] make install

You can now start the sysrepo daemon (`sudo sysrepod -d` to start in debug
mode).

In order to experiment with gNMI support, you also need to make sure to use
`--with-sysrepo` when running `configure` for this project.

After installing sysrepo and building this project, the final step is to load
the appropriate YANG models into sysrepo. Note that for now ***we are only
looking to support a very small subset of openconfig-interfaces***. We provide a
script that you can run to load the YANG models: `sudo
sysrepo/install_yangs.sh`. You can check that the YANG models were installed
properly with `sysrepoctl -l`. The output should look like this:

```
Sysrepo schema directory: /etc/sysrepo/yang/
Sysrepo data directory:   /etc/sysrepo/data/
(Do not alter contents of these directories manually)

Module Name                   | Revision   | Conformance | Data Owner          | Permissions | Submodules                    | Enabled Features
-----------------------------------------------------------------------------------------------------------------------------------------------
ietf-interfaces               | 2014-05-08 | Installed   | root:root           | 666         |                               |
openconfig-extensions         | 2017-04-11 | Installed   |                     |             |                               |
openconfig-types              | 2017-08-16 | Installed   |                     |             |                               |
openconfig-yang-types         | 2017-07-30 | Installed   |                     |             |                               |
openconfig-interfaces         | 2017-07-14 | Installed   | root:root           | 666         |                               |
iana-if-type                  | 2014-05-08 | Installed   |                     |             |                               |
ietf-inet-types               | 2013-07-15 | Installed   |                     |             |                               |
ietf-netconf-acm              | 2012-02-22 | Installed   | root:root           | 666         |                               |
ietf-netconf                  | 2011-06-01 | Installed   | root:root           | 666         |                               |
ietf-netconf-notifications    | 2012-02-06 | Installed   | root:root           | 666         |                               |
```

The P4 Runtile server library that you get after that will be able to support
gNMI subscriptions (`ONCE` mode only) for the openconfig-interfaces model. Here
is an example of a supported subscription request from a Python client:
```
channel = grpc.insecure_channel(<SERVER_ADDR>)
stub = gnmi_pb2.gNMIStub(channel)

def req_iterator():
    while True:
        req = gnmi_pb2.SubscribeRequest()
        subList = req.subscribe
        subList.mode = gnmi_pb2.SubscriptionList.ONCE
        sub = subList.subscription.add()
        path = sub.path
        for name in ["interfaces", "interface", "..."]:
            e = path.elem.add()
            e.name = name
        print "***************************"
        print "REQUEST"
        print req
        print "***************************"
        yield req
        return

for response in stub.Subscribe(req_iterator()):
    print "***************************"
    print "RESPONSE"
    print response
    print "***************************"
```
