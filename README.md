# Transfer orphaned messages from DR dc to Main dc

The objective of this gist is to provide an script that transfer orphaned messages left
in a RabbitMQ cluster (`OKR`) in the DR data center (`LTM`) back to the
same RabbitMQ cluster (`OKR`) but in the Main data center (`PC1`).

In a nutshell, this is the set up :
  - We have two data centers. The Main dc called `PC1` and DR dc called `LTM`
    ```
       PC1 DC                   LTM DC
     -----------------      -------------

    ```

  - We have a RabbitMQ cluster called `OKR` in both data centers
    ```
       PC1 DC                   LTM DC
     -----------------      -------------



      OKR RMQ cluster         OKR RMQ cluster

    ```

  - Both `OKR` clusters have a federation queue link with an upstream RabbitMQ cluster called `ART`. This federation link forwards messages from `ART` cluster to the `OKR` cluster which has consumer applications.
    > We have skipped the exact details of how messages are forwarded. Check out RabbitMQ federation docs if you want to know more about it.


    ```
       PC1 DC                   LTM DC
     -----------------      -------------
      ART RMQ cluster ----------+
             |                  |    
             | federation       |
            \|/                \|/
      OKR RMQ cluster         OKR RMQ cluster

    ```
  - Applications access `OKR` clusters via a Load Balancer (`LB`) which points to the `OKR` cluster in the active dc. Typically, the active dc is `PC1`
    ```
       PC1 DC                   LTM DC
     -----------------      -------------
      ART RMQ cluster ----------+
             |                  |    
             | federation       |
            \|/                \|/
      OKR RMQ cluster         OKR RMQ cluster
          |                          |
          +--------------+--/ -------+
                         |
        Apps----------> [LB]  LB connects to OKR in PC1 (Main dc) or LTM (DR dc)

    ```
  - While applications are connected to `OKR` cluster in `PC1`, there are no consumers in `OKR` cluster in `LTM` dc. This means that messages will never flow from `ART` cluster to `OKR` cluster in `LTM` dc. Only to `PC1` where applications are connected.


## Switch to DR site
This is what happens when the Load Balancer is switched from `PC1` to `LTM` dc:
  - Applications are disconnected from `OKR` in `PC1` dc
  - Applications connect to `OKR` in `LTM` dc
  - Messages will flow now from `ART` cluster to `OKR` cluster in `LTM` dc
    > It is out of scope how this guide to explain how that works

## Switch back to Main site
This is what happens when the Load Balancer switches back to `PC1` dc:
  - Applications are disconnected from `OKR` in `LTM` dc
  - Applications connect to `OKR` in `PC1` dc
  - Messages will flow now from `ART` cluster to `OKR` cluster in `LTM` dc
  - Any message produced and not consumed in `OKR` in `LTM` will stay there. These are what we call the orphaned messages.
  - We cannot shovel the messages from `LTM` to `PC1` because that would trigger the flow of messages from `AKT` cluster down to `OKR` in `LTM`. This is because shovel creates a local AMQP consumer which automatically triggers the flow of messages from `AKT`.

The proposed solution is to transfer those orphaned messages while preventing incoming messages from `AKT`. The script performs the following operations at the cluster level, i.e. across all vhosts:
  - Disable the federation upstreams (with `AKT`). It disconnects the cluster from the fictitious `AKT` cluster .
    > All federation policies remain unaffected. Once we restore the federation upstream, the policies resume their operations.
    > To disable a federation upstream, we clone it with a different name therefore it is not used because there are no policies referring to it. And we delete it so that we cut off the federation link with `AKT` cluster

  - Transfer messages, using shovel, to `OKR` cluster in `PC1`
    > It will only transfer messages for queues which have a federation policy. Other queues remain as they are.

  - Enable the federation upstreams back again

# Demonstration

## Prerequisites

- Docker installed. We will use Docker to deploy 2 standalone nodes representing `OKR` cluster in the two data centers.
- `curl` or similar tools to run some queries against the management ui

## 1. Deploy OKR clusters for LTM and PC1 data centers

Run the following script to deploy 2 standalone RabbitMQ nodes representing clusters `OKR` in `LTM` and `PC1` data centers.
```
$ rabbit/deploy
```
> We can safely re-run this script. It will always delete the current RabbitMQ nodes before launching new ones.

Once both nodes are running, verify that they are reachable on their respective ports:
```
$ curl -s -u guest:guest localhost:15672/api/overview | jq .cluster_name
"pc1-okr"
$ curl -s -u guest:guest localhost:15673/api/overview | jq .cluster_name
"ltm-okr"
```

## 2. Create queues, policies and upstreams in both clusters with messages only on LTM-OKR cluster

First of all, we need to create the federation upstream to the fictitious `akt` cluster, the federation policy (`federate-with-akt`) that federates queues with the federation upstream, the queues (`perf-test-1` to `perf-test-10`) and 10K messages on each queue.

To create all the above in the default vhost, run the following script:
```
./setup_vhost_in_both_clusters
```
> In occasions, RabbitMq PerfTest does not produce exactly the number of messages we indicate (10K messages). It is irrelevant. What it is relevant is that we transfer exactly the total count of produced messages in LTM-OKR cluster

Additionally, we may create all the above in other vhosts so that we can simulate more complex scenarios which involve multiple vhosts not just one:
```
VHOST=vhost-1 ./setup_vhost_in_both_clusters
```


## 3. Check queues in RabbitMQ Management UI in the LTM-OKR cluster

Second, we are going to wait until we see in the [RabbitMQ Management UI](http://localhost:15673/#/queues) that the queues have 10K messages each. You can also check that the `akt` upstreams is declared and all 10 queues have a policy applied.
> the upstream will not be connected yet because there is no akt cluster/node running. But it is enough to demonstrate that we have
an upstream

## 4. Transfer orphaned messages from LTM-OKR TO PC1-OKR

We start the transfer of orphaned message by invoking the command below. But first we need to declare 3 environment variables:
- `HTTP_SOURCE_CLUSTER` [**Required**] This is the http url of the source cluster (the one with the messages).
    In our case, it is `http://localhost:15673`

- `HTTP_SOURCE_CREDENTIALS` [**Required**] These are the http credentials in the source cluster in the form of `username:password`. In our case it is `guest:guest`.

- `AMQP_TARGET_CLUSTER` [**Required**] This is the AMQP url of the target cluster (where we want to transfer messages to)
    In our case, it is `amqp://pc1-okr-rabbitmq`. Both RabbitMQ nodes have a name in Docker. This AMQP url is used by one node to connect to the other. Hence we are going to use their own DNS names assigned by us when we deployed them.

- `MAX_ACTIVE_SHOVEL_COUNT` [**Optional**, default value is **50**] This is the maximum number of shovels this script will schedule at any time. For instance, assuming a default value of 50 and 1000 non-empty federated queues, the script will initially create 50 shovels. As queues are emptied, their corresponding shovels are deleted and new shovels are created. At most there will be up to 50 shovels running.  

- `TARGET_QUEUE` [**Optional**] Name of the target queue. *This is meant for testing purposes only*. It allows us to use the same cluster as source and target cluster. If you do not specify this environment variable, the target queue has the same name as the source queue. In our case, because we have two distinct clusters, we do not need to use this feature.

We can invoke the script by specifying the environment variables in-line:
```
HTTP_SOURCE_CLUSTER="localhost:15673" \
  HTTP_SOURCE_CREDENTIALS="guest:guest" \
  AMQP_TARGET_CLUSTER="amqp://pc1-okr-rabbitmq" \
  MAX_ACTIVE_SHOVEL_COUNT=5 \
  ./transfer

```

> With `MAX_ACTIVE_SHOVEL_COUNT=5`, there will be at most 5 running shovels

or by exporting them first:
```
export HTTP_SOURCE_CLUSTER="localhost:15673"
export HTTP_SOURCE_CREDENTIALS="guest:guest"
export AMQP_TARGET_CLUSTER="amqp://pc1-okr-rabbitmq"
export MAX_ACTIVE_SHOVEL_COUNT=5
./transfer
```

It will first print out this output. It detects there are 10 non-empty federated queues. But it only schedules transfer for 5 of them given the configured maximum is 5:
```
>>>> Transfer started at Wed Jul 17 09:38:15 CEST 2019

> Temporary files generated under /gist/75Sq
There are orphaned messages to transfer
Disable federation upstreams
  Clone federation upstreams
    Successfully cloned federation upstreams
  Cancel Federation upstreams
    Succesfully deleted upstream akt
Initiate transfer of orphaned messages
Currently there are 0 active shovels. Scheduling 5 shovels out of 10.
    # Queue "perf-test-1" with 2162 msg (s) on vhost "/"
  Succesfully created shovel transfer-perf-test-1
    # Queue "perf-test-10" with 10000 msg (s) on vhost "/"
  Succesfully created shovel transfer-perf-test-10
    # Queue "perf-test-2" with 10000 msg (s) on vhost "/"
  Succesfully created shovel transfer-perf-test-2
    # Queue "perf-test-3" with 10000 msg (s) on vhost "/"
  Succesfully created shovel transfer-perf-test-3
    # Queue "perf-test-4" with 10000 msg (s) on vhost "/"
  Succesfully created shovel transfer-perf-test-4

```

In the next 10sec, it detects that 5 empty queues with shovels hence it deletes them.  It also detects that there are still 5 non-empty federated queues.
```
>>>> Waiting 10sec before checking if transfer has completed (attempt #1)

> Temporary files generated under /gist/NoOe
Transfer of orphaned messages has completed for some queues
Terminate transfer of orphaned messages
 Succesfully deleted shovel transfer-perf-test-1
 Succesfully deleted shovel transfer-perf-test-10
 Succesfully deleted shovel transfer-perf-test-2
 Succesfully deleted shovel transfer-perf-test-3
 Succesfully deleted shovel transfer-perf-test-4
There are orphaned messages to transfer
Disable federation upstreams
Initiate transfer of orphaned messages
Currently there are 0 active shovels. Scheduling 5 shovels out of 5.
  # Queue "perf-test-5" with 10000 msg (s) on vhost "/"
  Succesfully created shovel transfer-perf-test-5
  # Queue "perf-test-6" with 10000 msg (s) on vhost "/"
  Succesfully created shovel transfer-perf-test-6
  # Queue "perf-test-7" with 10000 msg (s) on vhost "/"
  Succesfully created shovel transfer-perf-test-7
  # Queue "perf-test-8" with 10000 msg (s) on vhost "/"
  Succesfully created shovel transfer-perf-test-8
  # Queue "perf-test-9" with 10000 msg (s) on vhost "/"
  Succesfully created shovel transfer-perf-test-9

```

In the last 10sec(s), it detects that there are no more messages and it prints outs:
```
>>>> Waiting 10sec before checking if transfer has completed (attempt #8)

> Temporary files generated under /gist/ZhLi
There are no orphaned messages to transfer
Terminate transfer of orphaned messages
Succesfully deleted shovel transfer-perf-test-5
 Succesfully deleted shovel transfer-perf-test-6
 Succesfully deleted shovel transfer-perf-test-7
 Succesfully deleted shovel transfer-perf-test-8
 Succesfully deleted shovel transfer-perf-test-9
Enable Federation upstreams
 Successfully restored upstreams
 Succesfully deleted upstream akt-to-restore
```

## 5. Teardown clusters or re-run the demonstration again

To tear down the cluster just run:
```
$ rabbit/undeploy
```

If you want to run the demonstration again, all you need to do repeat the same commands right from the start:
```
$ rabbit/deploy
$ ./setup_vhost_in_both_clusters
$ VHOST=vhost-1 ./setup_vhost_in_both_clusters
$ ./transfer
```

## About the `_idempotent_transfer` script invoked by the `transfer` script

**TL;DR** Users should never be bothered with `_idempotent_transfer`. Users instead should use `transfer`.

`_idempotent_transfer` is an idempotent script that we can call it as many times as we want. It only transfers orphaned messages from queues which have a federation policy applied.
  > It is important that our script is resilient to failures. In other words, we can call it as many times as needed and from where it is necessary too.
  >
  > It is also very important that we do not rely on local storage like file systems or even memory to store RabbitMQ object definitions (such as federation upstreams). Instead we use RabbitMQ itself. Should the machine from where we run the script failed, we can run it from another machine without loosing any data.

The script terminates successfully if there are no orphaned messages to transfer. In the contrary, if there are orphaned messages, it first disables all federation upstreams. It does it by cloning the federation upstream parameter with a different name and deleting the original one. Therefore we won't loose it.

Finally, it creates one shovel per queue, if it has not being created yet, before terminating unsuccessfully.
It is up to the `transfer` script to repeatedly call it until it successfully terminates.

`_idempotent_transfer` will only terminate successfully when there are no more orphaned messages. But before it terminates, it enables the federation upstream and deletes all the shovels.
