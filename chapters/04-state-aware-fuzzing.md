# State-Aware Fuzzing of Consensus Implementations

Several research fuzzers target blockchain consensus implementations directly. They differ from ordinary input fuzzers in one way that matters: a consensus message is only interesting if it fits the receiver's current state. A prevote for height 9,000 sent to a node at height 12 is dropped at the first check. Random bytes almost never reach deep logic.

| Work | Idea |
|---|---|
| LOKI (NDSS 2023) | a fuzzing node joins the network, learns consensus state from messages, and generates state-aware malicious messages |
| Tyr (IEEE S&P 2023) | behaviour divergence models to detect consensus failures across nodes |
| Fluffy (OSDI 2021) | differential fuzzing of Ethereum execution clients for consensus bugs |

## Architecture of a fuzzing node

```
real network of N honest nodes (local testnet)
       ▲          ▲
       │ messages │
  fuzzing node (valid identity, possibly a validator)
    - tracks protocol state from observed messages
    - mutates/generates messages consistent with current height/round
    - monitors: crashes, stalls, divergent commits, resource use
```

The fuzzing node holds valid keys, so its messages pass signature checks. That is the point: it emulates a Byzantine validator, which the protocol must tolerate up to the fault threshold, rather than an outsider whose messages are discarded.

## Mutation strategies

| Strategy | Example on a CometBFT prevote | What it targets |
|---|---|---|
| Field boundary | height = current + 1, round = 2³¹ − 1 | integer handling, future-round logic |
| State replay | resend last round's prevote in the current round | round/height checks |
| Equivocation | two prevotes for different block IDs, same round | evidence handling |
| Type confusion | prevote with a precommit-only field set | decoder and validation paths |
| Timing | deliver just before or after a timeout fires | timeout races |

Coverage feedback helps: instrument the honest nodes (Go's coverage tooling for CometBFT, for example) and keep mutations that reach new code, as ordinary greybox fuzzers do.

## Oracles

- Node crash or panic.
- Liveness: no new block within T.
- Safety: different blocks committed at the same height across nodes.
- Resource: memory or CPU explosion from specific messages.

## Worked example

> **Execution status:** Not executed in this book's CI; verify locally. CometBFT command names and ports change between versions; check `cometbft --help` and the repository's localnet instructions.

Set up a four-node CometBFT localnet and attach the oracles from Chapter 1. Building the fuzzing node itself is Exercise 1; this example gives you the harness it plugs into.

**Step 1: generate a 4-validator testnet.**

```bash
cometbft testnet --v 4 --o ./localnet --populate-persistent-peers
ls localnet        # node0 node1 node2 node3
```

The CometBFT repository also provides a Docker-based localnet (see its `Makefile` and `networks/local` directory). Use either, but keep everything on your machine.

**Step 2: start nodes** with distinct ports, using a test ABCI app such as the built-in `kvstore`:

```bash
for i in 0 1 2 3; do
  cometbft start --home localnet/node$i --proxy_app=kvstore \
    --rpc.laddr tcp://127.0.0.1:$((26657+i*10)) \
    --p2p.laddr tcp://0.0.0.0:$((26656+i*10)) &
done
```

With `--populate-persistent-peers`, the generated configs reference peer addresses; if you change ports, edit `persistent_peers` in each `config.toml` to match.

**Step 3: poll commits and feed the oracle.** Each node's RPC returns committed blocks:

```python
import requests, time
RPC = [f"http://127.0.0.1:{26657+i*10}" for i in range(4)]
logs = {f"n{i}": {} for i in range(4)}
progress = {k: time.time() for k in logs}
while True:
    for i, url in enumerate(RPC):
        st = requests.get(f"{url}/status", timeout=2).json()["result"]["sync_info"]
        h = int(st["latest_block_height"]); n = f"n{i}"
        if h not in logs[n]:
            logs[n][h] = st["latest_block_hash"]; progress[n] = time.time()
    for p in check(logs, time.time(), progress, timeout=30):   # from Chapter 1
        if p != "OK": print(p)
    time.sleep(1)
```

Expected on a healthy network: no output. Stop one node with `kill` and, after 30 seconds, a liveness line for that node should appear while the other three keep committing (3 of 4 voting power exceeds two thirds).

**Step 4: add the fuzzing node.** Run a fifth process that connects as a peer with one validator's key (stop that validator's honest process first, so you have one Byzantine node, not an extra one), decodes incoming votes to learn the current height and round, and sends mutated votes from the table above. Every crash, stall or conflict the oracle prints is a candidate finding; reproduce it twice before reporting it privately to the CometBFT team.

## Common pitfalls

- **Fuzzing from outside the validator set.** Unsigned or wrongly signed messages are rejected immediately; you test only the signature check.
- **Running the fuzzing node alongside its honest twin by accident.** Then you are running Twins, not fuzzing, and may exceed f.
- **Stateless mutation.** Messages for the wrong height never reach interesting code.
- **Ignoring resource oracles.** Memory growth from retained messages is a common real bug class.
- **Reporting the first crash without minimising.** Reduce the message sequence to the shortest that reproduces.

## Exercises

1. Set up a 4-node local CometBFT testnet and write a fuzzing node that sends mutated prevotes.
2. Implement a liveness oracle and a safety oracle.
3. Read LOKI's evaluation and list the bug classes it found.
4. Add Go coverage instrumentation to one honest node and keep only mutations that increase coverage.

## Further reading

- Ma et al., "LOKI: State-Aware Fuzzing Framework for the Implementation of Blockchain Consensus Protocols", NDSS 2023
- Chen et al., "Tyr: Finding Consensus Failure Bugs in Blockchain System with Behaviour Divergent Model", IEEE S&P 2023
- Yang et al., "Finding Consensus Bugs in Ethereum via Multi-transaction Differential Fuzzing" (Fluffy), OSDI 2021
- CometBFT documentation: <https://docs.cometbft.com>
