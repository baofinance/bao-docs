# Bao Finance Developer Docs

## Implementation

Bao Finance aims to be an open source toolkit that uses existing financial primitives (and perhaps new ones in near future) to build an ecosystem of composable monetary building blocks backed by secure oracles that users can run generalized autonomous financial strategies customized to their own risk tolerance on.

## Contracts

- [BAO Token (BAO) & Voting Escrow BAO (veBAO)](./src/01-bao-token.md)
- [BAO Distribution](./src/02-bao-distr.md)
- [Liquidity gauges (token minting)](./src/03-liq-gauges.md)
- [Gauge Voting](./src/04-vote-onchain.md)
- [Bao Baskets](./src/06-Baskets.md)
- [Bao Markets](./src/07-Markets.md)
- [Fee Distribution](./src/08-fee-distr.md)
- [Contribute](./src/09-contribute.md)

## Helpful external documents

- [Curve Finance detailed docs](https://curve.readthedocs.io/toctree.html) breaks down a lot of the same contracts Bao uses for its governance system (voting escrow, gauges, etc.)
- [Curve Finance user docs](https://resources.curve.fi/) describes some of the processes that Bao shares in common where the code is forked (voting escrow, gauges, etc.))
- [Bao generalized docs](https://info.bao.finance/docs/), Comprehensive overview of how to use Bao that is less development focused

## mdbook

- clone the repo
- install [mdbook](https://rust-lang.github.io/mdBook/guide/installation.html)
- run [mdbook serve --open](https://rust-lang.github.io/mdBook/guide/creating.html) in your command line
