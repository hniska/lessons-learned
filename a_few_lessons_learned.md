# A few lessons learned

This document started out as a lessons learned around layered services architecture (LSA), but early on there was more leassons around things that arenn&#39;t directly related to LSA. This document contains things ranging from from high to low and can&#39;t promise a common thread throughout. Anyway, let&#39;s at least start with discussing LSA.

There are a few different reasons to why one chooses to use a layered service architecture (LSA) with NSO. Might be security boundaries, geographical boundaries, domain boundaries and probably the most common one, scale and performance. There might also be other technical reasons, like that a reload of NSO takes to long compared to the wanted package reload frequency, let's say the release cycle demands a reload every night then a CDB size of 30GB might not work.

Important to remember is that an LSA setup that is not running commit queues will be **slower** than a single NSO instance. And if the services arent working on a disjunct set of devices (i.e residing on different RFS instances) no speed gains will be had. For the use-cases that are not for scale and performance there aren&#39;t as many things to take into consideration.

Even if services aren&#39;t written with LSA in mind, not all hope is lost. The service can still be split in to a CFS and RFS layer. Not all the benefits of an LSA design will be there but that might be small price to pay compared to doing a total service redesign. For an example on how this can be done, look at examples.ncs/service-provider/mpls-vpn-layered-service-architecture.

## A decent LSA "ready" design

So, what do we mean with LSA ready? it&#39;s a design that even though you in the beginning only run it on a single NSO instance it is ready to be separated in to a CFS and an RFS layer.

Create the services as stacked services and have the lowest service only writing to a single device.

![l3vpn](/media/image1.png)

That way you can later split the service where the lowest service is transferred to an RFS node and later even transfer devices between RFS nodes (discussed below). There is of course a bit more to it like dispatch tables etc. But it&#39;s a start. When split in to the two layers it could look something like this

![l3vpn-lsa](/media/image2.png)

Lowest service layer (endpoint) writes to the device before moving to LSA. That layer is then moved to the RFS node and the l3vpn layer then writes to

```
/devices/device/RFS-node/config/dRFS/l3vpn-endpoint
```

dRFS = device RFS. A list of the devices in the RFS layer under which the services the device has been published.

The endpoint and the l3vpn-endpoint look the same so there are hardly any changes to the top l3vpn service when the endpoint is moved to the RFS layer.

If it&#39;s desirable to be able to move devices between RFS devices in the future, there are a few more things to consider.

- No allocations in the RFS layer
  - A lot more complex to move this together with the configs, usually allocations data is stored in a shared part of the NSO tree.
- Only one device per service instance in the RFS layer. Even if your RFS service is stacked, you can/should only have one device top to bottom (in the RFS layer).
- Moving a dRFS instance means moving all services and if the child services touch multiple devices it can be come arbitrary complex.

There is a good example of this in chapter 2 in the NSO LSA guide.

## All LSA setups

### No reading from CFS by the RFS layer

If the data is needed in a lower layer pass that data downstream. Otherwise it&#39;s just another thing to keep in sync with your code instead of letting NSO do what its good at.

### Time consuming things should be done asynchronously

CPU and time intensive things should be run asynchronously in a reactive FASTMAP loop (RFM). Have the create method release quickly and have it getting triggered again when the next state is reached.

## For speed and scalability

### Commit queues is a must

Unless the service code is really slow the slowest part of a commit will be the device. So, before LSA is considered commit queues should be enabled. Enabling commit-queues often yields such an increase in transactional throughput that LSA is not needed or its introduction can be postponed.

Without commit queues a layered service architecture will not give any speed gains. Commit speed will be the slowest device in the commit, it will even slower in an LSA setup than in a single NSO instance as latency within the LSA environment is higher than within a single NSO node.

When is it recommended to turn on commit queues?

The default transaction model in NSO locks the database from commit time till the slowest device in the transaction has accepted the configuration.

![normal-commit](/media/commit.jpg)

Commit queues loosen up the default atomicity of the transaction, the lock is released already when the configuration is saved in the CDB.

![queue-commit](/media/queuecommit.jpg)

Lets say a non commit-queue commit takes 10 seconds, if the expected peak throughput is 120 commits an hour, then roughly each commit has about 30 seconds to complete without interfering with other transactions. That gives 20 seconds to spare and most probably commit-queues is not necessary. Interfering meaning that the next transaction have to wait till the previous one has completed.

