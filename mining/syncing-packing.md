---
description: >-
  A guide to syncing and packing
---

# Syncing and Packing

One of the first things you'll do when mining is sync and pack some or all of the weave data.

## Syncing

"Syncing" refers to the process of downloading data from network peers. When you launch your
miner you'll configure a set of `storage_module`s that cover some or all of the weave. Your
node will continuously check for any gaps in your configured data and then search out peers to
download the missing data.

## Packing

Storage modules can be either "unpacked" (e.g. `storage_module 16,unpacked`) or "packed"
(e.g. `storage_module 16,En2eqsVJARnTVOSh723PBXAKGmKgrGSjQ2YIGwE_ZRI`). Before you can mine
data it must be packed to your mining address. There are actually two symetric operations that
fall under the "packing" umbrella:

1. `pack` - Symmetrically encrypt a chunk of data with your mining address.
2. `unpack` - Decrypt a packed chunk of data.

Both operations consume roughly the same amount of computational power. 

**Note:** You will almost always have to **unpack** data when syncing it. Whatever
peer you sync the data from will likely return it to you packed to its own address. Before
you can do anything with it you will first need to unpack. You may then have to pack it to your
own address. i.e. each chunk of data synced will usually need 1-2 packing operations.

## Storage Module Format

Each storage module has 2 directories `chunk_storage` and `rocksdb`.

### `chunk_storage`

This directory stores the actual weave data in a series of roughly 2GB sparse files. Each
file contains 8000 chunks stored as `<< Offset, ChunkData >>`.

- `Offset` is a 3-byte integer used to handle partial chunks (i.e. chunks less than 256KiB).
  This is only relevant for unpacked data as packed chunks are always 256 KiB.
- `ChunkData` is the 256 KiB (262,144 bytes) chunk data.

The data may be packed or unpacked (both formats take up the same mount of space). The
maximum file size is 2,097,176,000 bytes. Each file is named with the starting offset of
the chunks it contains (e.g. `chunk_storage` file `75702992896000` stores the 8000 chunks
starting at weave offset 75,702,992,896,000).

A full partition will contain 3.6 TB (3,600,000,000,000 bytes) of chunk data. Depending
on how you've configured your storage modules, your `chunk_storage` directory may only store
a subset of a partition.

