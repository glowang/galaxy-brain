# Whitelist & Blocks Only Interactions

* Version message has a "relay" bool.

  Transaction relay flag. If 0x00, no inv messages or tx messages announcing new
  transactions should be sent to this client until it sends a filterload message
  or filterclear message. If the relay field is not present or is set to 0x01,
  this node wants inv messages and tx messages announcing new transactions.


### original blocks only test

* node is started up in `-blocksonly` mode.
* node.getnetworkinfo()['localrelay'] is False.
* p2p connection sends txn to the node, then gets disconnected

* new connection is added
* node.getpeerinfo()[0]['relaytxes'] is True.
* submit transaction to node via `sendrawtransaction`
* confirm p2p connection gets the transaction

* node restarted with `-whitelist=127.0.01` & `-whitelistforcerelay` & in
  `-blocksonly` mode.
* two connections added
* about conn 1: node.getpeerinfo()[0]['whitelisted'] = True,
                                     ['permissions'] = ['noban', 'forcerelay',
                                     'relay', 'mempool']
* conn1 sends txn via P2P
* confirm 2nd connection gets the transaction

### blocksonly mode

* `-blocksonly` help text: Whether to reject transactions from network peers. Automatic
  broadcast and rebroadcast of any transactions from inbound peers is disabled,
  unless '-whitelistforcerelay' is '1', in which case whitelisted peers'
  transactions will be relayed. RPC transactions are not affected.
  default = False.

* `g_relay_txes` set to false if node is started in `-blocksonly` mode
* this is the only way `g_relay_txes` is set.

* rpc: getnetworkinfo['localrelay'] -> returns `g_relay_txes`

`g_relay_txes` is checked in three places in `net_processing`:
1. `PushNodeVersion`
   - in version message, the relay bool is set to true if `g_relay_txes` & it's
     not a `block-relay-only` connection.

2. `ProcessMessage#INV`
   - disconnect the peer if (`fBlocksOnly`)
     -> node is in `-blocksonly` mode, or its a `blockrelay` connection
     -> AND node is not whitelisted

   - this means:
     if you started in `-blocksonly` mode, you will still accept transactions
     from whitelisted peers

     if you have a `blockrelay` connection with a whitelisted peer, you will
     still accept transactions

3. `ProcessMessage#TX`
    - disconnect the peer if
      -> node is in `-blocksonly` mode & peer isn't whitelisted
      -> OR peer is a `blockrelay` connection

### whitelist

* `-whitebind` help text: Bind to given address and whitelist peers connecting to it.
  Allowed permissions are:
    bloomfilter (allow requesting BIP37 filtered blocks and transactions)
    noban (do not ban for misbehavior)
    forcerelay (relay transactions that are already in the mempool; implies relay)
    relay (relay even in -blocksonly mode)
    mempool (allow requesting BIP35 mempool contents)

  default = noban, mempool, relay

* `-whitelist` help text: Whitelist peers connecting from the given IP address
  or CIDR notated network. Uses same permissions as `-whitebind.`

* `-whitelistrelay` help text: Add 'relay' permission to whitelisted inbound
   peers with default permissions. This will accept relayed transactions even
   when not relaying transactions

  default = true

  sets the `PF_RELAY` flag to true for this peer. "Relay and accept
  transactions from this peer, even if `-blocksonly` is true"

* `-whitelistforcerelay` help text: Add 'forcerelay' permission to whitelisted
  inbound peers with default permissions. This will relay transactions even if
  the transactions were already in the mempool.

  default = false

  sets the `PF_FORCERELAY` flag to true for this peer. "Always relay
  transactions from this peer, even if already in mempool. forcerelay implies
  relay."


### interactions between blocksonly & whitelist

* if -blocksonly is set, set -whitelistrelay to false.
* if -whitelistforcerelay is set, set -whitelistrelay to true.

### P2P & RPC

P2P: version message, relay bool
-> set to false if node is in `blocksonly` mode, or connection type is
  `block-relay`

RPC: getnetworkinfo['localrelay']
-> returns `g_relay_txes`. indicates if the node is in `-blocksonly` mode.
-> help text: true if transaction relay is requested from peers


### questions
* so you send txns to whitelisted peers? but if version said its block relay
  only connection, why wouldn't they disconnect you? how do they know there's a
  whitelist overwrite going on?

* if you're in blocksonly mode, do you still send out locally submitted
  transactions? whats the difference if both indicate the same in the version
  message?

* trace the receiving logic. what do you do when you receive a version message
  saying block-relay-only connection?

* is there somewhere that sets the relaytxes part of version message to true if
  there's whitelist behavior going on?

* so in blocksonly mode, if you try to submit a txn to a peer, its not just bad
  for privacy, they disconnect you? `doc/reduce-traffic.md` doesn't mention
  that in the `-blocksonly` section

### scenarios
1. start a node in `blocksonly` mode. make a connection. send a transaction
   to them -> they disconnect.

2. start a node in `blocksonly` mode. make a whitelisted connection. send a
   transaction to them -> they disconnect.

3. start a node in `blocksonly` mode. make a connection with `-whitelistrelay`.
   send a txn -> they don't disconnect? but also don't get the txn?

--

4. start a node in `blocksonly` mode. make a connection with `-whitelistforcerelay`.
   send a txn. do they disconnect?

5. start a node in normal mode. make a `blocksrelay` connection. send a
   transaction to them. do they disconnect?

6. start a node in normal mode. make a `blocksrelay` whitelisted connection.
   send a transaction to them. do they disconnect?