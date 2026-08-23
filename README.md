# Reliable Transport Protocols

Implementations of three reliable data transfer protocols — Alternating-Bit (ABT), Go-Back-N (GBN), and Selective-Repeat (SR) — written in C++ against a simulated network layer that loses, corrupts, and delays packets.

Course project for EECE 7374, Northeastern University, Oct–Dec 2024.

## Files

| File | Description |
|---|---|
| `ABT.cpp` | Alternating-Bit (stop-and-wait, rdt3.0) |
| `GBN.cpp` | Go-Back-N with cumulative ACKs |
| `SR.cpp` | Selective-Repeat with per-packet ACKs and receiver buffering |
| `Final_Report_RDT.pdf` | Timeout design, timer implementation, and throughput experiments |

Each file implements only the transport-layer routines called by the provided simulator: sender-side `A_output`, `A_input`, `A_timerinterrupt`, `A_init` and receiver-side `B_input`, `B_init`.

## Design notes

**Corruption detection.** `compute_checksum()` sums the sequence number, ACK number, and each payload byte treated as an unsigned 8-bit integer. Any packet whose recomputed checksum disagrees with the carried value is treated as corrupt. ABT and GBN respond by re-sending the last valid ACK; SR drops silently and lets the sender's timer handle it.

**Timeout.** All three protocols use a fixed expiration of 11 time units. The value was chosen empirically: running ABT over a lossless, corruption-free channel with long inter-message intervals gave a mean RTT of 10.9 units across 1,000 transmissions. A timeout near the average RTT balances premature retransmission at low loss against wasted waiting at high loss.

**Buffering.** ABT uses a queue — at most one packet is ever in flight, so FIFO access suffices. GBN uses a vector, since a timeout retransmits the entire window and needs index access. SR uses hash maps keyed by sequence number on both sender and receiver, because out-of-order packets are ACKed, buffered, and retransmitted individually.

**Multiple software timers on one hardware timer (SR).** SR needs a logical timer per in-flight packet, but the simulator provides one. Software timers live in an `unordered_map<int, float>` mapping sequence number to expiration time, and the hardware timer is always synced to the packet expiring soonest. Starting a timer computes `get_sim_time() + TIMEOUT` and re-syncs the hardware timer if the new packet is now earliest. Stopping one — on ACK or on its own expiry — erases the entry and re-syncs if the removed timer was the earliest. On interrupt, every expired entry is collected, its packet retransmitted, and its expiry extended, then the hardware timer is set to the next earliest. Each packet is therefore retransmitted at exactly the time it would be with independent timers, rather than by a periodic bulk sweep.

## Experiments

Throughput measured at receiver B over 1,000 messages, mean inter-message time 50, corruption probability 0.2.

- **Loss sweep** — loss probabilities {0.1, 0.2, 0.4, 0.6, 0.8} at window sizes 10 and 50.
- **Window sweep** — window sizes {10, 50, 100, 200, 500} at loss probabilities {0.2, 0.5, 0.8}.

Findings:

- At high loss, GBN and SR both beat ABT. ABT has no pipelining, so every packet waits on the one before it.
- Larger windows become a liability for GBN at high loss: one early loss forces retransmission of the whole window. At loss 0.8 with a large window, GBN falls below ABT.
- SR degrades more gracefully but is not immune. At loss 0.8 with corruption 0.2, ACKs for buffered out-of-order packets are themselves frequently lost, driving redundant retransmission.
- A follow-up sweep of smaller SR windows at loss 0.8 found throughput peaking around a window size of 16 — large enough to keep packets in flight, small enough to limit wasted retransmission.

Full graphs and analysis are in `Final_Report_RDT.pdf`.

## Build and run

These files contain no `main()`. Each is compiled together with the simulator from the assignment template, which supplies `starttimer`, `stoptimer`, `tolayer3`, `tolayer5`, `getwinsize`, and `get_sim_time`:

```bash
g++ -o abt ABT.cpp <simulator source>
g++ -o gbn GBN.cpp <simulator source>
g++ -o sr  SR.cpp  <simulator source>
```

Run with the simulator's command-line parameters — seed, message count, mean inter-message time, loss probability, corruption probability, and trace level. GBN and SR additionally take a window size via `-w`:

```bash
./sr -s 1234 -m 1000 -t 50 -l 0.2 -c 0.2 -w 16 -v 0
```

Built and tested with g++ on Ubuntu (WSL).

## Authors

Kishore Kumar Thiyagarajan and Meenakshi Nandagopal.
