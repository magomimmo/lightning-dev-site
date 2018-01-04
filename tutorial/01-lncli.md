---
layout: page
title: Stage 1 - Setting up a local cluster
---

## Prerequisites

This tutorial assumes you have completed installation of Go, `btcd`, and `lnd`. If not, please refer to the [installation
instructions](/guides/installation/).

## Introduction

In this stage of the tutorial, we will learn how to set up a local cluster of `lnd` nodes for `Alice`, `Bob`, and `Charlie`, have them talk to each other, set up channels, and route payments between one another. We will also establish a baseline understanding of the different components that must work together as part of developing on `lnd`.

The schema will be the following. Keep in mind that you can easily extend this network to include additional nodes `David`, `Eve`, etc. by simply running more local `lnd` instances.

[//]: # (TODO Max: Replace this with an actual image)
```
   (1)                        (1)                         (1)
+ ----- +                   + --- +                   + ------- +
| Alice | <--- channel ---> | Bob | <--- channel ---> | Charlie |    
+ ----- +                   + --- +                   + ------- +        
    |                          |                           |           
    |                          |                           |
    + - - - -  - - - - - - - - + - - - - - - - - - - - - - +            
                               |
                      + --------------- +
                      | BTC/LTC network | <--- (2)
                      + --------------- +        
```

## Understanding the components

### LND nodes

`lnd` is the main component that we will interact with. `lnd` stands for Lightning Network Daemon, and handles channel opening/closing, routing and sending payments, and managing all the Lightning Network state that is separate from the underlying Bitcoin network itself.

Running an `lnd` node means that it is listening for payments, watching the blockchain, etc. By default it is awaiting user input.

As said, we're going to setup three `lnd` nodes, one for each peer. Typically, to keep them separated from each other, each `lnd` node will be running in its own terminal window from where you can see its log outputs.

We'll also run an `lnd` node, named miner, for collecting BTC while the underline `btcd` daemon generates blocks.    

### LNCLI Command Line Client

`lncli` is the command line client used to interact with your `lnd` nodes. We'll create a forth terminal window to use it.

### BTCD Daemon

`btcd` represents the gateway that `lnd` nodes will use to interact with the Bitcoin / Litecoin network. `lnd` needs `btcd` for creating on-chain addresses or transactions, watching the blockchain for updates, and opening/closing channels. In our current schema, all of the nodes are connected to the same `btcd` instance. In a more realistic scenario, each of the `lnd` nodes will be connected to their own instances of `btcd` or equivalent.

We'll need a fifth terminal window to launch the `btcd` daemon.

We will also be using `simnet` instead of `testnet`. Simnet is a development/test network run locally that allows us to generate blocks at will, so we can avoid the time-consuming process of waiting for blocks to arrive for any on-chain functionality.

### BTCCTL Command Line Client

`btcctl` is the command line client used to interact with your `btcd` node. We'll use the same terminal window used for `lncli` command line client.  

## Setting up our environment

Developing on `lnd` can be quite complex since there are many more moving pieces, so to simplify that process, we will walk through a recommended workflow.

### Configuring btcd daemon

Even if you could run `btcd` by passing to it all the needed options at the terminal, we prefer to edit those options in the `btcd.conf` file which, by default, is located in the `~/.btcd` directory on POSIX OSes (e.g. GNU/Linux) and under the `~/Library/Application Support/Btcd` directory on MacOS.

```bash
mkdir ~/.btcd
touch ~/.btcd/btcd.conf
```

Edit `btcd.conf` file as follows:

```
[Application Options]
simnet=1
rpcuser=kek
rpcpass=kek
txindex=1
```

Breaking down the components:

* `simnet=1` specifies that we are using the `simnet` network. This can be
changed to `testnet=1`, or omitted entirely to connect to the actual Bitcoin
/ Litecoin network;
* `rpcuser=kek` and `rpcpass=kek` sets default credentials for authenticating clients to the `btcd` instance;
* `txindex=1` is required so that the `lnd` client is able to query historical transactions from `btcd`.

> NOTE: For a full list of all the `btcd.conf` options see [sample-btcd.com](https://github.com/Roasbeef/btcd/blob/master/sample-btcd.conf)

### Configuring btcctl command line client

Considering that during the tutorial we're going to use more times the `btcctl` command line client to interact with `btcd`, we're going to configure its `btcctl.conf` counterpart as well.

`btcct.conf` is located under `~/.btcctl` directory on POSIX OSes and under `~/Library/Application Support/Btcctl` directory on MacOS.

```
mkdir ~/.btcctl
touch ~/.btcctl/btcctl.conf
```

Edit `btcctl.conf` file as follows:

```
[Application Options]
simnet=1
rpcuser=kek
rpcpass=kek
```

Breaking down the components: same as above.

### Running btcd

Let's start by running `btcd` from the first terminal window. If you don't have it up already, ensure you have your `$GOPATH` set, and run

```bash
# from a new terminal window
btcd
```

## Configuring lnd nodes

Let's now set up our `lnd` nodes. To keep things as clean and separated as possible, open up a second terminal window, ensure you have `$GOPATH` set and `$GOPATH/bin` in your `PATH`, and create a new directory under `$GOPATH` called `dev` that will represent our development space. We will create separate folders to store the state for alice, bob, charlie, and miner and run all of our `lnd` nodes on different `localhost` ports instead of using [Docker](/guides/docker/) to make our networking a bit easier.

```bash
# Create our development space
mkdir -p $GOPATH/dev/{alice,bob,charlie,miner}
```

The directory structure should now look like this:

```bash
tree $GOPATH/dev

├── alice
├── bob
├── charlie
└── miner
```

### Configuring lnd.conf

To skip having to type out a bunch of flags on the command line every time, we can instead create a `lnd.conf` file, and the arguments specified therein will be loaded into `lnd` automatically. Any additional configuration added as a command line argument will be applied *after* reading from `lnd.conf`, and will overwrite the `lnd.conf` option if applicable.

* On MacOS, `lnd.conf` is located at: `~/Library/Application\ Support/Lnd/lnd.conf`
* On POSIX OSes: `~/.lnd/lnd.conf`

Here is an example `lnd.conf` that can save us from re-specifying a bunch of command line options:

```bash
[Application Options]
datadir=test_data
logdir=test_log
debuglevel=info
debughtlc=true
no-macaroons=true
noencryptwallet=true

[Bitcoin]
bitcoin.simnet=1
bitcoin.active=1
bitcoin.rpcuser=kek
bitcoin.rpcpass=kek
```

Breaking down the components:

* `lnd` options:
  * `datadir`: The directory that `lnd`'s data will be stored inside
  * `logdir`: The directory to log output;
  * `debuglevel`: The logging level for all subsystems. Can be set to `trace`, `debug`, `info`, `warn`, `error`, `critical`;
  * `debughtlc`: To automatically settle a special type of HTLC sent to it. This means that you won't need to manually insert invoices in order to test payment connectivity;
  * `no-macaroons`: Disable macaroon authentication for tutorial purposes;
  * `noencryptwallet`: Disable encryption for tutorial purpose only;
* `Bitcoin` options:
  * `bitcoin.simnet`: Specifies to use `simnet`;
  * `bitcoin.active`: Specifies that bitcoin is active;
  * `bitcoin.rpcuser` and `bitcoin.rpcpass`: The username and password for authenticating with the running `btcd` daemon.

> NOTE: For a full list of all the `lnd.conf` options see [sample-lnd.com](https://github.com/lightningnetwork/lnd/blob/master/sample-lnd.conf)

### Running lnd nodes

We can now run an `lnd` for each peer. Open 4 new terminal windows, one for each peer and `cd` in the corresponding directory we previously prepared:

```bash
# from alice's terminal window
cd $GOPATH/dev/alice
lnd --rpcport=10001 --peerport=10011 --restport=8001

# from bob's terminal window
cd $GOPATH/dev/bob
lnd --rpcport=10002 --peerport=10012 --restport=8002

# from charlie's terminal window
cd $GOPATH/dev/charlie
lnd --rpcport=10003 --peerport=10013 --restport=8003

# from miner's terminal window
cd $GOPATH/dev/miner
lnd --rpcport=10004 --peerport=10014 --restport=8004
```

Each `lnd` node should now be running and displaying output.

Breaking down the components:

* `rpcport`: The port to listen for the RPC server. This is the primary way an application will communicate with `lnd`;
* `peerport`: The port to listen on for incoming P2P connections. This is at the networking level, and is distinct from the Lightning channel networks and Bitcoin/Litcoin network itself;
* `restport`: The port exposing a REST api for interacting with `lnd` over HTTP. For example, you can get Alice's channel balance by making a GET request to `localhost:8001/v1/channels`. This is not needed for this tutorial, but you can see some examples [here](https://gist.github.com/Roasbeef/624c02cd5a90a44ab06ea90e30a6f5f0).

### Aliasing lncli calls

During the tutorial we're going to interact a lot via `lncli` with the `lnd` nodes we previously launched from dedicated terminal windows.

To see all the commands and options available for `lncli`, simply type `lncli --help` or `lncli -h`, or `lncli help` or `lncli h`:

```
lncli -h
NAME:
   lncli - control plane for your Lightning Network Daemon (lnd)

USAGE:
   lncli [global options] command [command options] [arguments...]

VERSION:
   0.3

COMMANDS:
     ...
     newaddress       generates a new address.
     ...
     sendcoins        send bitcoin on-chain to an address
     connect          connect to a remote lnd peer
     ...
     openchannel      Open a channel to an existing peer.
     closechannel     Close an existing channel.
     listpeers        List all active, currently connected peers.
     walletbalance    compute and display the wallet's current balance
     channelbalance   returns the sum of the total available channel balance across all open channels
     getinfo          returns basic information related to the active daemon
     ...     
     sendpayment      send a payment over lightning
     ...
     addinvoice       add a new invoice.
     ...
     listchannels     list all open channels
     ...
     stop             Stop and shutdown the daemon.
     ...
     help, h          Shows a list of commands or help for one command

GLOBAL OPTIONS:
   --rpcserver value        host:port of ln daemon (default: "localhost:10009")
   ...
   --no-macaroons           disable macaroon authentication
   ...
   --help, -h               show help
   --version, -v            print the version
```

> NOTE: above we only reported the `lncli` commands and global options we're going to use in this tutorial.

`lncli` submits RPC calls to the peers' `lnd` nodes and for each call we have to specify the `--no-macarrons` flag and the `--rpcserver` address which corresponds to `--rpcport=1000X` we set when starting the corresponding `lnd` node. To avoid to much typing for every interaction, we can set few aliases. Add the following to your `.bashrc` (or `.bash_profile` on MacOS):

```bash
alias lncli-alice="lncli --rpcserver=localhost:10001 --no-macaroons"
alias lncli-bob="lncli --rpcserver=localhost:10002 --no-macaroons"
alias lncli-charlie="lncli --rpcserver=localhost:10003 --no-macaroons"
alias lncli-miner="lncli --rpcserver=localhost:10004 --no-macaroons"
```

To make sure this was applied, rerun your `.bashrc` file:

```bash
# in the same terminal window you configured lnd nodes
source ~/.bashrc # or ~/.bash_profile on MacOS
```

### Working with lncli

Now that we have our `lnd` nodes up and running, let's interact with them via the `lncli` aliases we just prepared.

Let's test that we can connect to peers by requesting basic information:

```bash
# alice's info
lncli-alice getinfo
{
    "identity_pubkey": <ALICE_PUBKEY>,
    "alias": "",
    "num_pending_channels": 0,
    "num_active_channels": 0,
    "num_peers": 0,
    "block_height": 0,
    "block_hash": "683e86bd5c6d110d91b94b97137ba6bfe02dbbdb8e3dff722a669b5d69d77af6",
    "synced_to_chain": false,
    "testnet": false,
    "chains": [
        "bitcoin"
    ]
}

# bob's info
lncli-bob getinfo
{
    "identity_pubkey": <BOB_PUBKEY>,
    "alias": "",
    "num_pending_channels": 0,
    "num_active_channels": 0,
    "num_peers": 0,
    "block_height": 0,
    "block_hash": "683e86bd5c6d110d91b94b97137ba6bfe02dbbdb8e3dff722a669b5d69d77af6",
    "synced_to_chain": false,
    "testnet": false,
    "chains": [
        "bitcoin"
    ]
}

# charlie's info
lncli-charlie getinfo
{
    "identity_pubkey": <CHARLIE_PUBKEY>,
    "alias": "",
    "num_pending_channels": 0,
    "num_active_channels": 0,
    "num_peers": 0,
    "block_height": 0,
    "block_hash": "683e86bd5c6d110d91b94b97137ba6bfe02dbbdb8e3dff722a669b5d69d77af6",
    "synced_to_chain": false,
    "testnet": false,
    "chains": [
        "bitcoin"
    ]
}
```

### Setting up Bitcoin addresses

Let's now create a new Bitcoin address for each peer. These will be the addresses that store peers' on-chain balances. Copy them down, because later we'll need them to fund Alice and Charlie and to set the miner as the recipient of the blocks generation.

```bash
# alice's address
lncli-alice newaddress np2wkh
{
    "address": <ALICE_ADDRESS>
}

# bob's address
lncli-bob newaddress np2wkh
{
    "address": <BOB_ADDRESS>
}

# charlie's address
lncli-charlie newaddress np2wkh
{
    "address": <CHARLIE_ADDRESS>
}

# moiner's address
lncli-miner newaddress np2wkh
{
    "address": <MINER_ADDRESS>
}
```

### Funding Peers

That's a lot of configuration! Recall that at this point, we've generated on-chain addresses for Alice, Bob, Charlie and the `miner` as well. Now, we will get some practice working with `btcd` and fund these addresses with some `simnet` Bitcoin.

Quit all the `lnd` running nodes and `btcd` as well:

```bash
lncli-alice stop
lncli-bob stop
lncli-charlie stop
lncli-miner stop
btcctl stop
```

#### Funding miner and activating segwit

We'll use the miner to fund peers, but first we need to fund the miner itself by setting his address as the recipient of the blocks rewards.

First, run `btcd` again by specifying the `--miningaddr` option:

```bash
# from the same terminal window previously used to launch btcd
btcd --miningaddr=<MINER_ADDRESS>
```

Then we need to generate about 300 blocks to activate `segwit`. This step funds the miner as well.

```bash
# from the terminal windows previously used to stop the nodes
btcctl generate 300
```
Check that segwit is active:

```
btcctl getblockchaininfo | grep -A 1 segwit
    "segwit": {
      "status": "active",
```

#### Fund Alice and Charlie

To fund Alice and Charlie we now only need to relaunch the miner's `lnd` node and send some coins to their addresses from the miner's node:

```bash
# from the terminal window previously used to launch the miner's lnd node
lnd --rpcport=10004 --peerport=10014 --restport=8004

# form the terminal window previously used to stop nodes

# fund Alice
lncli-miner sendcoins --addr=<ALICE_ADDRESS> --amt=1000000

# fund CHARLIE_PUBKEY
lncli-miner sendcoins --addr=<CHARLIE_ADDRESS> --amt=1000000

# generate six block confirmations
btcctl generate 6
```

#### What about Bob?

We're not going to fund Bob, just to demonstrate that a peer does not need to be funded on-chain for participating in off-chain payment channels. He only need to have an on-chain address and be connected with other peers, as we'll see in a moment.

#### re-run peers' nodes

After having activated segwit and funded Alice and Charlie we can re-run all the `lnd` nodes:

```bash
# from alice's terminal window
lnd --rpcport=10001 --peerport=10011 --restport=8001

# from bob's terminal window
lnd --rpcport=10002 --peerport=10012 --restport=8002

# from charlie's terminal window
lnd --rpcport=10003 --peerport=10013 --restport=8003
```

### Check peers' balances

Let's now check the on-chain balance of each peer:

```bash
# in the same terminal windows already used for lncli and btcctl
# alice's balance
lncli-alice walletbalance
{
    "total_balance": "1000000",
    "confirmed_balance": "1000000", <--- 0.01 BTC
    "unconfirmed_balance": "0"
}
# bob's balance
lncli-bob walletbalance
{
    "total_balance": "0",
    "confirmed_balance": "0",
    "unconfirmed_balance": "0"
}

# charlie's balance
lncli-charlie walletbalance
{
    "total_balance": "1000000",
    "confirmed_balance": "1000000", <--- 0.01 BTC
    "unconfirmed_balance": "0"
}
```

### Creating the P2P Network

Now the things are going to be more interesting. Let's start creating the P2P network connecting Alice to Bob and Bob to Charlie.

Connect Alice to Bob:

```bash
# Get Bob's pubkey:
lncli-bob getinfo
{
    "identity_pubkey": <BOB_PUBKEY>,
    "alias": "",
    "num_pending_channels": 0,
    "num_active_channels": 0,
    "num_peers": 0,
    "block_height": 306,
    "block_hash": "0ad0407db0f94765c35448e85f1bf258aeba05405d44bead9618f6db5417077f",
    "synced_to_chain": true,
    "testnet": false,
    "chains": [
        "bitcoin"
    ]
}

# Connect Alice and Bob together
lncli-alice connect <BOB_PUBKEY>@localhost:10012
{
    "peer_id": 0
}
```

Notice that `localhost:10012` corresponds to the `--peerport=10012` flag we set when starting the Bob `lnd` node.

Connect Charlie to Bob:

```bash
lncli-charlie connect <BOB_PUBKEY>@localhost:10012
{
    "peer_id": 0
}
```

Let's check that the peers are now connected as expected:

```bash
# Check that Alice has added Bob as a peer:
lncli-alice listpeers
{
    "peers": [
        {
            "pub_key": <BOB_PUBKEY>,
            "peer_id": 1,
            "address": "127.0.0.1:10012",
            "bytes_sent": "149",
            "bytes_recv": "149",
            "sat_sent": "0",
            "sat_recv": "0",
            "inbound": true,
            "ping_time": "0"
        }
    ]
}

# Check that Bob has added Alice and Charlie as peers:
lncli-bob listpeers
{
    "peers": [
        {
            "pub_key": <ALICE_PUBKEY>,
            "peer_id": 1,
            "address": "127.0.0.1:50757",
            "bytes_sent": "175",
            "bytes_recv": "175",
            "sat_sent": "0",
            "sat_recv": "0",
            "inbound": false,
            "ping_time": "233"
        },
        {
            "pub_key": <CHARLIE_PUBKEY>,
            "peer_id": 2,
            "address": "127.0.0.1:50763",
            "bytes_sent": "175",
            "bytes_recv": "175",
            "sat_sent": "0",
            "sat_recv": "0",
            "inbound": false,
            "ping_time": "278"
        }
    ]
}

# Check that Charlie has added Bob as a peer:
lncli-charlie listpeers
{
    "peers": [
        {
            "pub_key": <BOB_PUBKEY>,
            "peer_id": 1,
            "address": "127.0.0.1:10012",
            "bytes_sent": "201",
            "bytes_recv": "201",
            "sat_sent": "0",
            "sat_recv": "0",
            "inbound": true,
            "ping_time": "311"
        }
    ]
}
```

> NOTE: Bob has 2 connections: one connection with Alice and a second connection with Charlie.

### Setting up Lightning Network

Before we can send payments, we will need to set up payment channels from Alice
to Bob, and from Charlie to Bob.

> NOTE: In this tutorial we're going to use the so called single-funded channels (i.e channel opened by a funding transaction containing only inputs from the funder). At the moment dual-funded channels are not supported.

> NOTE: Currently there is a limit of 16,777,216 (i.e. 0.16777216 BTC) Satoshi for the capacity of a single payment channel.

First, let's open the Alice<-->Bob channel:

```bash
lncli-alice openchannel --node_key=<BOB_PUBKEY> --local_amt=100000
{
	"funding_txid": <ALICE_FUNDING_TXID>
}
```

> NOTE: `--local_amt` specifies the amount of Satoshi (e.g. `1,000,000` Satoshi) that Alice will commit to the channel. To see the full list of options, you can try `lncli openchannel -h`.

Now let's open the Charlie<-->Bob channel:

```bash
lncli-charlie openchannel --node_key=<BOB_PUBKEY> --local_amt=80000 --push_amt=20000
{
	"funding_txid": <CHARLIE_FUNDING_TXID>
}
```
> NOTE: This time we supplied the `--push_amt` argument, which specifies the amount of coins we want to other party to have at the first channel state. `--push_amt` is the number of Satoshi to push to the remote side as part of the initial commitment state (default: 0). In other words, it is an amount of initial funds that the sender (i.e. Charlie) is unconditionally giving to the receiver (i.e. Bob).

We now need to mine 6 blocks so that payment channels are considered fully valid (i.e. 3 blocks for the funding transaction and other 3 blocks for the confirmation of the subscription):

```bash
btcctl generate 6
```

Check that Alice<-->Bob and Charlie<-->Bob channels have been created:

```bash
# Alice's channels
lncli-alice listchannels
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": <BOB_PUBKEY>,
            "channel_point": <ALICE_FUNDING_TXID>:<OUTPUT_NDX>,
            "chan_id": "333152023347200",
            "capacity": "100000",
            "local_balance": "91312",  <--- Alice
            "remote_balance": "0",      <--- Bob
            "commit_fee": "8688",       <--- on chain tx fee
            "commit_weight": "600",
            "fee_per_kw": "12000",
            "unsettled_balance": "0",
            "total_satoshis_sent": "0",
            "total_satoshis_received": "0",
            "num_updates": "0",         <--- number of balance updates
            "pending_htlcs": [
            ],
            "csv_delay": 4
        }
    ]
}

# Bob's channels
lncli-bob listchannels
{
    "channels": [
        {
            "active": true,
            "remote_pubkey": <ALICE_PUBKEY>,
            "channel_point": <ALICE_FUNDING_TXID>:<OUTPUT_NDX>,
            "chan_id": "333152023347200",
            "capacity": "100000",
            "local_balance": "0",         <--- Bob
            "remote_balance": "91312",   <--- Alice  
            "commit_fee": "8688",         <--- on chain tx fee
            "commit_weight": "552",
            "fee_per_kw": "12000",
            "unsettled_balance": "0",
            "total_satoshis_sent": "0",
            "total_satoshis_received": "0",
            "num_updates": "0",           <--- number of balance updates
            "pending_htlcs": [
            ],
            "csv_delay": 4
        },
        {
            "active": true,
            "remote_pubkey": <CHARLIE_PUBKEY>,
            "channel_point": <CHARLIE_FUNDING_TXID>:<OUTPUT_NDX>,
            "chan_id": "333152023281664",
            "capacity": "800000",
            "local_balance": "20000",      <--- Bob
            "remote_balance": "51312",     <--- Charlie
            "commit_fee": "8688",           <--- on chain tx fee
            "commit_weight": "724",
            "fee_per_kw": "12000",
            "unsettled_balance": "0",
            "total_satoshis_sent": "0",
            "total_satoshis_received": "0",
            "num_updates": "0",             <--- number of balance updates
            "pending_htlcs": [
            ],
            "csv_delay": 4
        }
    ]
}
```

Note as the commission fee (8,688 Satoshi) for the on-chain funding transactions to open the channel have been payed by the channel funders (i.e. Alice and Charlie).

### Sending single hop payments

Finally, the exciting part: sending payments! Let's send a payment from Alice to Bob.

> NOTE: Currently there is limit of 4,294,967 Shatoshi (i.e. 0.04294967 BTC) for a single off-chain payment.

First, let's verify the peer's channel balances:

```bash
# Alice
lncli-alice channelbalance
{
    "balance": "91312"
}

# Bob
lncli-bob channelbalance
{
    "balance": "20000" <-- 0 Alice<->Bob + 20,000 in the Charlie<-->Bob
}
# Charlie
```

Bob needs to generate an invoice:

```bash
lncli-bob addinvoice --value=10000
{
        "r_hash": "<a_random_rhash_value>",
        "pay_req": "<encoded_invoice>",
}
```

And Alice needs to send the corresponding payment to Bob:

```bash
lncli-alice sendpayment --pay_req=<encoded_invoice>
{
	"payment_error": "",
	"payment_preimage": "65906f1cef5510c55637ecfa2aba6c86bd2d3383eb98aff1336b148cdd6cff75",
	"payment_route": {
		"total_time_lock": 456,
		"total_amt": 10000,
		"hops": [
			{
				"chan_id": 337550069792768,
				"chan_capacity": 100000,
				"amt_to_forward": 10000,
				"expiry": 456
			}
		]
	}
}

# Check that Alice's channel balance was decremented accordingly:
lncli-alice channelbalance
{
    "balance": "81312"  <-- -10,000
}

# Check that Bob's channel was credited with the payment amount:
lncli-bob channelbalance
{
    "balance": "30000"  <-- +10,000
}
```

### Multi-hop payments

Now we know how to send single-hop payments. By considering that we have already opened the channel between Charlie and Bob, to send multi hop payments from Alice to Charlie is not that much more difficult.

#### Alice pays Charlie

Let's make a payment from Alice to Charlie by routing through Bob. First Charlie, as did Bob before, has to create an invoice:

```bash
lncli-charlie addinvoice --value=10000
{
        "r_hash": <a_random_rhash_value>,
        "pay_req": <encoded_invoice>,
}
```

Then Alice has to pay it:

```bash
lncli-alice sendpayment --pay_req=<encoded_invoice>
{
	"payment_error": "",
	"payment_preimage": "d3d9a5d1f03302592a6beda00e1906711d693bbeb911e31a7811f59ba5204efb",
	"payment_route": {
		"total_time_lock": 600,
		"total_fees": 1,        <-- very small fee to Bob
		"total_amt": 10001,
		"hops": [
			{
				"chan_id": 337550069792768,
				"chan_capacity": 100000,
				"amt_to_forward": 10000,
				"fee": 1,
				"expiry": 456
			},
			{
				"chan_id": 337550069858304,
				"chan_capacity": 80000,
				"amt_to_forward": 10000,
				"expiry": 456
			}
		]
	}
}
```

Now check the peers' channels have been updated as expected

```bash
# alice
lncli-alice channelbalance
{
    "balance": "71310"
}
# bob
lncli-bob channelbalance
{
    "balance": "30001"
}
charlie
lncli-charlie channelbalance
{
    "balance": "61312"
}
```

### Closing channels

For practice, let's try closing the channels by passing the channel point and the output index to the `closechannel` command.

The channel point consists of two numbers separated by a colon, which uniquely identifies the channel. The first number is `funding_txid` and the second number is `output_index`. You get the channel point from the `lisrchannels` command:


```
lncli-bob listchannels
{
    "channels": [
        {   ...
            "channel_point": <ALICE_FUNDING_TXID>:<OUTPUT_NDX>,
            ...
        },
        {   ...
            "channel_point": <CHARLIE_FUNDING_TXID:<OUTPUT_NDX>,
            ...
    ]
}

# Close the Alice<-->Bob channel from Alice's side.
lncli-alice closechannel --funding_txid=<ALICE_FUNDING_TXID> --output_index=0

# close the Charlie channel with Bob
lncli-charlie closechannel --funding_txid=<CHARLIE_FUNDING_TXID> --output_index=0
```

Finally, as usual, we need to generate at least a block to close channels:

```bash
btcctl generate 1
```

And check the peers' on-chain balances:

```bash
# check alice's on-chain balance
lncli-alice walletbalance
{
    "total_balance": "962874",
    "confirmed_balance": "962874",
    "unconfirmed_balance": "0"
}

# Check that Bob's on-chain balance was credited by his settled amount in the
# channel. Recall that Bob previously had no on-chain Bitcoin:
{
    "total_balance": "30001",
    "confirmed_balance": "30001",
    "unconfirmed_balance": "0"
}

# check charlie's on-chain balance
lncli-charlie walletbalance
{
    "total_balance": "972876",
    "confirmed_balance": "972876",
    "unconfirmed_balance": "0"
}
```

### Stop lnd and btcd nodes

Finally stop all running noded

```bash
lncli-alice stop
lncli-bob stop
lncli-charlie stop
lncli-miner stop
btcctl stop
```

At this point, you've learned how to work with `btcd`, `btcctl`, `lnd`, and
`lncli`. In [Stage 2](/tutorial/02-web-client), we will learn how to set up and
interact with `lnd` using a web GUI client.

_In the future, you can try running through this workflow on `testnet` instead
of `simnet`, where you can interact with and send payments through the testnet
Lightning Faucet node. For more information, see the "Connect to faucet
lightning node" section in the [Docker guide](/guides/docker/) or check out the
[Lightning Network faucet
repository](https://github.com/lightninglabs/lightning-faucet)._

[//]: # (TODO Max: Replace the link to Github LN Faucet to an internal guide for interacting with the Lightning Faucet)

#### Navigation
- [Proceed to Stage 2 - Web Client](/tutorial/02-web-client)
- [Return to main tutorial page](/tutorial/)

### Questions
- Join the #dev-help channel on our [Community
  Slack](https://join.slack.com/t/lightningcommunity/shared_invite/MjI4OTg3MzQ4MjI2LTE1MDMxNzM1NTMtNjlmOGYzOTI1Ng)
- Join IRC:
  [![Irc](https://img.shields.io/badge/chat-on%20freenode-brightgreen.svg)](https://webchat.freenode.net/?channels=lnd)
