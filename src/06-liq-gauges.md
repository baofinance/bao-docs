# Bao Liquidity Gauges & BAO emissions

Bao Incentivizes Liquidity providers with BAO token emissions using liquidity gauges. The liquidity gauges the DAO chooses to incentivize are associated to liquidity pools necessary for Bao synthetics markets to work well.

- liquidity gauges: Measures liquidity provided by users over time, in order to distribute BAO and other rewards

- Gauge Controller: Central controller that maintains a list of gauges, weights and type weights, and coordinates the rate of BAO production for each gauge

- Minter: BAO minting contract, generates new BAO according to the liquidity gauges


## BAO Inflation

BAO follows a piecewise linear inflation schedule. The inflation is reduced by 2^(1/4) each year. Each time the inflation reduces, a new mining epoch starts.

### Intial Supply

The initial supply of BAO is ~1.2 billion tokens, which is X% of the eventual  supply of billion tokens. All of these initial tokens are gradually vested (with every block). The initial inflation rate which supports the above inflation schedule is X (X.XX millions per year). All of the inflation is distributed to Bao liquidity providers, according to measurements taken by the liquidity gauges. During the first year, the approximate inflow into circulating supply is X million BAO per day. The initial circulating supply is 0.

## Liquidity Gauges

Inflation is directed to users who provide liquidity within the protocol. This usage is measured via “Liquidity Gauge” contracts. Each pool has an individual liquidity gauge. The Gauge Controller maintains a list of gauges and their types, with the weights of each gauge and type.

To measure liquidity over time, the user deposits their LP tokens into the liquidity gauge. Coin rates which the gauge is getting depends on current inflation rate, gauge weight, and gauge type weights. Each user receives a share of newly minted BAO proportional to the amount of LP tokens locked. Additionally, rewards may be boosted by up to factor of 2.5 if the user vote-locks tokens for Curve governance in the Voting Escrow contract.

Suppose we have the inflation rate  changing with every epoch (1 year), gauge weight  and gauge type weight . Then, all the gauge handles the stream of inflation with the rate  which it can update every time , or mining epoch changes.

To calculate a user’s share of , we must calculate the integral: 
 
 where  is the balance supplied by the user (measured in LP tokens) and  is total liquidity supplied by users, depending on the time ; the value  gives the amount of tokens which the user has to have minted to them. The user’s balance  changes every time the user  makes a deposit or withdrawal, and  changes every time _any_ user makes a deposit or withdrawal (so  can change many times in between two events for the user . In the liquidity gauge contract, the vaule of  is recorded per-user in the public integrate_fraction mapping.

To avoid requiring that all users to checkpoint periodically, we keep recording values of the following integral (named integrate_inv_supply in the contract):

 

The value of  is recorded at any point any user deposits or withdraws, as well as every time the rate  changes (either due to weight change or change of mining epoch).

When a user deposits or withdraws, the change in  can be calculated as the current (before user’s action) value of  multiplied by the pre-action user’s balance, and sumed up across the user’s balances:  The per-user integral is possible to repalce with this sum because  changed for all times between  and .

## Boosting 

In order to incentivize users to participate in governance, and additionally create stickiness for liquidity, we implement the following mechanism. A user’s balance, counted in the liquidity gauge, gets boosted by users locking BAO tokens in Voting Escrow contract, depending on their vote weight : 
 
 The value of  is taken at the time the user performs any action (deposit, withdrawal, withdrawal of minted BAO tokens) and is applied until the next action this user performs.

If no users vote-lock any BAO (or simply don’t have any), the inflation will simply be distributed proportionally to the liquidity  each one of them provided. However, if a user stakes enough BAO, they are able to boost their stream of BAO by up to factor of 2.5 (reducing it slightly for all users who are not doing that).

Implementation details are such that a user gets the boost at the time of the last action or checkpoint. Since the voting power decreases with time, it is favorable for users to apply a boost and do no further actions until they vote-lock more tokens. However, once the vote-lock expires, everyone can “kick” the user by creating a checkpoint for that user and, essentially, resetting the user to no boost if they have no voting power at that point already.

Finally, the gauge is supposed to not miss a full year of inflation (e.g. if there were no interactions with the gauge for the full year). If that ever happens, the abandoned gauge gets less BAO.

## Gauge Weight Voting

Users can allocate their veBAO towards one or more liquidity gauges. Gauges receive a fraction of newly minted BAO tokens proportional to how much veBAO the gauge is allocated. Each user with a veBAO balance can change their preference at any time.

When a user applies a new weight vote, it gets applied at the start of the next epoch week. The weight vote for any one gauge cannot be changed more often than once in 10 days.

## Gauge Controller

The “Gauge Controller” maintains a list of gauges and their types, with the weights of each gauge and type. In order to implement weight voting, GaugeController has to include parameters handling linear character of voting power each user has.

GaugeController records points (bias + slope) per gauge in vote_points, and _scheduled_ changes in biases and slopes for those points in vote_bias_changes and vote_slope_changes. New changes are applied at the start of each epoch week.

Per-user, per-gauge slopes are stored in vote_user_slopes, along with the power the user has used and the time their vote-lock ends.

The totals for slopes and biases for vote weight per gauge, and sums of those per type, are scheduled / recorded for the next week, as well as the points when voting power gets to 0 at lock expiration for some of users.

When a user changes their gauge weight vote, the change is scheduled for the next epoch week, not immediately. This reduces the number of reads from storage which must to be performed by each user: it is proportional to the number of weeks since the last change rather than the number of interactions from other users.

## Gauge Types

Each liquidity gauge is assigned a type within the gauge controller. Grouping gauges by type allows the DAO to adjust the emissions according to type, making it possible to e.g. end all emissions for a single type.

Currently active gauge types are as follows:
- Ethereum Stable pools
- Ethereum Crypto pools

## LiquidityGauge (Specification)

### Querying Gauge Information

### Querying User Information

### Checkpoints

### Deposits and withdrawals

### Killing the Gauge


## LiquidityGaugeReward (Specification)

### Querying Reward Information

### Claiming Rewards



## LiquidityGaugeV2 (Specification)

### Querying Reward Information

### Approvals and Transfers

### Checking and Claiming Rewards

### Setting the Rewards Contract



## LiquidityGaugeV3 (Specification)

### Querying Reward Information

### Checking and Claiming Rewards


## GaugeController (Specification)

### Querying Gauge and Type Weights

### Vote-Weighting

### Adding New Gauges and Types


## Minter (Specification)

### Minting BAO