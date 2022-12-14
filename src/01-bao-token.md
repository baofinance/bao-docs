# BAO Token & Vote Escrowed BAO (veBAO)

The BAO token is the instrument through which governance power and revenue sharing occurs for Bao's synthetics markets. Users must lock their BAO tokens into the voting escrow contract in exchange for a non-transferrable veBAO balance, in order to participate in governance. This applies to both gauge weight voting and snapshot votes.

The BAO token is inflationary with a linear piecewise minting function and mints new tokens as emissions to the LP tokens incentivized through BAO liquidity Gauges that have staked their LP inside a gauge contract (more information on this in ["BAO liquidity gauges & emissions"](./03-liq-gauges.md)).

## notes

- Source: [here](https://etherscan.io/address/0xce391315b414d4c7555956120461d21808a69f3a#code)
- The BAO token adheres to the ERC20 standard and has some additional functiontality for the minting permissions/access.
- additional functions include:

### reading

```
@external
@view
def available_supply() -> uint256:
    """
    @notice Current number of tokens in existence (claimed or unclaimed)
    """
    return self._available_supply()
```
Returns "self.start_epoch_supply + (block.timestamp - self.start_epoch_time) * self.rate" which is the position based on the current timestamp relative to when the token contract was deployed. This number will always be larger than the total supply as new tokens are only created when permitted to emit them from the available supply to mint. This balance represents what could have been minted given only the minting function y axis point, totalSupply() reflects the actual in minted supply in existence.

@external
@view
def mintable_in_timeframe(start: uint256, end: uint256) -> uint256:
    """
    @notice How much supply is mintable from start timestamp till end timestamp
    @param start Start of the time interval (timestamp)
    @param end End of the time interval (timestamp)
    @return Tokens mintable from "start" till "end"

Returns token balance mintable in a given range defined by timestamps.

### writing

```
@external
def update_mining_parameters():
    """
    @notice Update mining rate and supply at the start of the epoch
    @dev Callable by any address, but only once per epoch
         Total supply becomes slightly larger if this function is called late
    """
    assert block.timestamp >= self.start_epoch_time + RATE_REDUCTION_TIME  # dev: too soon!
    self._update_mining_parameters()
```
This function writes to the token contract and changes the mining parameters assuming the timestamp assertions do not revert the call (once a year). The mining parameters, namely the mining rate, but also the mining epoch are changed to start the next epoch and reduce the rate over time using RATE_REDUCTION_COEFFICIENT (storage variable).


## Vote Escrowed BAO

veBAO is a non-transferable ERC20 token that can only be received by locking BAO tokens up into the Voting Escrow contract for a maximum of 4 years. 1000 BAO tokens locked for 4 years is equal to 1000 veBAO token and the tokens that are locked decay in their governance weighting over time as the lock expiry date approaches closer. For example, if an address locks 1000 BAO for 4 years, at the start timestamp of the lock, they will have ~1000 veBAO tokens; after 2 years have passed into the 4 year lock, assuming their were no extensions made to the lock by the token holder, there will be roughly half the veBAO tokens the address started with as the veBAO locking function decays the user's veBAO balance as the lock becomes more mature/gets closer to expiration.

Users have the ability to extend their veBAO locks if they want to maintain a certain level of governance power instead of letting it decay.

Each address can only have 1 lock at a time.

### Fees

Holders of veBAO (AKA, those who lock up their BAO in the voitng escrow contract) will be able to claim fees in baoUSD on a weekly basis in proportion to how much veBAO balance they have. Those with the most veBAO balance will be able to claim the most amount of fees. Fees are generated by protocol revenue and given to veBAO holders.

The BAO Matrix:
### Bao token Matrix here

## notes

- Source: [here](https://etherscan.io/address/0x8bf70dfe40f07a5ab715f7e888478d9d3680a2b6#code)
- The vote escrow token is an ERC20 token contract but does not have transfer capabilities, hence why the tokens are locked when you create a veBAO lock

### reading

In this case reading the veBAO balance is handled by 2 functions from the contract.
```
@external
@view
def balanceOf(addr: address, _t: uint256 = block.timestamp) -> uint256:
    """
    @notice Get the current voting power for `msg.sender`
    @dev Adheres to the ERC20 `balanceOf` interface for Aragon compatibility
    @param addr User wallet address
    @param _t Epoch time to return voting power at
    @return User voting power
    """
```
balanceOf() which returns address "_t" veBAO balance

```
@external
@view
def balanceOfAt(addr: address, _block: uint256) -> uint256:
    """
    @notice Measure voting power of `addr` at block height `_block`
    @dev Adheres to MiniMe `balanceOfAt` interface: https://github.com/Giveth/minime
    @param addr User's wallet address
    @param _block Block to calculate the voting power at
    @return Voting power
    """
```
balanceOfAt() returns what the balance will be for "addr" at a given block number

```
@external
@view
def totalSupply(t: uint256 = block.timestamp) -> uint256:
    """
    @notice Calculate total voting power
    @dev Adheres to the ERC20 `totalSupply` interface for Aragon compatibility
    @return Total voting power
```
totalSupply() returns the total number of veBAO tokens

```
locked: public(HashMap[address, LockedBalance])
```
locked() returns the lock balance for a given address

### writing

```
@external
@nonreentrant('lock')
def create_lock(_value: uint256, _unlock_time: uint256):
    """
    @notice Deposit `_value` tokens for `msg.sender` and lock until `_unlock_time`
    @param _value Amount to deposit
    @param _unlock_time Epoch time when tokens unlock, rounded down to whole weeks
    """
```
create a veBAO lock given the caller's BAO token balance to be locked and length of the lock [current timestamp + (1 week into the future min, or 4 years max)]

```
@external
@nonreentrant('lock')
def create_lock_for(_to: address, _value: uint256, _unlock_time: uint256):
    """
    @notice Deposit `_value` tokens for `_to` address and lock until `_unlock_time`
    @param _value Amount to deposit
    @param _unlock_time Epoch time when tokens unlock, rounded down to whole weeks
    """
```
create_lock_for() is only callable by the distribution contract to allow people with locked BAOv1 to migrate directly into the voting escrow contract with a veBAO lock

```
@external
@nonreentrant('lock')
def increase_amount(_value: uint256):
    """
    @notice Deposit `_value` additional tokens for `msg.sender`
            without modifying the unlock time
    @param _value Amount of tokens to deposit and add to the lock
    """
```
increase the caller's veBAO lock amount using BAO tokens in the balance of the caller

```
@external
@nonreentrant('lock')
def increase_unlock_time(_unlock_time: uint256):
    """
    @notice Extend the unlock time for `msg.sender` to `_unlock_time`
    @param _unlock_time New epoch time for unlocking
    """
```
extend the time of the caller's lock if it is less than the maximum of 4 years until expiration

@external
@nonreentrant('lock')
def withdraw():
    """
    @notice Withdraw all tokens for `msg.sender`
    @dev Only possible if the lock has expired
    """
withdraw BAO tokens from the vote escrow contract, only when the lock has past the expiry timestamp