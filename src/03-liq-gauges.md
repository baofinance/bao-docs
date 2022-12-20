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

### notes

- Source [here](https://etherscan.io/address/0x840e75261c2934f33c8766f6ea6235330dc1d72d#code)

The “Gauge Controller” maintains a list of gauges and their types, with the weights of each gauge and type. In order to implement weight voting, GaugeController has to include parameters handling linear character of voting power each user has.

GaugeController records points (bias + slope) per gauge in vote_points, and _scheduled_ changes in biases and slopes for those points in vote_bias_changes and vote_slope_changes. New changes are applied at the start of each epoch week.

Per-user, per-gauge slopes are stored in vote_user_slopes, along with the power the user has used and the time their vote-lock ends.

The totals for slopes and biases for vote weight per gauge, and sums of those per type, are scheduled / recorded for the next week, as well as the points when voting power gets to 0 at lock expiration for some of users.

When a user changes their gauge weight vote, the change is scheduled for the next epoch week, not immediately. This reduces the number of reads from storage which must to be performed by each user: it is proportional to the number of weeks since the last change rather than the number of interactions from other users.

### reading



### writing



## LiquidityGauges 

### notes

- Source [here](https://etherscan.io/address/0xe7f3a90aee824a55b0f8969b6e29698966ee0191#code)
- 3 gauges currently in existence including...
- BAO-ETH UNI LP (0xe7f3a90aee824a55b0f8969b6e29698966ee0191)
- bSTBL-DAI CRV LP (0x675F82DF9e2fC99F8E18D0134eDA68F9232c0Af9)
- baoUSD-3CRV CRV LP (0x0a39eE038AcA8363EDB6876d586c5c7B9336a562)

### Querying Gauge Information

lp_token(), returns the LP token associated to the liquidity gauge

totalSupply(), returns the total amount of LP tokens that are currently deposited into the gauge

working_supply(), returns the “working supply” of the gauge - the effective total LP token amount after all deposits have their boost accounted for (up to 2.5x)

### Querying User Information

balanceOf(), returns the current amount of LP tokens that addr has deposited into the gauge

working_balances(), returns the caller's working balance which applies whatever boost the caller has

claimable_tokens(), returns the claimable amount of BAO tokens for the caller

integrate_fraction(), returns the total amount of BAO, both mintable and already minted, that has been allocated to address from this gauge.

### Checkpoints

LiquidityGauge.user_checkpoint(addr: address)→ bool: nonpayable

Record a checkpoint for addr, updating their boost.

Only callable by addr or Minter - you cannot trigger a checkpoint for another user.

### Deposits and withdrawals

```deposit(amount: uint256, receiver: address = msg.sender): nonpayable```

Deposit LP tokens into the gauge.

Prior to depositing, ensure that the gauge has been approved to transfer amount LP tokens on behalf of the caller.

amount: Amount of tokens to deposit

receiver: Address to deposit for. If not given, defaults to the caller. If specified, the caller must have been previous approved via approved_to_deposit

```withdraw(amount: uint256): nonpayable```

Withdraw LP tokens from the gauge.

amount: Amount of tokens to withdraw

```approved_to_deposit(caller: address, receiver: address)→ bool: view```

Return the approval status for caller to deposit LP tokens into the gauge on behalf of receiver

```set_approve_deposit(depositor: address, can_deposit: bool): nonpayable```

Approval or revoke approval for another address to deposit into the gauge on behalf of the caller.

depositor: Address to set approval for

can_deposit: Boolean - can this address deposit on behalf of the caller?

### Killing the Gauge

```kill_me(): nonpayable```

Toggle the killed status of the gauge.

This function may only be called by the ownership or emergency admins within the DAO.

A gauge that has been killed is unable to mint BAO. Any gauge weight given to a killed gauge effectively burns BAO.

### Reward Info

```reward_contract()→ address: view```

The address of the staking rewards contract that LP tokens are staked into.

```rewarded_tokens(idx: uint256)→ address: view```

Getter for an array of rewarded tokens currently being received by reward_contract.

The contract is capable of handling up to eight reward tokens at once - if there are less than eight currently active, some values will return as ZERO_ADDRESS.

```transfer(_to: address, _value: uint256)→ bool:```

Transfers gauge deposit from the caller to _to.

This is the equivalent of calling withdraw(_value) followed by deposit(_value, _to). Pending reward tokens for both the sender and receiver are also claimed during the transfer.

Returns True on success. Reverts on failure.

```transferFrom(_from: address, _to: address, _value: uint256)→ bool:```

Transfers a gauge deposit between _from and _to.

The caller must have previously been approved to transfer at least _value tokens on behalf of _from. Pending reward tokens for both the sender and receiver are also claimed during the transfer.

Returns True on success. Reverts on failure.

```approve(_spender: address, _value: uint256)→ bool:```

Approve the passed address to transfer the specified amount of tokens on behalf of the caller.

Returns True on success. Reverts on failure.

```claimable_reward(_addr: address, _token: address)→ uint256: nonpayable```

Get the number of claimable reward tokens for a user.

This function determines the claimable reward by actually claiming and then returning the received amount. As such, it is state changing and only of use to off-chain integrators. The mutability should be manually changed to view within the ABI.

_addr Account to get reward amount for

_token Token to get reward amount for

Returns the number of tokens currently claimable for the given address.

```claim_rewards(_addr: address = msg.sender): nonpayable```

Claim all available reward tokens for _addr. If no address is given, defaults to the caller.

```claim_historic_rewards(_reward_tokens: address[8], _addr: address = msg.sender): nonpayable```

Claim reward tokens available from a previously-set staking contract.

_reward_tokens: Array of reward token addresses to claim

_addr: Address to claim for. If none is given, defaults to the caller.

```set_rewards(_reward_contract: address, _sigs: bytes32, _reward_tokens: address[8]): nonpayable```

Set the active reward contract.

_reward_contract: Address of the staking contract. Set to ZERO_ADDRESS if staking rewards are being removed.

_sigs: A concatenation of three four-byte function signatures: stake, withdraw and getReward. The signatures are then right padded with empty bytes. See the example below for more information on how to prepare this data.

_reward_tokens: Array of rewards tokens received from the staking contract.

This action is only possible via the contract admin. It cannot be called when the gauge has no deposits. As a safety precaution, this call validates all the signatures with the following sequence of actions:

LP tokens are deposited into the new staking contract, verifying that the deposit signature is correct.

balanceOf is called on the LP token to confirm that the gauge’s token balance is now zero.

The LP tokens are withdrawn, verifying that the withdraw function signature is correct.

balanceOf is called on the LP token again, to confirm that the gauge has successfully withdrawn it’s entire balance.

A call to claim rewards is made to confirm that it does not revert.

These checks are required to protect against an incorrectly designed staking contract or incorrectly structured input arguments.

It is also possible to claim from a reward contract that does not require onward staking. In this case, use 00000000 for the function selectors for both staking and withdrawing.

```rewards_receiver(addr: address)→ address: view```

This function allows for the redirection of claimed rewards to alternative accounts. If an account has enabled a default rewards receiver this function will return that default account, otherwise it’ll return ZERO_ADDRESS.

```last_claim()→ uint256: view```

The epoch timestamp of the last call to claim from reward_contract.

```claim_rewards(_addr: address = msg.sender, _receiver: address = ZERO_ADDRESS): nonpayable```
Claim all available reward tokens for _addr. If no address is given, defaults to the caller. If the _receiver argument is provided rewards will be distributed to the address specified (caller must be _addr in this case). If the _receiver argument is not provided, rewards are sent to the default receiver for the account if one is set.

```claimed_reward(_addr: address, _token: address)→ uint256: view```

Get the number of already claimed reward tokens for a user.

```claimable_reward(_addr: address, _token: address)→ uint256: view```

Get the number of claimable reward tokens for a user

This call does not consider pending claimable amount in reward_contract. Off-chain callers should instead use claimable_reward_write as a view method.

```claimable_reward_write(_addr: address, _token: address)→ uint256: nonpayable```

Get the number of claimable reward tokens for a user. This function should be manually changed to “view” in the ABI. Calling it via a transaction will checkpoint a user’s rewards updating the value of claimable_reward. This function does not claim/distribute pending rewards for a user.


## Gauge Controller

### Querying Gauge and Type Weights

```gauge_types(gauge_addr: address)→ int128: view```

The gauge type for a given address, as an integer.

Reverts if gauge_addr is not a gauge.

```get_gauge_weight(gauge_addr: address)→ uint256: view```

The current gauge weight for gauge_addr.

```get_type_weight(type_id: int128)→ uint256: view```

The current type weight for type_id as an integer normalized to 1e18.

```get_total_weight()→ uint256: view```

The current total (type-weighted) weight for all gauges.

```get_weights_sum_per_type(type_id: int128)→ uint256: view```

The sum of all gauge weights for type_id.

### Vote-Weighting

Vote weight power is expressed as an integer in bps (units of 0.01%). 10000 is equivalent to a 100% vote weight.

```vote_user_power(user: address)→ uint256: view```

The total vote weight power allocated by user.

```last_user_vote(user: address, gauge: address)→ uint256: view```

Epoch time of the last vote by user for gauge. A gauge weight vote may only be modified once every 10 days.

```vote_user_slopes(user: address, gauge: address)```

Information about user’s current vote weight for gauge.

Returns the current slope, allocated voting power, and the veBAO locktime end.

### Vote for Gauge Weights

```vote_for_gauge_weights(_gauge_addr: address, _user_weight: uint256): nonpayable```

Allocate voting power for changing pool weights.

_gauge_addr Gauge which msg.sender votes for

_user_weight Weight for a gauge in bps (units of 0.01%). Minimal is 0.01%. Ignored if 0


### Adding New Gauges and Types

All of the following methods are only be callable by the DAO admin as the result of a successful vote.

```add_gauge(addr: address, gauge_type: int128): nonpayable```

Add a new gauge.

addr: Address of the new gauge being added

gauge_type: Gauge type

Once a gauge has been added it cannot be removed. New gauges should be very carefully verified prior to adding them to the gauge controller.

```gauge_relative_weight(addr: address, time: uint256 = block.timestamp)→ uint256: view```

Get the relative the weight of a gauge normalized to 1e18 (e.g. 1.0 == 1e18).

Inflation which will be received by this gauge is calculated as inflation_rate * relative_weight / 1e18. * addr: Gauge address * time: Epoch time to return a gauge weight for. If not given, defaults to the current block time.

```add_type(_name: String[64], weight: uint256 = 0): nonpayable```

Add a new gauge type.

_name: Name of gauge type

weight: Weight of gauge type

```change_type_weight(type_id: int128, weight: uint256)```

Change the weight for a given gauge type.

Only callable by the DAO ownership admin.

type_id Gauge type id

weight New Gauge weight

## Minter (Specification)

### Minting BAO

```mint(gauge_addr: address): nonpayable```

Mint allocated tokens for the caller based on a single gauge.

gauge_addr: LiquidityGauge address to get mintable amount from

```mint_many(gauge_addrs: address[8]): nonpayable```

Mint BAO for the caller from several gauges.

gauge_addr: A list of LiquidityGauge addresses to mint from. If you wish to mint from less than eight gauges, leave the remaining array entries as ZERO_ADDRESS.

```mint_for(gauge_addr: address, for: address): nonpayable```

Mint tokens for a different address.

In order to call this function, the caller must have been previously approved by for using toggle_approve_mint.

gauge_addr: LiquidityGauge address to get mintable amount from

for: address to mint for. The minted tokens are sent to this address, not the caller.

```toggle_approve_mint(minting_user: address): nonpayable```

Toggle approval for minting_user to mint BAO on behalf of the caller.

```allowed_to_mint_for(minter: address, for: address)→ bool: view```

Getter method to check if minter has been approved to call mint_for on behalf of for.