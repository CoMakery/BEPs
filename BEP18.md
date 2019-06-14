# BEP-18: State sync enhancement

- [BEP-18: State sync enhancement](#bep-18-state-sync-enhancement)
  - [1.  Summary](#1--summary)
  - [2.  Abstract](#2--abstract)
  - [3.  Status](#3--status)
  - [4.  Motivation](#4--motivation)
  - [5.  Specification](#5--specification)
    - [5.1 Take snapshot](#51-take-snapshot)
    - [5.2 Sync snapshot](#52-sync-snapshot)
    - [5.3 Manifest format](#53-manifest-format)
    - [5.4 Snapshot chunk format](#54-snapshot-chunk-format)
      - [5.4.1 App state chunk](#541-app-state-chunk)
      - [5.4.2 Tendermint state chunk](#542-tendermint-state-chunk)
      - [5.4.3 Block chunk](#543--block-chunk)
    - [5.5 Operation suggestion](#555-operation-suggestion)
  - [6. License](#6-license)

## 1.  Summary

This BEP describes [state sync](https://docs.binance.org/fullnode.html#state-sync) enhancement on the Binance Chain.

## 2.  Abstract

[State sync](https://docs.binance.org/fullnode.html#state-sync) is a way to help newly-joined user sync latest status of binance chain. It sync latest syncable peer's status so that fullnode user (who want catch up with chain as soon as possible) doesn't need sync from block height 0 (with cost that discard all historical blocks locally).

BEP-18 Proposal describes an enhancement of existing state sync implementation to improve user experience. It introduces the following details:

- What's the procedure to take a snapshot
- What's the procedure to sync snapshot from other peers
- Snapshot (manifest, snapshot chunks) format

## 3.  Status

This BEP is still working in progress.

## 4.  Motivation

We propose this BEP to enhance full node user experience (and ease their paining) on using state sync because of following implementation limitation.

1. Users complain most about state syncing testnet is very slow and usually stucking on some request. <br> <br> In this enhancement, we want data responsed more evenly across peers so that syncing can continously making progress and the overall syncing time can reduce from 30 - 45 min to around 5 min.

2. Interruption during state sync (node process get killed because of reboot computer or user inpatient) would make already synced state in vain (because current full node doesn't persist synced part on dist) more worse it mistakenly write a lockfile prevent user state sync again. <br> <br> In this enhancement, we want support break-resume downloading and keep consistent status for arbitrarily restart.

3. User cannot control which height they can sync from. Current implementation only allows user sync latest syncable height. <br> <br> In this enhancement, we want make syncing from historical syncable height possible. User either specify a height to sync in config file or follow latest syncable height. 

## 5.  Specification

State sync will download manifest and snapshot chunks from other peers.

### 5.1 Take snapshot

There are two ways to taking snapshot from a fullnode: automatically or manually. Snapshots will be put under `$HOME/data/snapshot/<height>`. All types involved in snapshot are encoded by go-amino and wrapped by state sync reactor transimission message after snappy compression. More detail will be explained later.

1. To make fullnode automatically take snapshot, just make sure `state_sync_reactor` in `$HOME/config.toml` is set to true.
2. To manually take a snapshot, stop the node if it is running then run `./bnbchaind snapshot --home <home> --height <syncable_height>`, the `<syncable_height>` can be found from standard output (instead of default bnc.log) of bnbchaind process.

If the snapshot taking procedure is interrupted, node will be still in good status, but it cannot provide the interrupted height for other peers.

Note: automatically take snapshot would keep occupy disk space. Fullnode would not delete them automatically, so user should periodically delete un-needed snapshot manually if they want save disk space.

### 5.2 Sync snapshot

Syncing snapshot is designed to be only can happen once during full node first start. To enable state sync from others, `state_sync_reactor` should be true and `state_sync_height` should be set to non-negative (default `-1`).

If user want sync from (majority) peers' latest syncable height, they should set  `state_sync_height` to 0

If user want sync from a specific height, they should set `state_sync_height` to a syncable height.

Stop and restart fullnode during state sync is allowed. The next time full node is started it will resume by loading Manifest and downloaded chunks then download missing chunks.

Once state sync is succeed, a `STATESYNC.LOCK` file will be created uhnder `$HOME/data` to prevent state sync next time.

### 5.3 Manifest format

Manifest serves a summary of snapshot chunks to be synced. It also maintain the order and different types of snapshot chunks. Fullnode firstly ask peer's manifest file at beginning of state sync and will trust majority peers with same manifest.

SHA256 hash sum of each chunk synced will be checked against the hash decleared within minifest file.

| Field          | Type        | Description                                                                                                                                                                                                                                                                                                                                                      |
|----------------|-------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Version        | int32       | snapshot version                                                                                                                                                                                                                                                                                                                                                 |
| Height         | int64       | height of this snapshot                                                                                                                                                                                                                                                                                                                                          |
| StateHashes    | []SHA256Sum | hashes of tendermint state chunks                                                                                                                                                                                                                                                                                                                                |
| AppStateHashes | []SHA256Sum | hashes of app state chunks                                                                                                                                                                                                                                                                                                                                       |
| BlockHashes    | []SHA256Sum | hashes of block in this snapshot, currently only one block is synced                                                                                                                                                                                                                                                                                             |
| NumKeys        | []int64     | num of keys for each substore, this sacrifices clear boundary between cosmos and tendermint, saying tendermint knows applicaction db might organized by substores.<br><br>But reduce network/disk pressure that each chunk must has a field indicates what's his storeBut reduce network/disk pressure that each chunk must has a field indicates what's his store |

### 5.4 Snapshot chunk format

#### 5.4.1 App state chunk

 App state chunk includes iavl tree nodes. Usually each app state chunk takes up to 4MB serialized iavl tree nodes (before snappy compression). 
 
 Iavl tree node bigger than 4MB is splitted into different incomplete chunks, that's where `Completenss` field effect.

| Field        | Type     | Description                                                                                                                                                                                  |
|--------------|----------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| StartIdx     | int64    | compare (startIdx and number of complete nodes) against (Manifest.NumKeys) we know each node's substore                                                                                      |
| Completeness | uint8    | flag of completeness of this chunk, not enum because of go-amino doesn't support enum encoding                                                                                               |
| Nodes        | [][]byte | iavl tree serialized node, one big node (i.e. active orders and orderbook) might be split into different chunks (complete is flag to indicate that), ordering is ensured by list on manifest |

#### 5.4.2 Tendermint state chunk

| Field     | Type   | Description |
|-----------|--------|-------------|
| Statepart | []byte | current tendermint state            |

#### 5.4.3 Block chunk

| Field      | Type   | Description                                                                                                                                    |
|------------|--------|------------------------------------------------------------------------------------------------------------------------------------------------|
| Block      | []byte | amino encoded block                                                                                                                            |
| SeenCommit | []byte | amino encoded Commit<br>we need this because Block only keep LastSeenCommit, for commit of this block, we need load it in same way it was saved |

### 5.5 Operation suggestion
1. As mentioned in section [5.1 Take snapshot](#51-take-snapshot), fullnode cannot delete snapshot directories (`$HOME/data/snapshot/<height>`) automatically. This needs noticed by full node users who enabled `state_sync_reactor` in `$HOME/config.toml`. Either run a script periodically delete the snapshots or turn off `state_sync_reactor` (if they want be selfish!) should be considered.

2. Once state sync succeeded, later full node restart would not state sync anymore (in case the local blocks are not continuous).

But if user do want state sync again (don't care that there are missing blocks between last stop and latest state sync snapshot) and he want to keep already synced blocks, he should delete $BNCHOME/data/STATESYNC.LOCK.

## 6. License

The content is licensed under [CC0](https://creativecommons.org/publicdomain/zero/1.0/).
