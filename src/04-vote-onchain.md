# on-chain voting with veBAO

using a given address that has veBAO balance/voting power in an active lock one can call vote_for_gauge_weights() in the [GaugeController](https://etherscan.io/address/0x840e75261c2934f33c8766f6ea6235330dc1d72d#code)

## notes

The function takes the address of the gauge msg.sender wants to direct voting toward "gauge_addr" given msg.sender's total veBAO balance

The "_user_weight" parameter tells the gauge controller how much of msg.sender's veBAO balance should be used to vote for an increase in weight to a given gauge_addr

so for example, one could take 1,000,000 veBAO from one address and split up the 1,000,000 weight amongst various different gauges using multiple calls to this function or simply use all 1 million veBAO and direct it towards a singular pool with 1 call.

### vote through the UI if you would like :)
