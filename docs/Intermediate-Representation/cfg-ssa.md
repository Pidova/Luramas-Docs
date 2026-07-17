---
id: cfg-ssa
title: Control Flow & SSA
sidebar_position: 2
---

# Control Flow & SSA

The analysis layer beneath the [passes](./passes.md). 
The CFG gives the passes a control structure to reason about; SSA gives every value a single definition so propagation and elimination can be right. 
This page explains how both are built.

## Control-flow graph

Before anything is optimized, the lifter reconstructs control flow in **ir/lifter/generation/cfg/**. 
The output is a **cfg**: an entry block, an ordered block list, a predecessor map, and per-block high-level scope ids.

### Blocks

Each **block** carries up to three typed successors, which is what makes edge handling precise:

* **fall** - The block below, reached by falling through.
* **then** - The taken side of a conditional.
* **jump** - An unconditional or computed jump target.

Each successor also carries an **edge_kind**: **jump** for a normal edge, **back** for a loop back-edge, and every block records its **entry** / **ending** addresses and its **node_range**. 
Blocks expose **get_successors()**, **get_block_successors()**, and **dominant_successor_edge(target)** so passes can walk edges by kind. 
The **cfg** orders its **blocks** vector by dominance (fall -> then -> jump), which gives passes a predictable iteration order.

### How the CFG is built

**compute** splits the statement stream into basic blocks at leaders (branch targets, fall-through after branches, labels), wires each block's fall/then/jump successors from the branch instructions, and fills the predecessor map. 
**sort::topological** produces a traversal order (optionally ignoring back-edges so loops don't create cycles), and 
**traversal** / **iterate** provide the walking helpers passes rely on. 
The CFG is recomputed as passes mutate the code, so control-flow passes always see current structure.

## Dominators

SSA needs dominance, so the CFG layer computes it with standard algorithms from Boost.Graph:

* **immediate_dominators** runs the **Lengauer-Tarjan** dominator-tree algorithm over the block graph, giving each block its immediate dominator (the last block every path from entry must pass through before reaching it).
* **dominance_frontier** computes, for each block, the set of blocks where its dominance *ends*: the join points just outside its dominated region. The implementation is the classic frontier climb: for every join point **b**, walk each predecessor up the dominator tree until reaching **IDom(b)**, adding **b** to each block's frontier along the way.
* **dominator_tree** inverts the immediate-dominator map into a children-per-node tree.

* Dominators - Lengauer-Tarjan, near-linear in edges.
* Dominance frontier - one climb per predecessor of each join point.

## SSA

Static Single Assignment is the backbone of the whole IR. 
**ir/lifter/generation/ssa/** renames each register write into a fresh virtual register, so every value has exactly one definition and a set of uses. 

With that in place we can imagine passes as:
* **constant propagation** - "replace a use with its unique definition," 
* **dead-store elimination** -  "a definition with no uses"
* **dead-code elimination** - "a statement whose only effect is a dead definition."

The passes stay simple because SSA does all the heavy lifting.

### How SSA is built

The builder (**generate(nodes, cfg)**) follows the standard Cytron construction in two phases:

**Place phi nodes**:
For each register, collect the blocks that define it (**def_blocks**). 
Propagate a worklist with those blocks, and for each block popped, insert a phi for the register at every block in its dominance frontier; 
then add those frontier blocks to the worklist, since a phi is itself a definition and can force further phis.
This is the iterated-dominance-frontier algorithm; the worklist runs until it drains.

**Rename**: 
Walk the dominator tree renaming each definition to a new virtual register and each use to the version currently in scope, emplacing versions across edges into the phi operands.

### What SSA tracks

Per node the builder records:

* **lvalues** - registers this statement writes, mapped to their new virtual registers.
* **rvalues** - registers this statement reads, as an **ssa_rv_reg** that is either a **scalar** (one version) or a **phi** (several versions merged at a join).
* **locals** - values local to the current block.

An **ssa_rv_reg** also contains **fvalid** (whether the version exists in the original: flagging is faster than deleting) **fset_unknown** (explicitly unknown flag). 
The builder also tracks captured registers and upvalue volatility: the edge cases that make naive SSA wrong on real code so the passes don't have to.

## Using SSA in a pass

SSA is used in passes for variable optimizations. Passes rarely rebuild SSA by hand; See **[Helper Functions](./helpers.md#ssa-queries-toolsssa)**.