But let's say the peak is 360 transactions an hour, that would give each transaction exactly 10 seconds to complete without interfering with the next one and that is if they are absolutely evenly distributed. In this case commit-queues is a must. How many transactions can a single NSO handle with the commit-queue numbers above (600ms CDB lock time), in theory 6000 transactions an hour, of course this does not translate directly to any real world scenarios but at least gives a hint to how to think about commit-queues or not.

#### Commit queues and a few gotchas

- Think about failure handling
  - Remember that when a commit fails, NSO will be out of sync with the device
- A northbound API commit with commit sync will return 2xx Successful i.e. the request was successfully received, understood and accepted even though the actual writing to the device failed.
  - Northbound system would need to check the returned text too.
- Overlapping requests
  - If a commit queue item sets leaf X and a new commit queue item comes right after and sets X, the second queue item will fail with an overlapping request error.


### Reactive FASTMAP and commit queues

When a commit queue commit fails for a device the CDB will still have the intended configuration and be out of sync with the device. That poses a problem if your reactive FASTMAP loop has &quot;config on device&quot; as a criterion for progressing. If the service has such a criterion the progress subscriber (will be hard with kickers) will have to look at the service plan commit-queue list. To learn more about service plans check the &quot;Progress reporting using plan-data&quot; chapter in the development guide.

Snippet from tailf-ncs-plan.yang
```yang
  grouping plan-data {
    description
      "This grouping contains the plan data that can show the
       progress of a Reactive FASTMAP service. This grouping is optional
       and should only be used by services i.e lists or presence containers
       that uses the ncs:servicepoint callback";
    container plan {
      config false;
      tailf:cdb-oper {
        tailf:persistent true;
      }
      uses plan-components;
      container commit-queue {
        presence "The service is being committed through the commit queue.";
        list queue-item {
          key id;
          leaf id {
            type uint64;
            description
              "If the queue item in the commit queue refers to this service
               this is the queue number.";
          }
        }
      }
```
## Generally good practices

### **Makefile**

Make sure your project is reproducible in an easy way. A good start is to use a Makefile that sets up the NSO run-time environment for you project. With that you can easily just clean up your whole run-time and with one command setup everything needed to start your test. For an example Makefile take a look at examples.ncs/service-provider/mpls-vpn/Makefile, there it sets up the netsims, populate them with good startup data and also populate the CDB with some basic info.

This will also make it easier when having to show TAC on how to reproduce a bug.

### Containers (Docker)

One step further than using Makefiles is to go with containers.

#### How and why containers?

Directly from https://gitlab.com/nso-developer/nso-docker

There are many reasons for why Docker and containers in general might be good for you. The main drivers for using Docker with NSO lies around packaging, ensuring consistency in testing and production as well as making it simple and convenient to create test environments.

- build a docker image out of a specific version of NSO and your packages
  - distributed as one unit!
  - you test the combination of NSO version X and version Y of your packages
    - think of it as a &quot;version set&quot;
  - the same version set that is tested in CI is deployed in production
    - guarantees you tested same thing you deploy
  - conversely, using other distribution methods, you increase the risk of testing one thing and ending up deploying something else - i.e. you didn&#39;t really test what you use in production
- having NSO in a container makes it easy to start
  - simple to test
  - simple to run in CI
  - simple to use for development
- you do NOT need Kubernetes, Docker swarm or other fancy orchestration
  - run Docker engine on a single machine

It&#39;s also worth noting that using Docker does not mean you have to replace all of your current infrastructure. If you are currently using OpenStack or some system that primarily deals with virtual machines you don&#39;t have to rip this out. On the particular VM that runs NSO you can simply install Docker and have a single machine Docker environment that runs NSO. You are using Docker for the packaging features!

Yet another alternative is to use Docker for development and CI and when it&#39;s time to deploy to production you use something entirely different. Docker images are glorified tar files so it is possible to extract the relevant files from them and deploy by other means.

#### Setup of an example environment

Here you can find a good example on how to work with Docker containers https://gitlab.com/nso-developer/nso-docker

### Execution time

Test with data that is close to real world data, if a list will have 1000 entries in the production system make sure you test with more than that. Keep an eye on the execution time and try keeping the execution time to less than one second or something like 1ms per config line. Important here is commit time without the network as that is the part that we have power over. Of course, some service will break that, the important thing is that it&#39;s a known fact and doesn&#39;t show up as Jack in the box in production.

![transaction](/media/image3.png)

To test, a first step is to enable timecmd
```
admin@ncs# devtools true
admin@ncs# config
Entering configuration mode terminal
admin@ncs(config)# timecmd commit no-networking
% No modifications to commit.
Command executed in 0.00 sec
```

To see a more detailed report, do a &quot;| details&quot;, most important here is to keep an eye on the amount of time the lock is in place:

```
admin@ncs(config)# commit no-networking | details
 2019-11-07T12:13:52.527 applying transaction...
entering validate phase for running (tid=7600, session-id=538)...
 2019-11-07T12:13:52.528 run pre-trans-lock service callbacks...
 2019-11-07T12:13:52.528 run transforms and transaction hooks...
 2019-11-07T12:13:52.535 run transforms and transaction hooks done [0.007 sec]
 2019-11-07T12:13:52.535 pre-trans-lock service callbacks done [0.007 sec]
  **2019-11-07T12:13:52.535**** grabbing transaction lock**... ok [0.000 sec]
 2019-11-07T12:13:52.536 creating rollback file... ok [0.004 sec]
 2019-11-07T12:13:52.541 run transforms and transaction hooks...
 2019-11-07T12:13:52.897 run transforms and transaction hooks done [0.356 sec]
 2019-11-07T12:13:52.897 mark inactive... ok [0.009 sec]
 2019-11-07T12:13:52.908 pre validate... ok [0.009 sec]
 2019-11-07T12:13:52.918 run validation over the change set...
 2019-11-07T12:13:52.945 validation over the change set done [0.027 sec]
 2019-11-07T12:13:52.945 run dependency-triggered validation...
 2019-11-07T12:13:52.946 dependency-triggered validation done [0.000 sec]
 2019-11-07T12:13:52.946 check configuration policies...
 2019-11-07T12:13:52.946 configuration policies done [0.000 sec]
leaving validate phase for running (tid=7600, session-id=538) [0.419 sec]
entering write-start phase for running (tid=7600, session-id=538)...
 2019-11-07T12:13:52.947 cdb: write-start
leaving write-start phase for running (tid=7600, session-id=538) [0.021 sec]
entering prepare phase for running (tid=7600, session-id=538)...
 2019-11-07T12:13:52.968 cdb: prepare
 2019-11-07T12:13:52.968 ncs-internal-device-mgr: prepare
leaving prepare phase for running (tid=7600, session-id=538) [0.005 sec]
entering commit phase for running (tid=7600, session-id=538)...
 2019-11-07T12:13:52.973 cdb: commit
 2019-11-07T12:13:53.014 cdb: delivering commit subscription notifications at prio 1
 2019-11-07T12:13:53.017 cdb: all commit subscription notifications acknowledged
 2019-11-07T12:13:53.017 ncs-internal-service-mux: commit
 2019-11-07T12:13:53.017 ncs-internal-device-mgr: commit
  **2019-11-07T12:13:53.023**** releasing transaction lock**
leaving commit phase for running (tid=7600, session-id=538) [0.049 sec]
 2019-11-07T12:13:53.023 applying transaction done [0.495 sec]
```

To dig deeper use the NSO progress trace (Chapter 17 in the developer guide). In there you get even more details, like how much time is spent with the reverse-diff.
```
TIMESTAMP   TID SESSION ID  CONTEXT SUBSYSTEM   PHASE   SERVICE SERVICE PHASE   DURATION    MESSAGE
2019-11-04T15:55:40.938 1587    67  cli ncs     /reverse[name=&#39;1&#39;]  pre-modification        service pre-modification...
2019-11-04T15:55:40.939 1587    67  cli ncs     /reverse[name=&#39;1&#39;]  pre-modification    0.000   service pre-modification ok
2019-11-04T15:55:40.939 1587    67  cli ncs     /reverse[name=&#39;1&#39;]          applying FASTMAP reverse diff-set...
2019-11-04T15:55:41.294 1587    67  cli ncs     /reverse[name=&#39;1&#39;]      0.354   applying FASTMAP reverse diff-set ok
2019-11-04T15:55:41.392 1587    67  cli ncs     /reverse[name=&#39;1&#39;]  create      service create...
2019-11-04T15:55:46.536 1587    67  cli ncs     /reverse[name=&#39;1&#39;]  create  5.144   service create ok
2019-11-04T15:55:46.537 1587    67  cli ncs     /reverse[name=&#39;1&#39;]          saving FASTMAP reverse diff-set and applying changes...
2019-11-04T15:55:50.865 1587    67  cli ncs     /reverse[name=&#39;1&#39;]      4.327   saving FASTMAP reverse diff-set and applying changes ok
2019-11-04T15:55:50.867 1587    67  cli ncs     /reverse[name=&#39;1&#39;]  post-modification       service post-modification...
2019-11-04T15:55:50.867 1587    67  cli ncs     /reverse[name=&#39;1&#39;]  post-modification   0.000   service post-modification ok
2019-11-04T15:55:50.887 1587    67  cli                 9.950   run transforms and transaction hooks done
```

