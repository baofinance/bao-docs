# Bao Distribution

The Bao Distribution contract is used to migrate those with locked BAOv1 from the yield farming phase to the new BAOv2 token that uses a vote escrow governance system.

## notes

- Source [here](https://etherscan.io/address/0x885d90a424f87d362c9369c0f3d9a2d28af495f4#code)

### reading 

```
/**
 * Get how many tokens an account is able to claim at a given timestamp. 0 = now.
 * This function takes into account the date of the account's last claim, and returns the amount
 * of tokens they've accrued since.
 *
 * @param _account Account address to query.
 * @param _timestamp Timestamp to query.
 * @return c _account's claimable tokens, scaled by 1e18.
 */
function claimable(address _account, uint64 _timestamp) public view returns (uint256 c)
```
claimable() returns the claimable balance during an active distribution, for "_account"

```
/**
 * Get the amount of tokens that would have been accrued along the distribution curve, assuming _daysSinceStart
 * days have passed and the account has never claimed.
 *
 * f(x) = 0 <= x <= 1095 : (2x/219)^2
 *
 * @param _amountOwedTotal Total amount of tokens owed, scaled by 1e18.
 * @param _daysSinceStart Time since the start of the distribution, scaled by 1e18.
 * @return _amount Amount of tokens accrued on the distribution curve, assuming the time passed is _daysSinceStart.
 */
function distCurve(uint256 _amountOwedTotal, uint256 _daysSinceStart) public pure returns (uint256 _amount)
```
returns the amount of tokens within the given distribution curve voted f(x) = 0 <= x <= 1095 : (2x/219)^2 at a given timestamp

```
mapping(address => bool) public lockStatus;
```
returns a boolean true or false based on if the address has an active veBAO lock

### writing

- startDistribution(), starts the distribution of BAOv2 tokens for a given locked BAOv1 holder (caller) based on a merkle root containing a snapshot of balances taken just before deployment 
- claim(), claims the claimable token amount for the caller assuming they have started their distribution
- lockDistribution(), locks the remaining tokens currently claimable by the caller as well as the remaining total balance that would have been streamed over 3 years to that caller all into a veBAO lock between 3 year minimum and 4 year maximum
- endDistribution(), takes the remaining balance left to be distributes it and gives the caller their claimable balance as normal to their wallet, then takes the reamining balance to be distributed over a 3 year time frame and gives it to the user immediatley at a slash of ~95% (only on tokens that are not claimable)

```
/**
 * Starts the distribution of BAO for msg.sender.
 *
 * @param _proof Merkle proof to verify msg.sender's inclusion and claimed amount.
 * @param _amount Amount of tokens msg.sender is owed. Used to generate the merkle tree leaf.
 */
function startDistribution(bytes32[] memory _proof, uint256 _amount) external
```

```
/**
     * Claim all tokens that have been accrued since msg.sender's last claim.
     */
    function claim() external nonReentrant
```

```
/**
 * Lock all tokens that have NOT been claimed since msg.sender's last claim
 *
 * The Lock into veBAO will be set at _time with this function in-line with length of distribution curve (minimum of 3 years)
 */
function lockDistribution(uint256 _time) external nonReentrant
```

```
/**
 * Claim all tokens that have been accrued since msg.sender's last claim AND
 * the rest of the total locked amount owed immediately at a pre-defined slashed rate.
 *
 * Slash Rate:
 * days_since_start <= 365: (100 - .01369863013 * days_since_start)%
 * days_since_start > 365: 95%
 */
function endDistribution() external nonReentrant
```