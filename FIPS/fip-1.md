---
fip: 1
title: Implement Issuance-based FCT Mechanism
author: Tom Lehman <tom@facet.org>, Michael Hirsch <michael@facet.org>
status: Draft
type: Standards Track
category: Core
created: 2025-05-21
---


## Abstract

This FIP proposes a comprehensive overhaul of the Facet protocol's native gas token, Facet Compute Token (FCT). The changes (1) derive a deterministic total‑supply cap via an issuance‑based halving schedule, (2) replace the 10,000‑block "stair‑step" rate adjustment with a dual‑threshold system that recalibrates the moment either the block limit *or* the issuance target is hit, moving from 10,000-block to 1,000-block periods, and (3) mint FCT in proportion to ETH burned for calldata, not raw gas units. Together, these measures aim to deliver predictable dilution, minimise issuance swings, and insulate mint costs from exogenous gas‑price spikes.

## Motivation

### Current System

For context, a review of how FCT works today: FCT issuance is controlled by a dynamic rate adjustment mechanism that operates in 10,000-block periods (approximately 1.4 days). Here's how it works:

- The system targets 400,000 FCT per 10,000-block adjustment period  
- Every 10,000-block period, the issuance rate adjusts based on how much FCT was minted in the previous period  
- If more than the target amount was minted, the rate decreases proportionally (up to a maximum change of 50%)  
- The target issuance rate halves every year to ensure a fixed total supply  
- The amount of FCT minted per transaction is based on the gas used for calldata, multiplied by the current issuance rate

### Problems

#### 1\. Uncertain Maximum Supply

The current time‑based halving calendar makes it impossible to know the maximum supply of FCT.

#### 2\. Delayed Rate Response and Resulting Issuance Instability

The current scheme updates the mint rate only after each fixed 10,000‑block window. Demand spikes within a window therefore enjoy an outdated rate until the window closes. This FIP addresses this by moving to 1,000-block periods and implementing dual-threshold adjustments.

#### 3\. Gas Price‑Driven Volatility in Minting Costs

Basing FCT output on data‑gas units exposes users to raw gas‑price swings. A single transaction's mint could vary by an order of magnitude solely because of L1 network congestion, complicating economic planning.

### Specification

#### Halving Schedule

Currently the FCT issuance rate halves every 2,628,000 blocks (approximately every year). With this FIP, halvings occur when the total minted FCT supply crosses specific thresholds, irrespective of the number of elapsed blocks, though, if the mechanism works as intended, halvings should continue to occur yearly.

* **Thresholds:**  
  * 1st Halving: 50% of Total Supply.  
  * 2nd Halving: 75% (50% \+ 25%) of Total Supply.  
  * Subsequent Halvings: Occur after minting half of the remaining supply towards the target.  
* **Effect:** Each halving event reduces the **Target FCT Issuance Per Period** by 50%.

#### Setting Total Supply

When the original FCT mechanism was designed, the goal was to target a maximum supply of 210 million. However, this cap was not an actual feature of the mechanism itself. The current mechanism includes no specific set cap on total supply, although there is still an implicit cap.

Therefore, we need a neutral way of choosing and capping total supply, since the current mechanism does not determine it. To do this, we'll build on a feature that's already built into the current mechanism: halving the issuance rate every year.

Yearly halvings imply 50% of the token will be issued after the first year. To set the max supply, we maintain this approach and specify that half of the total supply will be issued in the first year, starting from the protocol launch in December 2024\. We calculate the maximum supply accordingly:

* **Basis:** 50% of the total FCT supply will be minted by the time the first halving target block height is reached.  
* **First Halving Target Block Height:** 2,628,000 blocks.  
* **Calculation at Fork Block:**  
  * Let `B` \= blocks passed since genesis.  
  * Let `T_halving_blocks` \= 2,628,000 (target blocks for first halving).  
  * Let `M` \= actual FCT minted so far.  
  * Block Proportion \= `B` / `T_halving_blocks`.  
  * Expected Mint Proportion \= (`Block Proportion`) \* 0.50.  
  * **Inferred Total Supply \= `M` / `Expected Mint Proportion`**  
* **Example:**  
  * Assume the fork occurs at block `B = 1,182,600`.  
  * `T_halving_blocks` \= 2,628,000.  
  * Block Proportion \= 1,182,600 / 2,628,000 \= 0.45.  
  * Expected Mint Proportion \= 0.45 \* 0.50 \= 0.225.  
  * If `M` \= 140,000,000 FCT at this block, then Inferred Total Supply \= 140,000,000 / 0.225 \= 622,222,222.222 FCT.

#### Adjusted Per-Period Target Calculation

To ensure the first halving occurs as close to the original yearly goal as possible, we adjust the per-1,000-block period issuance targets such that hitting the targets will result in 50% of the Total Supply being issued by the first halving target block height (2,628,000).

* **Calculation:**  
  * Target FCT to mint by first halving \= `Inferred Total Supply` \* 0.50.  
  * Total number of 1k-block periods until first halving target \= `T_halving_blocks` / 1,000.  
  * **Initial Target FCT Issuance Per Period \= (`Target FCT to mint by first halving`) / (`Total number of periods until first halving target`)**  