### Large lists in services

A quite common problem is that the service creation time in the beginning is within reason but when you get up into higher number of list entries in the service the execution time starts to get to unacceptable number. Here is an example, you have a service that creates policy-maps, when it creates just one policy-map it&#39;s quick and responsive

![policy-service](/media/image4.png)

Instantiated

![policy-service-instances](/media/image5.png)

```
admin@ncs% timecmd commit dry-run
cli {
    local-node {
        data  devices {
                  device ce0 {
                      config {
             +            ios:policy-map One0 {
             +                class class-default {
             +                    shape {
             +                        average {
             +                            bit-rate 300000;
             +                        }
             +                    }
             +                }
             +            }
                      }
                  }
              }
Command executed in 0.09 sec
```
But after a while when that instance already has 1000 policy-maps and you want to add just one more policy-map it takes a long time

```
admin@ncs% timecmd commit dry-run
cli {
    local-node {
        data  devices {
                  device ce0 {
                      config {
             +            ios:policy-map **One1001** {
             +                class class-default {
             +                    shape {
             +                        average {
             +                            bit-rate 300000;
             +                        }
             +                    }
             +                }
             +            }
                      }
                  }
              }
Command executed in 7.25 sec
```

Take a look in the progress trace

| SERVICE PHASE | DURATION | MESSAGE |
| --- | --- | --- |
| pre-modification |   | service pre-modification... |
| pre-modification | 0.000 | service pre-modification ok |
|   |   | applying FASTMAP reverse diff-set... |
|   | 0.240 | applying FASTMAP reverse diff-set ok |
| create |   | service create... |
| create | 2.827 | service create ok |
|   |   | saving FASTMAP reverse diff-set and applying changes... |
|   | 3.835 | saving FASTMAP reverse diff-set and applying changes ok |
| post-modification |   | service post-modification... |
| post-modification | 0.000 | service post-modification ok |

The service spent 3.835 seconds just with the reverse diff-set. What is the reverse diff-set? It is what NSO uses to undo the service, it&#39;s the reverse of what the service created. To see the reverse diff-set for a service use the get-modifications reverse command

```
admin@ncs(config)# path to the service get-modifications reverse
```

What can be done if you end up here? One thing could be to split the service to a stacked service so that each policy-map is a child service. Then when adding a new instance of a policy-map NSO doesn&#39;t need to run the full reverse diff-set

![policy-stacked](/media/image6.png)

Instantiated

![policy-stacked-instance](/media/image7.png)

```
admin@ncs% timecmd commit dry-run
cli {
    local-node {
        data  devices {
                  device ce0 {
                      config {
             +            ios:policy-map Two1000 {
             +                class class-default {
             +                    shape {
             +                        average {
             +                            bit-rate 300000;
             +                        }
             +                    }
             +                }
             +            }
                      }
                  }
              }
```

| SERVICE PHASE | DURATION | MESSAGE |
| --- | --- | --- |
| pre-modification |   | service pre-modification... |
| pre-modification | 0.000 | service pre-modification ok |
| create |   | service create... |
| create | 0.010 | service create ok |
|   |   | saving FASTMAP reverse diff-set and applying changes... |
|   | 0.004 | saving FASTMAP reverse diff-set and applying changes ok |
| post-modification |   | service post-modification... |
| post-modification | 0.000 | service post-modification ok |

### XPATH

XPath is wonderful but sometimes hard to get right and it can be especially hard to find slow XPath queries in your service. The query runs OK on the small developer network but as soon as its loaded with X thousands of devices the service is slow, which one of the 100 XPath:s is it that is the culprit? Once again progress trace to the rescue. Tracing in debug verbosity XPath evaluation time is included

