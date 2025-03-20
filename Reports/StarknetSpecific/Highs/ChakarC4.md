# Chakara Contest on C4
## [H-5] `settlement.cairo` doesn't process callback correctly leading to `CrossChainMsgStatus` marked as SUCCESS even if it failed on destination chain
### TL;DR
This issue could've been prevented if the team would have implemented the same logic, as the Solidity contract's counterpart. Essentially, the function checks for a `SUCCESS` or `FAILED` cross-chain transaction. The Cairo contract logic would've set the answer to `SUCCESS` when it shouldn't have been. 

**What to look for in the codebase?**<br>
If possible, look at the Solidity counterpart. Usually isn't the case though. In this situation, you would need to, surprise surprise, have a deep understanding of the system. The white hatters have identified that the team have used the wrong logic for something that is different to what they set out to do. 

Sometimes it's not just hacking the contract, but it's realizing that the team were confused and the system will not work as intended!

### Vulnerability Description
When a cross-chain message is sent it can return a callback with status `FAILED` or `SUCCESS`. The problem is that even if the cross-chain message failed, the original message status on the source chain would be marked as `SUCCESS`.

### Code Sample
- Cairo contract:
```rust
    fn receive_cross_chain_callback(
                ref self: ContractState,
                cross_chain_msg_id: felt252,
                from_chain: felt252,
                to_chain: felt252,
                from_handler: u256,
                to_handler: ContractAddress,
                cross_chain_msg_status: u8, <--
                sign_type: u8,
                signatures: Array<(felt252, felt252, bool)>,
            ) -> bool {
    //other functionality

    let success = handler.receive_cross_chain_callback(cross_chain_msg_id, from_chain, to_chain, from_handler, to_handler , cross_chain_msg_status);

                let mut state = CrossChainMsgStatus::PENDING;
                if success{
                    state = CrossChainMsgStatus::SUCCESS;
                }else{
                    state = CrossChainMsgStatus::FAILED;
                }

                self.created_tx.write(cross_chain_msg_id, CreatedTx{
                    tx_id:cross_chain_msg_id,
                    tx_status:state, <--- update the status 
                    from_chain: to_chain,
                    to_chain: from_chain,
                    from_handler: to_handler,
                    to_handler: from_handler
                });
```

- Solidity contract:
```solidity
        function processCrossChainCallback(
            uint256 txid,
            string memory from_chain,
            uint256 from_handler,
            address to_handler,
            CrossChainMsgStatus status,
            uint8 sign_type,
            bytes calldata signatures
        ) internal {
            require(
                create_cross_txs[txid].status == CrossChainMsgStatus.Pending,
                "Invalid transaction status"
            );

            if (
                ISettlementHandler(to_handler).receive_cross_chain_callback( 
                    txid,
                    from_chain,
                    from_handler,
                    status,
                    sign_type,
                    signatures
                )
            ) {
                create_cross_txs[txid].status = status; <---
            
            } else {
                create_cross_txs[txid].status = CrossChainMsgStatus.Failed;
            }
        }
```
### Impact
The problem is that as long as the call to `handler.receive_cross_chain_callback` function was successful, the message as a whole will be marked in a `SUCCESS` state even though that `cross_chain_msg_status` could be SUCCESS or FAILED depending on if the message failed on the destination chain.

This could lead to a situation where a message fails to get processed on the destination chain, a callback is returned with `cross_chain_msg_status == FAILED` but the message is marked as `SUCCESS`.

That situation could be a user trying to bridge his tokens, the bridging fails so he doesn't receive his tokens on the destination chain, a callback is made and the message is marked as a `SUCCESS` even though it as not successfully executed.

### Recommendation
Change this line from this:
```rust
if success{
    state = CrossChainMsgStatus::SUCCESS
}
```

To this:
```rust
if success{
    state = cross_chain_msg_status;
}
```

### References
- Read on [Solodit](#https://solodit.cyfrin.io/issues/h-05-settlementcairo-doesnt-process-callback-correctly-leading-to-crosschainmsgstatus-marked-as-success-even-if-it-failed-on-destination-chain-code4rena-chakra-chakra-git)