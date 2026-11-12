# Week 5 assignment — Put a limit on the tunnel

Student ID (last 4 digits only): **1940**

Write each section in your own words, about your own runs. Quote the evidence files in
`assignment/evidence/` that support each point. Delete the instructions in `<!-- -->` as you go.

## 1. Analysis: the loop

I created the tunnel loop on s2 by changing its tunnel table entry for exit 3 (key 0003, handle 2) from port 3 to port 1, so it sends the packet back to s1 : `sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"`. 

s1 already sends exit 3 to s2, so one packet kept going back and forth between them. In `loop-before.txt`, s1 and s2 had handled 1264 and 1266 packets and 5 seconds later 1481 and 1480. The numbers kept rising from one packet, so the loop never stopped.

The IPv4 TTL does not stop a loop in this program. `ipv4_forward` decrements the TTL, but nothing checks it or drops the packet when it reaches 0. The TTL is an 8-bit number (0 to 255), so 0 − 1 becomes 255 and the packet keeps looping. 

## 2. Design: the hop limit

Field size and position: I added an 8-bit field called `hop_limit` at the end of the tunnel header after `dst_id`. This is because 8 bits (0 to 255) is the same size as the IPv4 TTL which is enough. I put it at the end to ensure `dst_id` stays in the same place, so an old switch can still read it.

Starting value: `send.py` starts `hop_limit` at 8. My network has only 2 switches, so 8 is more than enough for normal delivery and a loop will still stop quickly.

Decrement and drop: Each switch checks `hop_limit` in the ingress. If it is 0, the switch drops the packet. Otherwise, the switch subtracts 1 and forwards the packet. The switch checks before subtracting, so the value never goes below 0 and goes back to 255.

What else had to change: The tunnel header in `tunnel.p4`, so the parser and deparser read and write the new field. `send.py` adds the field to each packet and `receive.py` reads it and prints the hop limit that arrived.

## 3. Implementation

`tunnel.p4`: I added an 8-bit `hop_limit` field at the end of the tunnel header. In the ingress, if a packet has a tunnel header, the switch checks `hop_limit`. If it is 0, the packet is dropped. If not, the switch subtracts 1 and forwards it with `myTunnel_exact`. I did not change the parser or deparser because they follow the header definition, so they can read and write the new byte automatically.

`send.py`: I added the same `hop_limit` field at the end of the tunnel header to match `tunnel.p4`. I also added a `--hops` option to choose the starting value which is 8 by default.

`receive.py`: I added the same field, so it reads the header correctly and it now prints the `hop_limit` that arrived.

## 4. Evidence: it works

Tunnelled delivery (`delivery.txt`): the packet sent to tunnel exit 2 arrived at h2 and the packet sent to tunnel exit 3 arrived at h3. The switches used the tunnel number to choose where to send them, not the IP address. Both left with `hop_limit = 8` and arrived with 6 because s1 and s2 each subtracted 1.

Plain IPv4 (`delivery.txt`): `h1 ping -c 2 h3` still worked (0% loss, TTL 62).

The loop now ends (`loop-after.txt`): with the same loop as in section 1, the packet counts did not change after 5 seconds, so the loop stopped. For a second loop packet, s1's count went up by 5 and s2's by 4, so 9 arrivals in total. The packet was forwarded 8 times (`hop_limit` 8 down to 0) and dropped on the 9th arrival which matches my design. The large totals include packets from earlier runs because the logs are not cleared on restart.

## 5. Evaluation

<!-- a) The mixed rollout (mixed.txt): what did you predict before running s1 with your new
        program and s2 with tunnel_v1.p4? What actually happened to delivery, and to the loop?
        Explain every difference from your prediction, byte by byte if you need to.
     b) Propose a safe way to roll out a header change across a network of switches that can't
        all be upgraded at once. Compare at least two options (for example a version field, a
        new EtherType, or ordering fields so old switches still parse them).
     c) Loop protection in the data plane (this hop limit) versus the control plane (Week 3's
        check_paths()): what can each catch that the other can't? Why do real networks use both?
     d) What would this change cost on a hardware switch running at line rate? -->

## AI use

<!-- This assignment is Level 3 (Open): AI use is allowed if you declare it. State which tools you
     used (or "None") and what each did — for example "ChatGPT explained the bmv2 log format;
     the first draft of the hop-limit check came from Copilot, which I then changed and tested".
     Be ready to explain any part of your work yourself. -->

## Evidence files

| File | What it shows |
|---|---|
| `evidence/loop-before.txt` | The loop with your finished lab program: pass counts growing |
| `evidence/delivery.txt` | Tunnelled delivery with the hop limit, and a plain ping |
| `evidence/loop-after.txt` | The same loop with your hop limit: pass counts that stop |
| `evidence/mixed.txt` | The mixed rollout: s1 new, s2 old |
