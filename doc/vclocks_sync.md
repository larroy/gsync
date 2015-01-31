# Peer to peer synchronization using vector clocks and repository update clocks

(c)2015 Pedro Larroy. CC BY-NC-SA 2.0

## Abstract

A method is proposed to detect concurrent changes, conflicts and causality violations in a large set of data files or key-value pairs which are shared and synchronized across a cluster of compute nodes in which the wall clock is not necesarily synchronized. The method proposed also guarantees that on reconnection only the list of files or keys that have changed since the last synchronization is transmitted.

## Introduction

When keeping in sync a set of files or key-value pairs across a cluster that don't have a synchronized clock, several problems arise:

1. Wall time can't be used to detect the causality of changes to a file
2. After a period of disconnection the nodes have to exchange and index of files which were modified since the last synchronization
3. We want to detect conflicts that arise when concurrent writes to a file happen from different nodes between synchronization points

From now on we will refer to files for simplicity although the concept is the same for key-value pairs, rows in a database or other pieces of arbitrary data.


## Proposed method

The method proposed here consists of using the following data structures:

Per repository:

1. An integer; *repository revision*  `RV` incremented each local change
2. A *repository update clock* `RUC`. With nodes as keys and repository revision as values. Described further later.

Per file:

1. A *vector clock* `VC` for each file in the repository
2. A *last changed by node* `LCN` equal to the *node id* that changed the file last.
3. A *last changed at node revision* `LCR` equal to the *repository revision* of the node that did the change. 

Each node in the cluster needs to have a *node id* `NID` which is unique across the cluster and equality comparable.

The entries in each file's *vector clock* are keyed by *node id* to track the causality of changes made by each node in a shared file.

## Node local changes

When a node `A` with the same *node id* changes, or detects a change in a file `F1` in its repository does the following operations atomically: 

1. Increments its *repository revision* value `RV(A)`
2. Updates the vector clock `VC(F1)` and increments the value of `VC(F1)(A)`
3. Sets `LCN` = `A`
4. Sets `LCR` = `RV(A)`

Thanks to each file's vector we can establish a partial order of causal events on the file or "happened before relation". We know that a copy of F1 `F1'` happened after `F1` iif:   

$$VC(F1)(n) < VC(F1')(n) \quad \forall n \in F1 \cup F1' $$

## Repository update clock

After a period without synchronization or between the detection of changes a node needs to know what files were changed by any other node without transmitting the full list of files and their version clocks which can be costly.

To solve this, each node keeps its own *repository update clock* which is keyed by *node id* and with the maximum value of the *last changed at node revision* for all known files. $$RUC(LCN(file_i)) = \max (RUC(LCN(file_i)), LCR(file_j)) \quad \forall F
$$

The *repository update clock* `RUC` is a one dimensional vector keyed by *node id* and with values the respective `LCR` value represents the logical state of updates across nodes as any changes or updates from other nodes will cause the *repository update clock* to be updated to reflect these changes. The causality relation of `RUC` is not relevant for us but it allows us to build a minimal list of updated files as a function of the `RUC` of any other node.

## Request for updates

When node `A` has ``RUC(A) = {A: 2, B: 4}`` it means that the last changes it knows about are `A`, at *repository revision* 2 and `B` at *repository revision* 4.

To request the updates from node `B`, it sends `RUC(A)` which then `B` compares to its own `RUC(B) = {A: 2, B: 7, C: 3}` now `B` knows that it needs to send the files that where last changed by `B` and `LCR > 4`, and the ones last changed by `C`.

The result of this is that only the list of files that were changed and its vector clocks needs to be sent, instead of the full list.

## Update mechanism

The vector clock of each file is used for causality violation which allows us to detect conflicts. The resolution of the conflict is delegated to a higher level software module or to the user itself.

## Acknowledgements
I would like to acknowledge the input and ideas of Steven Jewel.

## References

[vector clock]: http://en.wikipedia.org/wiki/Vector_clock
[Amazon's dynamo]: http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf
