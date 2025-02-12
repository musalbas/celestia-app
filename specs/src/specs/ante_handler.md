# AnteHandler

Celestia makes use of a Cosmos SDK [AnteHandler](https://docs.cosmos.network/v0.46/modules/auth/03_antehandlers.html#antehandlers) in order to reject decodable sdk.Txs that do not meet certain criteria. The AnteHandler is defined in [app/ante/ante.go](https://github.com/celestiaorg/celestia-app/blob/7f97788a64af7fe0fce00959753d6dd81663e98f/app/ante/ante.go) and is invoked at multiple times during the transaction lifecycle:

1. `CheckTx` prior to the transaction entering the mempool
1. `PrepareProposal` when the block proposer includes the transaction in a block proposal
1. `ProcessProposal` when validators validate the transaction in a block proposal
1. `DeliverTx` when full nodes execute the transaction in a decided block

The AnteHandler chains together several decorators to ensure the following criteria are met:

- The tx does not contain any [extension options](https://github.com/cosmos/cosmos-sdk/blob/22c28366466e64ebf0df1ce5bec8b1130523552c/proto/cosmos/tx/v1beta1/tx.proto#L119-L122).
- The tx passes `ValidateBasic()`.
- The tx's [timeout_height](https://github.com/cosmos/cosmos-sdk/blob/22c28366466e64ebf0df1ce5bec8b1130523552c/proto/cosmos/tx/v1beta1/tx.proto#L115-L117) has not been reached if one is specified.
- The tx's [memo](https://github.com/cosmos/cosmos-sdk/blob/22c28366466e64ebf0df1ce5bec8b1130523552c/proto/cosmos/tx/v1beta1/tx.proto#L110-L113) is <= the max memo characters where [`MaxMemoCharacters = 256`](<https://github.com/cosmos/cosmos-sdk/blob/a429238fc267da88a8548bfebe0ba7fb28b82a13/x/auth/README.md?plain=1#L230>).
- The tx's [gas_limit](https://github.com/cosmos/cosmos-sdk/blob/22c28366466e64ebf0df1ce5bec8b1130523552c/proto/cosmos/tx/v1beta1/tx.proto#L211-L213) is > the gas consumed based on the tx's size where [`TxSizeCostPerByte = 10`](https://github.com/cosmos/cosmos-sdk/blob/a429238fc267da88a8548bfebe0ba7fb28b82a13/x/auth/README.md?plain=1#L232).
- The tx's feepayer has enough funds to pay fees for the tx. The tx's feepayer is the feegranter (if specified) or the tx's first signer. Note the [feegrant](https://docs.cosmos.network/v0.46/) module is enabled.
- The tx's count of signatures <= the max number of signatures. The max number of signatures is [`TxSigLimit = 7`](https://github.com/cosmos/cosmos-sdk/blob/a429238fc267da88a8548bfebe0ba7fb28b82a13/x/auth/README.md?plain=1#L231).
- The tx's [gas_limit](https://github.com/cosmos/cosmos-sdk/blob/22c28366466e64ebf0df1ce5bec8b1130523552c/proto/cosmos/tx/v1beta1/tx.proto#L211-L213) is > the gas consumed based on the tx's signatures.
- The tx's [signatures](https://github.com/cosmos/cosmos-sdk/blob/22c28366466e64ebf0df1ce5bec8b1130523552c/types/tx/signing/signature.go#L10-L26) are valid. For each signature, ensure that the signature's sequence number (a.k.a nonce) matches the account sequence number of the signer.
- The tx's [gas_limit](https://github.com/cosmos/cosmos-sdk/blob/22c28366466e64ebf0df1ce5bec8b1130523552c/proto/cosmos/tx/v1beta1/tx.proto#L211-L213) is > the gas consumed based on the blob size(s). Since blobs are charged based on the number of shares they occupy, the gas consumed is calculated as follows: `gasToConsume = sharesNeeded(blob) * bytesPerShare * gasPerBlobByte`. Where `bytesPerShare` is a a global constant (an alias for [`ShareSize = 512`](https://github.com/celestiaorg/celestia-app/blob/c90e61d5a2d0c0bd0e123df4ab416f6f0d141b7f/pkg/appconsts/global_consts.go#L27-L28)) and `gasPerBlobByte` is a governance parameter that can be modified (the [`DefaultGasPerBlobByte = 8`](https://github.com/celestiaorg/celestia-app/blob/c90e61d5a2d0c0bd0e123df4ab416f6f0d141b7f/pkg/appconsts/initial_consts.go#L16-L18)).
- The tx's total blob size is <= the max blob size. The max blob size is derived from the maximum valid square size. The max valid square size is the minimum of: `GovMaxSquareSize` and `SquareSizeUpperBound`.
- The tx does not contain a message of type [MsgSubmitProposal](https://github.com/cosmos/cosmos-sdk/blob/d6d929843bbd331b885467475bcb3050788e30ca/proto/cosmos/gov/v1/tx.proto#L33-L43) with zero proposal messages.
- The tx is not an IBC packet or update message that has already been processed.

In addition to the above criteria, the AnteHandler also has a number of side-effects:

- Tx fees are deducted from the tx's feepayer and added to the fee collector module account.
- Tx priority is calculated based on the the smallest denomination of gas price in the tx and set in context.
- The nonce of all tx signers is incremented by 1.
