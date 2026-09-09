---
layout: post
title: "Cluster picking in D365 SCM — and the two gotchas that got me"
---

Had an interesting one this week: a client wanted cluster picking turned on for a specific subset of items, without touching how the rest of picking works. Sounds simple. It mostly was — except for two things that didn't show up until I actually tested it.

Quick context if you haven't touched cluster picking before: it lets a picker satisfy multiple orders from one visit to a location, instead of walking back and forth separately for each one. Big time saver at volume. But the client only wanted it for a small, specific set of items — everything else had to keep working exactly like today, manual pack-station selection and all.

<!-- Screenshot idea: simple two-box flow diagram — "Order released" branching into "Cluster template" and "Standard template" -->

**The setup**

Instead of modifying the existing work template, I split the flow into two: a new work template for cluster-eligible orders, sequenced ahead of the standard one, with its own work classification. The existing template stays completely untouched and just catches everything else.

For the put location, I used a directive code on the cluster template's put line, pointing at a new location directive. That's the actual mechanism worth knowing — a directive code makes D365 look up the location directive by code instead of by sequence, so you get a deterministic put location for the cluster flow without disturbing the sequence-based search everything else still uses. Not a workaround, just the documented way to do it.

**Gotcha #1 — mixed orders split into two work IDs**

My first version of the eligibility query filtered directly on item number, at the top level of the work template's header query. Worked fine — right up until I tested an order with one eligible item and one non-eligible item on it. Instead of one work ID covering both lines, I got two: the eligible line went to the cluster template, the other line fell through to standard.

Turns out the header query evaluates per line, not per order — so a plain filter on the line's own item can never see its sibling lines on the same order.

<!-- Screenshot idea: the work template's Edit query dialog, Joins tab, showing the join tree -->

Fixed it with an Exists join instead: join a second instance of the order lines through the order header, put the item condition on that joined instance, and set the join mode to Exists (not the default Inner Join). Now the question the query asks is "does this order have any eligible line" instead of "is this specific line eligible" — so every line on a qualifying order routes together, no matter which line actually triggered the match.

**Gotcha #2 — reusable totes instead of throwaway position IDs**

This one wasn't really a bug I hit, more a "wait, we can do better than this" moment once I got to testing the actual mobile flow. The physical setup uses totes on a cart for the cluster — and by default, the system auto-generates a new position ID for the cluster every single time. Fine, but it means the tote itself isn't really "known" to the system as anything — it's just wherever position 1, 2, 3 happens to land that run.

What we actually wanted: a fixed ID on each tote, scanned the same way every time, reusable run after run — and ideally scannable again at packing to pull up the right shipment, no separate lookup step.

Turned out to be two settings, neither of which reads as connected to this if you're just skimming the field descriptions: **Activate positions** off on the cluster profile, and **Generate license plate** off on the mobile device menu item. With both off, the system stops auto-assigning positions and instead prompts the picker to scan a target license plate for each cluster position — so a fixed, physical tote ID becomes the actual position identifier, not a disposable system-generated one. (Small bonus: turning off Activate positions also means the cluster doesn't need every position filled to start — useful on its own if demand doesn't always exactly match your cart size.)

The nice part only became obvious once I ran the full flow end to end: the same tote ID gets scanned again at the packing step to bring up the shipment to pack. Finish packing, tote's empty, it goes straight back into rotation for the next cluster. Picking and packing end up sharing one identifier instead of two disconnected systems.

<!-- Screenshot idea: the handheld's "scan target license plate" prompt during cluster creation — captures the moment the fixed tote ID replaces the auto-generated position, which is the whole point of this section. A second option, if you want two images here: the packing screen right after scanning the tote ID, showing the shipment it pulled up — that's the "payoff" shot. Worth a quick check before posting that neither screenshot shows a real order/cluster/work ID you don't want public. -->

**Takeaway**

None of this needed custom code — it's all standard work template, location directive, and cluster profile configuration. But routing a subset of items into cluster picking without disturbing anything else takes actually understanding where each decision lives in the config, not just following a checklist. Both gotchas above only showed up in real testing, not in the setup steps — worth remembering before calling something like this "done."

Got a similar problem in your own WMS setup? Get in touch.