* **Example:**  
  * Using values from previous example (`Inferred Total Supply` \= 622M FCT, `T_halving_blocks` \= 2,628,000):  
  * Target FCT by first halving \= 622,000,000 \* 0.50 \= 311,000,000 FCT.  
  * Total number of periods \= 2,628,000 / 1,000 \= 2,628.  
  * **Initial Target FCT Issuance Per Period \= 311,000,000 / 2,628 ≈ 118,341 FCT**.

#### Issuance Period Adjustments

##### Period Definition

After this FIP, each period ends when **either** of these triggers is reached:

1. **1,000 blocks** pass (changed from 10,000 blocks), **or**  
2. The **Target FCT Issuance Per Period** (as adjusted by halvings) is fully minted within that period. In our example above this value is 118,341 FCT.

For example, if the current target is 100,000 FCT per period, and you reach that 100,000 minted at block 800 of the period, the period ends immediately—no need to wait for 1,000 blocks.

##### Rate Adjustment Rule

Let:

- **oldRate** \= the FCT-per-ETH-burned rate for the *current* period.  
- **newRate** \= the rate for the *next* period.  
- **target** \= the per-period target FCT to be minted (e.g., 1,183,409).  
- **minted** \= how many FCT were actually minted by the time the period ended.  
- **blockCount** \= how many blocks elapsed in this period if the target was reached first.

Depending on which trigger ends the period, the new rate is calculated as follows:

##### 1\. If 1,000 Blocks Hit First (Under-issuance)

If the period times out at 1,000 blocks before hitting the target, we minted fewer FCT than intended (`minted < target`). The system **increases** the issuance rate for the next period.

```
newRate = oldRate * (target / minted), capped at 2 * oldRate
```

This ensures a proportional "catch-up," but never more than doubling in one step. If `minted == 0` newRate is `2 * oldRate`.

##### 2\. If the Target Is Minted First (Over-issuance)

If the entire per-period target was minted in fewer than 1,000 blocks, we minted faster than intended (`minted = target, blockCount < 1,000`). The system **decreases** the issuance rate for the next period.

```
newRate = oldRate * (blockCount / 1,000), capped at 0.5 * oldRate
```

This slows the next period's minting speed proportionally, but never by more than 50% in a single step.

Finally, there is a global minimum rate of `2**128 - 1` and a global maximum rate of `1`.

##### Example Scenarios

Assume a current target of **152,207 FCT** per period and a current **oldRate** of X.

1. **Under-issuance (Severe)**  
     
   - 1,000 blocks pass. Only 76,104 FCT minted (exactly half the target).  
   - target / minted \= 152,207 / 76,104 ≈ 2.0.  
   - newRate \= oldRate × 2.0, capped at 2.0 → newRate \= 2 × oldRate.

   

2. **Under-issuance (Partial)**  
     
   - 1,000 blocks pass. 130,000 FCT minted (85% of the target).  
   - target / minted \= 152,207 / 130,000 ≈ 1.17.  
   - newRate \= oldRate × 1.17 (since 1.17 \< 2, no cap needed).

   

3. **Over-issuance (Moderate)**  
     
   - 152,207 FCT minted in 900 blocks (100 blocks short of 1k).  
   - blockCount / 1,000 \= 900 / 1,000 \= 0.9.  
   - newRate \= oldRate × 0.9 (since 0.9 \> 0.5, no cap needed).

   

4. **Over-issuance (Severe)**  
     
   - 152,207 FCT minted in only 100 blocks.  
   - blockCount / 1,000 \= 100 / 1,000 \= 0.1.  
   - newRate would be oldRate × 0.1, but that's below the 0.5× cap. So newRate is forced to oldRate × 0.5.

#### Minting Mechanism

Today FCT is minted in proportion to gas consumed. Under this FIP it will be minted in proportion to ETH burned.

* **Initial Post-fork Rate**: Pre-fork FCT minted was proportional to gas units, not ETH burned. Therefore, to set the initial rate post-fork we take the final pre-fork rate in gas units and convert it to ETH burned by dividing it by the base fee of the final pre-fork block.  
* **Calculation:**  
  1. Calculate `Data Gas` (gas for calldata, excluding overheads).  
  2. Calculate `ETH Burned for Data = Data Gas * Block Base Fee`.  
  3. `FCT Minted = ETH Burned for Data * Current Issuance Rate`.

#### Constants

| Name | Value | Notes |
| :---- | :---- | :---- |
| `MAX_MINT_RATE` | `2**128 - 1` | In FCT-wei per ETH-wei burned |
| `MIN_MINT_RATE` | `1`  | In FCT-wei per ETH-wei burned |

#### New state persisted in `setL1BlockValuesEcotone()`

| Field | Type | Description |
| :---- | :---- | :---- |
| `fct_mint_rate` | `uint128` | current rate |
| `current_period_start_block` | `uint32` | L2 block number at start of period |
| `current_period_fct_minted` | `uint128` | FCT already minted in this period |
| `total_fct_minted` | `uint256` | cumulative supply |

#### 
