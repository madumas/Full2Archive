# Full2Archive

This repository contains instructions to sync a full node and transform it into an archive node.

Parity and geth client software do not provide a functionality out of to box to switch a full node to an archive node,
 so the trick that we are using is to first sync a full node and then
 create on the same server a second node, set up in archival mode.
 
 We'll be using docker and Parity to do this, but the instructions could be easily
 adapted to your favorite Ethereum client and container or VM technology.
 
##Step 1: Create the full node 

1. First we create a local storage volume so data gets persistent if we delete and reconfigure our container 
 ```
 docker volume create fullnode --driver local --opt type=none --opt o=bind --opt device=/data/dockervolumes/fullnode
 ```
2. Then we create a simple network, which by default will be open to the world. This is the network interface our node 
will use to connect to the Ethereum network 
 ```
 docker network create nodenet
```

3. Lastly we launch the full node  
```
docker run -d --name full --mount type=volume,source=fullnode,target=/storage --network nodenet -d parity/parity:stable --base-path /storage
```
Our node is name `full` and is connected to our `nodenet` network. This will yield something like this in the logs: 
```
2019-03-13 00:19:37 UTC Starting Parity-Ethereum/v2.3.5-stable-ebd0fd0-20190227/x86_64-linux-gnu/rustc1.32.0
2019-03-13 00:19:37 UTC Keys path /storage/keys/ethereum
2019-03-13 00:19:37 UTC DB path /storage/chains/ethereum/db/906a34e69aec8c0d
2019-03-13 00:19:37 UTC State DB configuration: fast
2019-03-13 00:19:37 UTC Operating mode: active
2019-03-13 00:19:37 UTC Configured for Ethereum using Ethash engine
2019-03-13 00:19:38 UTC Updated conversion rate to Ξ1 = US$134.21 (35480990 wei/gas)
2019-03-13 00:19:42 UTC Syncing snapshot 0/2798        #0    4/25 peers      8 KiB chain    3 MiB db  0 bytes queue   10 KiB sync  RPC:  0 conn,    0 req/s,    0 µs
2019-03-13 00:19:42 UTC Public node URL: enode://enodeid@172.18.0.2:30303
2019-03-13 00:19:47 UTC Syncing snapshot 12/2798        #0    4/25 peers      8 KiB chain    3 MiB db  0 bytes queue   10 KiB sync  RPC:  0 conn,    0 req/s,    0 µs
```
The `Syncing snapshot` lines are indicating that Parity is using warp to quickly sync. A few hours
later, you will see the first `Imported` message indicating that the node is fully synced.

```
2019-03-13 01:48:12 UTC Syncing #7357841 0x5e26…cdc0     4.20 blk/s  347.4 tx/s   25.7 Mgas/s      0+   34 Qed  #7357881   20/25 peers      5 MiB chain  100 MiB db    3 MiB queue   16 MiB sync  RPC:  0 conn,    0 req/s,    0 µs
2019-03-13 01:48:17 UTC Syncing #7357856 0x6443…8685     2.99 blk/s  326.5 tx/s   22.7 Mgas/s      0+   22 Qed  #7357881   20/25 peers      5 MiB chain  101 MiB db    1 MiB queue   16 MiB sync  RPC:  0 conn,    0 req/s,    0 µs
2019-03-13 01:48:25 UTC Imported #7357881 0x0fc1…3531 (19 txs, 7.96 Mgas, 2142 ms, 8.96 KiB) + another 9 block(s) containing 651 tx(s)
2019-03-13 01:48:25 UTC Updated conversion rate to Ξ1 = US$133.86 (35573770 wei/gas)
2019-03-13 01:48:30 UTC Imported #7357882 0x530d…0d3d (137 txs, 7.99 Mgas, 510 ms, 27.65 KiB)
```

## Step 2: Create the archive node
1. We create a special network with no outside routing that will connect our archive node and our full node
```
docker network create internal --internal
```
2. We connect this network to our full node
```
docker network connect internal full
```
3. Like for the full node, we create a persistent local volume for our container
```
docker volume create archnode -d local -o o=bind -o device=/data/dockervolumes/archnode
```
4. And we launch our archival node
```
docker run -d --name arch --mount type=volume,source=archnode,target=/storage --network internal -d parity/parity:stable --base-path /storage --bootnodes enode://nodeid@172.19.0.2:30303 --pruning archive  --no-periodic-snapshot --cache-size-db 6000 --cache-size-state 1000 --scale-verifiers --num-verifiers 16
```
Let's go over the over the parameters.

docker params:

`--name arch`: the container is named "arch"

`--network internal`: connect only to the "internal" network

parity params:

`--bootnodes enode://nodeid@172.19.0.2:30303`: tell parity to connect to our full node. The nodeid can be found
in the logs of the full node.
 
`--pruning archive`: run an archival node
   
`--no-periodic-snapshot`: do not create snapshots to provide to other clients on the Ethereum client. This process
is very slow and ressource intensive on an archive node.
  
`--cache-size-db 6000`: provide 6000MB of cache to the database used by parity, rocksdb. 
   
`--cache-size-state 1000`: provide 1000MB for parity state cache
    
`--scale-verifiers`: tell parity to increase the number of verifiers if necessary. I do not think this has an effect
on sync performance
    
`--num-verifiers 16`: start with 16 verifiers. Again, I don't think it makes a difference on sync time.


 Of the above parameters, cache size is the one that help the most with syncing. Provide as much cache as you can
 but keep an healthy amount of free space for linux cache and buffers.
 
 Our node will start syncing:
 
```
2019-03-14 11:04:21 UTC Starting Parity-Ethereum/v2.3.5-stable-ebd0fd0-20190227/x86_64-linux-gnu/rustc1.32.0
2019-03-14 11:04:21 UTC Keys path /storage/keys/ethereum
2019-03-14 11:04:21 UTC DB path /storage/chains/ethereum/db/906a34e69aec8c0d
2019-03-14 11:04:21 UTC State DB configuration: archive
2019-03-14 11:04:21 UTC Operating mode: active
2019-03-14 11:04:21 UTC Warning: Warp Sync is disabled because of non-default pruning mode.
2019-03-14 11:04:21 UTC Configured for Ethereum using Ethash engine
2019-03-14 11:04:26 UTC Failed to auto-update latest ETH price: Fetch(Timeout)
2019-03-14 11:04:26 UTC Public node URL: enode://archnodeid@172.19.0.3:30303
2019-03-14 11:04:26 UTC Syncing    #3248 0xaf0b…5eab   641.26 blk/s    0.0 tx/s    0.0 Mgas/s    685+    0 Qed     #3937    1/25 peers      3 MiB chain   79 KiB db    1 MiB queue  785 KiB sync  RPC:  0 conn,    0 req/s,    0 µs
```
