# Node Outages, Failover, and Split-Brain

**Date:** 2026-07-29
**Sources:**
- DDIA Chapter 5 (Replication) — handling node outages, failover, and its problems
- [Leader Election vs Consensus — Andrei Ochesel (2022)](https://ocheselandrei.github.io/2022/06/01/leader-election-vs-consensus.html)
- [GitHub: This week's availability (Sept 2012 incident) — Jesse Newland](https://github.blog/news-insights/the-library/github-availability-this-week/)
- [GitHub: Downtime last Saturday (Dec 2012 incident) — Mark Imbriaco](https://github.blog/news-insights/the-library/downtime-last-saturday/)
- [An Introduction to pg_auto_failover — Dimitri Fontaine (2021)](https://tapoueh.org/blog/2021/11/an-introduction-to-the-pg_auto_failover-project/)

**Related entries:**
- [016-single-leader-replication.md](016-single-leader-replication.md) — the single-leader model creates the single-point-of-failure this note is about; async replication is why failover can lose data
- [017-replication-logs.md](017-replication-logs.md) — catch-up recovery and failover both rely on the log-position bookkeeping described there; failover slots are why logical replication needs slot synchronization
- [001-backoff-and-jitter.md](001-backoff-and-jitter.md) — failure detection here uses timeouts, the same mechanism (and same false-positive risk) as retry timing
- [005-facebook-tao.md](005-facebook-tao.md) — TAO's automatic master-DB promotion on failure is exactly the failover process described here, and its per-shard single master is the split-brain risk this note dissects

> **Note on scope:** Third of four replication notes. Entry #016 gave the single-leader model and #017 the replication log. This note is about what happens when nodes *fail* — the recovery and failover machinery, and the ways it goes catastrophically wrong. Object-storage-backed databases (which reframe some of this) are Entry #019.

---

## The Problem Failover Exists to Solve

The single-leader model from Entry #016 has a built-in vulnerability we flagged and deferred: **all writes go through one leader, so if the leader dies, writes stop.** Follower failures are comparatively easy to handle; leader failure is the hard, dangerous case. This note works through both, in increasing order of difficulty and danger:

```
   Follower fails       →  catch-up recovery         (easy, routine)
   Leader fails         →  failover                  (hard, risky)
   Failover goes wrong  →  split-brain, data loss    (catastrophic)
```

The uncomfortable truth this note builds toward is that **failover cannot be made perfectly safe** — it's a chain of individually-fragile steps, and the deepest theory says a guarantee of "exactly one leader" is impossible in a real distributed system. So the goal isn't perfection; it's understanding the failure modes well enough to bound the damage.

---

## Follower Failure: Catch-Up Recovery

A follower crashing is the benign case, and it's handled by exactly the mechanism Entry #016 used to add a new follower. Every follower keeps a local log of the changes it has received from the leader, and it knows the **log position** of the last transaction it processed before it crashed (an LSN in PostgreSQL, a binlog coordinate in MySQL — see Entry #017).

```
   Follower was at log position 4471 when it crashed.
        │
        │  (leader kept accepting writes; is now at position 5230)
        ▼
   Follower recovers, reconnects to leader, says: "send me everything after 4471."
        │
        ▼
   Leader streams positions 4472 … 5230.  Follower applies them, catches up, resumes.
```

Because the follower knows precisely where it left off, recovery is deterministic and requires no coordination beyond "give me the backlog since position X." No data is at risk, the leader and other followers are unaffected, and the recovering follower simply rejoins once caught up. This is why follower failure is routine: the single-leader model's single authoritative log makes "where was I?" a question with an exact answer.

---

## Leader Failure: Failover, Step by Step

When the *leader* fails, there's no such graceful path, because the leader is the thing every write depends on. The response is **failover**: promote a follower to be the new leader, and redirect clients and remaining followers to it. Failover can be manual (an operator triggers it) or automatic. It sounds simple; it is not. Here are the steps, each of which hides a problem:

```
   1. DETECT      Determine that the leader has actually failed.
   2. CHOOSE      Select a follower to become the new leader.
   3. PROMOTE     Reconfigure that follower as leader; point writes at it.
   4. REDIRECT    Point clients and other followers at the new leader; ensure
                  the old leader stands down if/when it returns.
```

**Step 1 — Detecting failure — is guesswork.** There's no reliable way to tell "crashed" from "slow" or "briefly unreachable over the network." Systems use **timeouts**: the leader is presumed dead if it hasn't responded to heartbeats for some window (say 30 seconds). This is the same timeout-based detection as retry timing (Entry #001), and it carries the same danger of false positives. Set the timeout too long and you extend the outage while everyone waits; set it too short and a momentary network blip or GC pause triggers an *unnecessary* failover — which, as we'll see, is often *worse* than the transient problem it was reacting to.

**Step 2 — Choosing a new leader — can lose data.** The best candidate is the follower with the most up-to-date data (the highest log position), to minimize data loss. But recall from Entry #016 that async replication means followers can lag: the ideal new leader may still be missing the leader's most recent writes. Those writes — already acknowledged to clients — are simply gone. Worse, if the old leader comes back and rejoins, its now-orphaned writes may conflict with writes the new leader accepted in the meantime. The usual resolution is to *discard* the old leader's un-replicated writes, which violates clients' durability expectations. (This is why semi-synchronous replication from Entry #016 matters: guaranteeing ≥2 copies of each acknowledged write means the promoted follower has them.)

**Step 3 and 4 — Promotion and redirection — can go wrong operationally.** Databases, application servers, and clients all need to agree on who the new leader is. If some clients keep writing to the old leader while others write to the new one, you have two leaders — **split-brain**, the subject of the next section. And a subtle real-world hazard: if the old leader reboots and comes back up *still believing it's the leader* (because nothing told it otherwise), it will happily accept writes. Preventing this "zombie leader" is a core design requirement.

TAO (Entry #5) does exactly this dance: when a master DB fails, one of its slaves is automatically promoted, and writes during the switchover are failed back to clients rather than risked. The "fail the writes rather than guess" choice is a deliberate safety trade.

---

## Split-Brain: When Two Nodes Both Think They Lead

**Split-brain** is the nightmare outcome of failover: two nodes simultaneously believe they are the leader, and both accept writes. Because the whole point of the single-leader model (Entry #016) was to have one authoritative order of writes, two leaders means two divergent histories — and data corruption that can be impossible to reconcile automatically.

```
             network partition or failed failover
                          │
        ┌─────────────────┴─────────────────┐
        ▼                                    ▼
   ┌──────────┐                        ┌──────────┐
   │ Leader A  │◄── clients write      │ Leader B  │◄── other clients write
   │ (old)     │                        │ (new)     │
   └──────────┘                        └──────────┘
        │                                    │
        └──────► two divergent write histories ◄──────┘
                 (which one is "correct"? often neither cleanly)
```

The classic mechanism is a **network partition**: the leader is actually alive and healthy, but a network fault makes it *look* dead to the failover system. A new leader is promoted — but the old one is still up, still reachable by some clients, still accepting writes. Now both are live. This is why detection (step 1) is so fraught: a false "leader is dead" verdict during a partition is precisely what manufactures a split-brain.

### Fencing: The Standard Defense

Since you can't prevent an old leader from *thinking* it's still leader, the defense is to make its writes *ineffective* — to "fence it off." The canonical technique (from Martin Kleppmann, referenced by the leader-election article) is a **fencing token**: a number that strictly increases every time a new leader is elected.

```
   Leader A elected  → token 33.  Writes tagged "33".
   A pauses (GC/partition); failover elects Leader B → token 34.  Writes tagged "34".
   A resumes, still thinks it's leader, sends a write tagged "33".
        │
        ▼
   Shared storage / downstream service has already seen token 34.
   It REJECTS the write tagged "33"  ✗   (stale token — old zombie leader fenced off)
```

The crucial requirement, emphasized by both the leader-election article and Entry #019's object-storage work, is that **the downstream system must actually check the token.** The token only helps if the resource being written to cooperates — comparing the incoming token against the highest it has seen and rejecting anything stale. Fencing tokens are also central to the object-storage leader election in Entry #019, where the leader *epoch* plays exactly this role. A brutally direct alternative some systems use is **STONITH** ("Shoot The Other Node In The Head") — physically powering off or isolating the suspected-dead node before promoting a replacement, so it *cannot* come back as a zombie. STONITH is the fileserver strategy in the December 2012 GitHub incident below.

---

## Two Cautionary Tales from GitHub (2012)

GitHub published unusually candid post-mortems of two 2012 outages that are textbook illustrations of failover going wrong — one a database-replication split-brain, one a fileserver STONITH failure. They're worth studying because they show these aren't theoretical risks.

### September 2012: Automated MySQL Failover Causes a Split-Brain

GitHub ran a 3-node MySQL cluster with an active (read/write master) role and standby roles, coordinated by **Percona Replication Manager** on top of **Pacemaker/Heartbeat**. The cascade:

```
   1. A routine schema migration spiked database load.
   2. The load made the master fail Percona's HEALTH CHECKS.
   3. Percona auto-promoted another node — which had a COLD buffer pool,
      so it too performed poorly and failed health checks.
   4. The role bounced back to the original. Ops disabled health checks
      (Pacemaker "maintenance-mode") to stop the flapping.
   5. maintenance-mode then prevented a standby from correctly following its
      master → that standby silently fell out of date.
   6. Disabling maintenance-mode triggered a Pacemaker SEGFAULT → cluster
      state PARTITIONED (split-brain in the coordination layer itself).
   7. Two uncoordinated master elections ran. The STALE node from step 5 was
      elected master and began serving reads/writes.
   8. To stop data drift, ops POWERED IT OFF — taking all of github.com down.
```

The consequences were serious: because MySQL row IDs are used to look up records in Redis (for dashboards and repo routing), the data drift desynchronized cross-datastore relationships — events showed on wrong dashboards, repos routed wrong, and for **seven minutes 16 private repositories were accessible to non-collaborators.** GitHub's own conclusion named "the automated failover of our main production database" as the root cause, and their fix was blunt: **change the config so failover of the active role only happens when a human initiates it.** In every failure case, they noted, automated failover *shouldn't* have happened — the "cure" (unnecessary failover triggered by a transient load spike) was worse than the disease.

The lessons map directly onto the theory above: **false-positive failure detection** (step 2 — a slow-but-alive master looked dead), the **cold-cache promotion problem** (step 3 of generic failover — a promoted node performs badly), and **split-brain from a partitioned coordination layer** (steps 6–7). It's the whole chapter, live.

### December 2012: A Network Freeze Defeats STONITH on the Fileservers

The December incident was *not* a database failover — it was GitHub's **fileserver** tier (active/passive pairs using **DRBD** replication, coordinated by Pacemaker/Heartbeat, protected by **STONITH**). But it's the perfect complement, because it shows split-brain defense failing on the *network* it depends on.

```
   1. A network switch software upgrade caused ~90 seconds where traffic
      between access switches was BLOCKED (spanning-tree reconvergence).
   2. Fileserver pairs in different racks exceeded their heartbeat timeouts.
      Each node concluded its partner was dead and tried to take over.
   3. STONITH commands (to power off the "dead" partner) were issued — but
      some COULDN'T BE DELIVERED because the network itself was frozen.
   4. When the network recovered, many pairs had BOTH nodes wanting to be
      active. They raced to STONITH each other → many pairs ended with
      BOTH nodes shot and stopped.
   5. Recovery took 5+ hours: for each pair, determine which node had the
      most current filesystem state and safely reactivate only that one.
```

The instructive point: STONITH is a strong anti-split-brain mechanism, but it **depends on the network to deliver the kill command** — and the very failure it's guarding against (a network partition) is what prevents delivery. A defense that fails in exactly the scenario it's designed for. And recovery was slow precisely because of the async-replication data question from failover step 2: *which node has the most up-to-date state?* had to be answered by hand, pair by pair.

Together the two incidents teach the same meta-lesson: **automatic failover is a loaded gun.** It's essential for availability, but a false trigger or a partition turns the safety machinery into the cause of the outage.

---

## The Deep Question: Leader Election vs Consensus

If failover is this dangerous, why not just use a "proper" leader-election algorithm and be done? This is where the leader-election-vs-consensus article delivers a genuinely counterintuitive result worth sitting with.

Intuitively, **leader election** feels *easier* than **consensus** (getting a group of nodes to agree on a value). Leader election even looks like a special case of consensus — "agree on which node is leader." And indeed, consensus algorithms like Raft and Paxos *use* leader election internally. So there seems to be a chicken-and-egg puzzle: consensus needs leader election, leader election needs agreement (consensus).

The surprising theoretical result the article traces: **in the presence of failures, true (single-)leader election is actually *harder* than consensus.** The reasoning chain:

```
   FLP impossibility:  you cannot guarantee consensus in an asynchronous
                       system if even one node can fail.
        │  (worked around by "failure detectors" — timeouts that guess who's dead)
        ▼
   Consensus needs only a WEAK failure detector (may wrongly suspect a live
   node, but eventually stops). This is achievable in practice.
        │
        ▼
   True single-leader election needs a PERFECT failure detector (never wrongly
   suspects a live node). This is IMPOSSIBLE in a real distributed system.
```

The two-process illustration makes it concrete: leader *i* becomes slow, so *j* suspects it failed and makes itself leader — but *i* is alive and *still thinks it's the leader*. Now there are two leaders, violating the single-leader guarantee. You cannot rule this out without perfect failure detection, which cannot exist.

The resolution is the key insight: **Paxos and Raft don't actually require a single leader to be correct.** Paxos stays correct even with zero or multiple leaders (it just may not make *progress*); Raft stays correct through split votes. What they use is only **eventual leader election** — it converges to one leader eventually, but may temporarily have zero or two. The consensus algorithm is built to *absorb* those mistakes. So:

| | Guarantee | Achievable? |
|---|---|---|
| Ideal (single) leader election | Exactly one leader, always | **No** — needs perfect failure detection |
| Eventual leader election | Converges to one; may briefly have 0 or 2 | Yes |
| Consensus | Correct agreement despite imperfect leadership | Yes (tolerates the above) |

The practical takeaways the article lands: **(1)** only *eventual* leader election exists in the real world, so any leader-election scheme you build must stay correct even when there are temporarily two leaders (hence fencing tokens); **(2)** as AWS's builders' library bluntly puts it, *"it's not possible to guarantee that there is exactly one leader in the system"* — even delegating to a consensus service doesn't give you a guaranteed single leader, which is why the GC-pause-zombie-leader scenario is unavoidable and must be fenced; and **(3)** don't roll your own — use proven implementations (ZooKeeper/ZAB, Raft, Paxos), or, if your storage supports atomic conditional writes, you may be able to skip dedicated leader election entirely (the bridge to Entry #019).

---

## Doing Failover Well: pg_auto_failover

To ground all this, pg_auto_failover (a PostgreSQL HA tool from the Citus/Microsoft team) is a clean example of engineering *around* these hazards with pragmatic choices rather than heavyweight consensus. Its design decisions read like a checklist of the problems above.

**A separate Monitor node, deliberately not distributed consensus.** For N Postgres nodes, you run N+1 nodes: one **Monitor** plus the N Postgres nodes. The Monitor is itself a Postgres instance running a state machine, chosen over a Patroni/etcd-style consensus system because the authors' customers found consensus too complex to operate. The trade-off is explicit and honest: the Monitor is a single point of failure, so they optimize for *making it easy to replace* rather than eliminating it. This is the classic "consensus is stronger but operationally heavy" tension from the leader-election discussion, resolved toward simplicity.

```
                    ┌──────────────┐
                    │   MONITOR    │  runs health checks + a finite state machine;
                    │ (a Postgres  │  decides promotions transactionally (ACID)
                    │  extension)  │
                    └──────┬───────┘
              node-active  │  (each node reports its state ~every second)
              reports      │
          ┌───────────────┼────────────────┐
          ▼                                 ▼
   ┌──────────────┐                 ┌──────────────┐
   │  PRIMARY     │ ──replication──►│  SECONDARY   │
   │ (pg_autoctl) │                 │ (pg_autoctl) │
   └──────────────┘                 └──────────────┘
```

**Split-brain prevention by self-fencing.** The subtle case: a primary isolated on the *losing* side of a network partition. pg_auto_failover detects this when the node's `pg_autoctl` agent *cannot reach the Monitor* AND its `pg_stat_replication` view is unexpectedly empty (no standby is following it). In that situation the agent **stops its own Postgres locally** — it fences *itself* rather than risk being a zombie leader. The stated philosophy is telling: they prioritize avoiding split-brain *even if it means stopping a production database.* This directly addresses the September-2012 GitHub failure mode.

**Running `pg_autoctl` instead of `postgres` solves the zombie-reboot problem.** Because the supervising agent — not raw Postgres — is what starts on boot, a rebooted ex-primary does *not* automatically come back up as a primary (which would cause split-brain). It checks with the Monitor first and gets reconfigured as a standby. This is step 4 of failover (preventing the returning old leader from acting as leader) handled by construction.

**Correct-node selection and no-data-loss promotion.** The Monitor tracks each node's timeline/LSN (log position — Entry #017). If the chosen failover target isn't the most advanced standby, pg_auto_failover **fetches the missing WAL from a more advanced standby before promoting** (a "fast-forward") — directly mitigating failover step 2's data-loss risk. It uses standard Postgres tooling (`pg_rewind`, `pg_basebackup`, physical replication slots) to reconnect nodes cleanly after a switch.

**Tunable automation, including turning it off.** Each node has a *candidate priority*; setting priority to zero on all nodes means failover does *nothing* automatically and requires manual promotion — the exact posture GitHub adopted after September 2012. And the honest caveat: pg_auto_failover is robust against *any single-node failure*, but handles *multiple simultaneous* failures only on a best-effort basis — which, the author notes, is where full consensus systems are genuinely stronger.

The through-line: pg_auto_failover doesn't pretend failover is safe by magic. It picks a simple coordinator, makes the isolated node fence *itself*, prevents zombie reboots structurally, fast-forwards to avoid data loss, and lets operators dial automation down to zero. That's what "doing failover well" looks like — managing the hazards, not eliminating them.

---

## Key Takeaways

- **Follower failure is easy** — catch-up recovery: the follower knows its last log position (Entry #017), reconnects, and asks the leader for everything since. No coordination, no data at risk.
- **Leader failure requires failover** — detect, choose a new leader, promote, redirect — and *every step hides a problem*: detection is timeout-based guesswork (false positives cause needless failovers), choosing can lose async-un-replicated writes, and a returning old leader can become a zombie.
- **Split-brain** (two nodes both acting as leader, typically from a network partition making a live leader look dead) causes divergent histories and possibly irreconcilable corruption. Defenses: **fencing tokens** (a monotonic number that downstream storage *must* check and reject if stale) and **STONITH** (physically kill the suspected-dead node).
- **The GitHub 2012 incidents are the theory made real:** Sept — automated MySQL failover on a false health-check failure, then a coordination-layer split-brain that promoted a stale node and briefly exposed 16 private repos, leading GitHub to make failover manual-only. Dec — a network freeze prevented STONITH kill commands from being delivered, so fileserver pairs shot each other and both stopped. Automatic failover is a loaded gun.
- **Leader election vs consensus:** counterintuitively, *true single-leader* election is *harder* than consensus (it needs an impossible perfect failure detector). Only *eventual* leader election exists; Paxos/Raft stay correct despite temporarily having 0 or 2 leaders. So you can never assume exactly one leader — you must fence.
- **pg_auto_failover** shows pragmatic failover engineering: a simple separate Monitor (not consensus), an isolated primary that *fences itself*, `pg_autoctl` preventing zombie reboots, WAL fast-forward to avoid data loss, and tunable-to-zero automation.