SUBSYSTEM   DURATION    MESSAGE

| xpath |   | evaluating: /devices/device[name=&#39;pe3&#39;]/device-type: ../remote-node or netconf or generic or cli or snmp |
| --- | --- | --- |
| xpath |   | get\_elem(/devices/device[name=&#39;pe3&#39;]/remote-node) = not\_found |
| xpath | 0.019 | result: /devices/device[name=&#39;pe3&#39;]/device-type: true |
| xpath |   | evaluating: /devices/device[name=&#39;pe3&#39;]/device-type: not(boolean(../remote-node) and boolean(./\*/ned-id)) |
| xpath |   | get\_elem(/devices/device[name=&#39;pe3&#39;]/remote-node) = not\_found |
| xpath | 0.024 | result: /devices/device[name=&#39;pe3&#39;]/device-type: true |
| xpath |   | evaluating: /devices/device[name=&#39;pe3&#39;]/device-type/cli: ../../authgroup |
| xpath |   | get\_elem(/devices/device[name=&#39;pe3&#39;]/authgroup) = default |
| xpath | 0.042 | result: /devices/device[name=&#39;pe3&#39;]/device-type/cli: true |

### Service back pointer list

Keep an eye on the service back pointer list

```
admin@ncs(config)# show full-configuration devices device ce1 config ios:policy-map | display service-meta-data
devices device ce1
 config
  ! Refcount: 1
  ! Backpointer: [/l3vpn:vpn/l3vpn:l3vpn[l3vpn:name=&#39;volvo&#39;] ]
  ios:policy-map volvo
   ! Refcount: 1
   ! Backpointer: [/l3vpn:vpn/l3vpn:l3vpn[l3vpn:name=&#39;volvo&#39;] ]
   class class-default
    ! Refcount: 1
    shape average 6000000
```

If that list gets to big it will also slow down the service creation. Try to understand if this can be changed, if all 1000 service instance creates this value do, we really expect that to be deleted when all service instances are deleted? Will they ever be deleted? If not, why not just make sure they are there by setting them in a day0 service that will be applied once on each device. Another approach is to set it in the service pre\_modification.

A few nonscientific numbers (tested on small laptop with rollbacks etc) arounds this.

A specific service type always creates:
```
admin@ncs(config)# show full-configuration devices device c0 config ios:router
devices device c0
config
  ios:router bgp 64512
```

If that list entry (bgp 64512) is created **without** back pointers

| Nr of service instances | Seconds |
| --- | --- |
| 1000 | 13 |
| 5000 | 69 |
| 10000 | 154 |

And the same but **with** back pointers

| Nr of services (also nr of back pointers) | Seconds |
| --- | --- |
| 1000 | 13 |
| 5000 | 100 |
| 10000 | 265 |

### CDB locks

Remember that if the CDB is locked when you call NSO your commit (and some actions) will be aborted, probably mostly a problem when doing OSS/BSS integrations to NSO via any of the northbound API:s. Can be mitigated with

- commit wait-for
- sync-from wait-for-lock

### Commit flags

Nice to know commit flags for LSA

- commit lso no-networking
  - Commits down to the RFS layer but not to devices
- commit no-lso no-networking
  - Commits stays in the CFS layer

### Sync-from is expensive

Sync-from is expensive so never do a sync-from from your services. Sync-from should only be done when absolutely necessary. If the devices are being configured by other systems than NSO and end up out-of-sync frequently the recommendation is to use the no-overwrite flag when committing.

```
admin@ncs(config)# commit no-overwrite
```
Commit no-overwrite checks only the configuration that corresponds with the current commit, instead of the whole device configuration. If the device is still out-of-sync use partial sync-from for remediation

```
ncs# devices partial-sync-from path [ \
/devices/device[name='ex0']/config/r:sys/interfaces/interface[name='eth0'] \
/devices/device[name='ex1']/config/r:sys/dns/server ]
sync-result {
    device ex0
    result true
}
sync-result {
    device ex1
    result true
}
```

### Alarms

In a production NSO system its important to keep an eye on the NSO alarms. NSO generates alarms for serious problems that must be remedied. Alarms are available over all north-bound interfaces and they exist at the path /alarms.