For reasons [explained below](#partition-sizes) you will rarely be able to sync a full 3.6TB partition. However,
your node will continue to search the network for missing chunks so while unlikely it is
possible that a previously "dormant" `chunk_storage` directory to see some activity if
previously missing chunk data comes online. In general, though, once you have "fully" synced
a storage_module you would expect there to be no further writes to the `chunk_storage`
directory. [Below](#partition-sizes) we provide an estimate of each partition's "full synced" size.

### `rocksdb`

The `rocksdb` contains several [RocksDB](https://rocksdb.org/) databases used to store metadata
related to chunk data (e.g. record keeping, indexing, proofs, etc..).

The exact size of the `rocksdb` directory will vary over time - unlike `chunk_storage` you
should expect the `rocksdb` directory to continue to be written to as long as your node is
running. The current rough size of a `rocksdb` directory is ~100 GB (although it will vary
from partition to partition and node to node).

# Partition Sizes

You will find as you sync data that you're never able to download a full 3.6TB partition -
and in fact some partitions seem to stop syncing well short of the 3.6TB. There are 2 steps
when adding data to the Arweave network:

1. Submit a transaction with a hash of the data a transaction fee pay for and reserve space in
the weave.
2. Upload the data to the weave (aka seeding).

The data that is missing from a "full synced" partition is either data that has been
filtered out by your [content policies](https://arwiki.wiki/#/en/content-policies)
(or the content policies of your peers), or it is data that was never seeded by the uploader.

Typically ther 2 reasons why a user might not seed data after they've paid for it:
1. They're just testing / exploring the protocol
2. They're a miner experimenting with sacrifice mining

## Sacrifice Mining

Sacrifice Mining is a mining strategy where a miner will pay to upload data to the weave but
then never actually seed it. Instead they will keep and mine the data themselves, only sharing
the chunks required for any blocks they mine. The premise is that all of this "sacrificed data"
gives them a hashrate boost that other miners don't have (since the other miners are unable
to mine the unseeded data).

Sam has a good [thread](https://twitter.com/samecwilliams/status/1374062282817290247)
describing this.

The important bits:
1. Both in practice and based on economic models: sacrifice mining is not profitable. The cost
to reserve space on the weave exceeds the incremental revenue a miner can hope to get from the
additional hashrate. The payback period for the initial investment is long and gets longer
as the weave grows - in practice it is likely that a miner is never able to recoupe their
initial investment.
2. Putting aside the profitability of sacrifice mining, it is ultimately good for the network
as a whole. Sam breaks down why this is in his [thread](https://twitter.com/samecwilliams/status/1374062282817290247).

That said if you look through the partition size data below you'll notice 2 periods where
partition sizes are materially smaller than the expected 3.6TB (partitions 0-8, and 30-32). We
believe these correspond to periods when miners were experimenting with the strategy,
ultimately abandoning it as they realized it was unprofitable.

## Will Old Partitions Ever Be Written To?

When a transaction is posted to the Arweave block chain it contains a hash of the data
attached to it. However the data itself is typically seeded later and depending on the user
and the amount of data it can take some time to be seeded. The data itself has already been
"locked in" when the transaction is posted (i.e. the hash in the transaction guarantees that
only a specific set of data will be accepted for that transaction) - so you will never see
"new" data added to old partitions. However since there is no timeline on when the data can be
uploaded you may see "old" data finally get seeded/uploaded to the network some time after the
transaction is posted.

## Latest Estimated Partition Sizes

[See table here](partition-sizes.md)

**Note:** These numbers are *mostly* reliable, but there is always a chance that a previously
"fully synced" partition grows in size (though never greater than 3.6TB). This can happen
any time the original uploader decides to finally seed their previously unseeded data. In
practice this gets less and less likely the older a partition is.


# Performance Tips

There are 3 primary bottlenecks when syncing and packing:

1. Your network bandwidth *(used to download chunks from peers)*
2. Your CPU *(used to pack and unpack chunks)*
3. Your disk write speed *(used to write chunks to disk)*

And to a lesser degree:

4. RAM *(more heavily used in mining than in sync/packing, but can become a bottleneck under
  certain situations)*

If any of the 3 primary resources are maxed out: congratulations your configuration is syncing
and packing as fast as it can!

## Increasing Bandwidth

Not much to do here other than negotiate a faster internet connection, or find a second one.

## Increasing CPU

Packing and unpacking can be parallelized across chunks, so you can add more cores or increase
the clock speed to increase your packing speed. See the
[hardware guide](hardware.md#benchmarking-your-miner) for guidance on evaluating CPU pack
speed.

## Increasing Disk Write Speed

During the syncing and packing phase you will typically hit a network bandwidth or CPU
bottleneck before you hit a disk IO bottleneck (the reverse is true once you start mining).

If you believe you've hit a disk IO bottleneck you have a few options.

First, confirm that you're getting your expected disk write speed. You can use tools like `fio`
or `dd` to measure your disks write speed. If it is below your expected speed, you'll want to
check your system configuration (software and hardware) for issues.

Second, you can add more disks to your node. This is really only relevant if you have
partitions that you intend to sync and pack but which you haven't added to your node
configuration. As a general rule you should add all your storage modules to your node while
syncing and packing as this will increase your disk IO bandwidth as well as help fully
use your network and CPU bandwidth.

Third, you can buy faster disks. This is generally not recommended to unblock a syncing and
packing bottleneck as there's a good chance that extra IO speed will go unused once you
start mining. Including it here for completeness.

## Increasing RAM

The RAM guidelines mentioned in the [Mining Guide](mining-guide.md#preparation-ram) are
focused on mining. Often RAM is not a primary bottleneck during syncing and packing. If you
are maxing out your RAM: review the guidelines below. It's possible you can optimize your node
configuration.

## Increasing Utilization

Okay, so you've reviewed your bottlenecks and determined that **none** of them are at 
capacity. Here are some tips to increase syncing and packing speed.

### sync_jobs

The `sync_jobs` flag controls the number of concurrent requests your node will make to peers
to download data. The default is `100`. Increasing this number should increase your utilization
of all resources: the amount of data you pull from peers (network bandwidth), the number of
chunks you're packing (CPU), and the volume of data written to disk (disk IO).

However, it is possible to increase this number **too much**. This can:

1. Cause your node to be rate-limited / throttled by peers and ultimately decrease your
bandwidth utilization.
2. Increase your RAM utilization due to backed up sync jobs. This is particularly common if
your miner has a poor network connection (e.g. high latency or data loss). Increasing the
volume of slow/hanging requests can cause a backup which eventually leads to an out of memory
error.

Setting `sync_jobs` to `200` or even `400` is unlikely to cause any issues. But before you
increase it further our recommendation is to first confirm:
1. Your CPU and network bandwidths are below capacity
2. Your network connectivity is good (e.g. using tools like `mtr` to track packet loss)

### packing_rate

In general you shouldn't need to change `packing_rate` - the default value is usually more
than high enough. That said you can increase it with minimal risk of negative consequences.
Consult our [benchmarking guide](hardware.md#benchmarking-your-miner) to determine your CPU's
maximum packing rate and set `packing_rate` to something higher (e.g. 2x the maximum packing
rate)

### storage_module

As mentioned above under [Increasing Disk Write Speed](#increasing-disk-write-speed) syncing
to all your storage modules at once will maximize your available disk write bandwidth. The
same applied to network bandwidth. Adding more storage modules when syncing increases the
set of peers you can pull data from (as different peers share different parts of the weave
data). This will help your node maximize its network bandwidth by pulling from the "best" set
of peers available at a give time.

### Repacking

If you configure your node to repack from one local storage module to another the node
will prioritize that over pulling data from peers. This can cause you to max out your CPU
capacity while your network bandwidth stays low.

This is not a problem. It simply means your node will max out its CPU doing local repacks before
it begins searching for peers to download more data from. If you'd rather focus on syncing, 
just make sure to configure your node without any repacking. Two examples of configurations
that will cause local repacking:

1. `storage_module 9,unpacked storage_module 9,En2eqsVJARnTVOSh723PBXAKGmKgrGSjQ2YIGwE_ZRI`
2. `storage_module 16,Q5EfKawrRazp11HEDf_NJpxjYMV385j21nlQNjR8_pY storage_module 16,En2eqsVJARnTVOSh723PBXAKGmKgrGSjQ2YIGwE_ZRI`

**Note:** [as mentioned earlier](#packing), whenever you sync data - even if you are syncing to
`unpacked` - you will likely have to perform at least one packing operation. 

### Multiple Full Replicas

If you intend to build more than 1 packed full replica, the following approach should get
you there fastest:

1. Download all the data to `unpacked` storage modules
2. Build each packed replica by [locally repacking](examples.md#packing-unpacked-data) from
  your `unpacked` storage modules
3. You can either keep the `unpacked` data around for later, or, you can do a
  [repack_in_place](examples.md#repacking-packed-data-in-place) when building your final packed replica.

This approach will reduce your download time (since you only have to download the data once)
and reduce the number of packing operations (since you only have to unpack peer data once).

**Note:** This approach is not recommended if your goal is to have 1 or fewer packed replicas.
It will work, but won't be any faster than just syncing straight to packed storage modules.

# FAQs

Faq about how to know when done
Faq about unpacked still requires packing
or once the weave moves on to the next partition, if the previous isnt full, it never will be backfilled ?
How to estimate performance when packing
How often are writez made to older partitions
polling mode FAQ
syncing/packing grafana sample dashboard and metrics
what happens if I delete a chunk_storage file by mistake?
what is the formate of chunk_storage files