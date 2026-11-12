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

<!-- Summarise what you changed in tunnel.p4, send.py and receive.py. The pull request shows
     the code itself: here, explain anything a reviewer might not see from the diff alone. -->

## 4. Evidence: it works

<!-- Show, with your evidence files:
     - tunnelled delivery to h2 and h3, and the hop limit the receivers saw (delivery.txt);
     - plain IPv4 still works (delivery.txt);
     - the same loop as in section 1 now ends, and after how many passes — does that match
       your design? (loop-after.txt) -->

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
