# Bitcoin anchored cross-chain governance

## Motivation
Currently there is no optimal and secure way to approach governance of fully decentralized (no admin keys and where every action needs to be decided by DAO vote) multi-chain DAOs (DAOs with their smart contract/treasury deployed on multiple chains). The use of tokens in the DAO governance proves challenging in the multi-chain world, as you can either have separate voting tokens on each chain (not ideal) or you need to have a way to bridge the tokens between chains (which, using existing bridges, relies on honest majority running the bridge), or at least bridge the calldata/instructions that were decided by DAO from one chain to another (which has the same drawbacks as token bridging itself).

## Bitcoin SPV on smart chains
Bitcoin blockchain, using SPV (simplified payment verification) is very easy and cheap to verify, even using smart contracts on different chains. This means that you can mirror current state of the bitcoin blockchain in a smart contract on other chains relatively cheaply, in turn creating a sort of bitcoin chain witness able to verify that any bitcoin transaction took place, such a solution relies purely on bitcoin's proof-of-work and requires just 1 out of N honest parties instead of honest majority. Even in an edge-case of there being no honest party, it is prohibitively expensive to spoof a bitcoin blockheader (as it still requires the same amount of proof-of-work as if it would be a valid blockheader).

## Bitcoin UTXO based tokens
We can allocate tokens directly to Bitcoin UTXOs, locking script on that UTXO then decides who owns and can then spend the tokens.

### Storing token amounts on smart chains
Amounts of governance tokens allocated to each Bitcoin's UTXO are stored on smart chains in the DAO contract, this has to be mirrored on every smart chain the DAO operates on.

### State transitions (token transfers)
State transition is done in 3 steps:
1. State transition data is created (transfer of tokens to different UTXOs) and a SHA256 commitment hash of this data is computed, a UTXO with tx id of 32 zeroes is used to assign the tokens to the newly generated UTXO of the bitcoin transaction used to commit the state transition.
2. Transaction on Bitcoin blockchain is created and sent, spending the UTXO holding the tokens and creating at least 1 OP_RETURN output with the SHA256 commitment hash of the data
3. Once the Bitcoin transaction is confirmed (has >6 confirmations), the state of the token allocation in DAO contracts on all the smart chains needs to be updated. This is done by supplying state transition data, transaction and merkle proof of the transaction to the DAO contracts, the DAO contract verifies that the transaction was sent and confirmed on bitcoin blockchain via Bitcoin SPV, checks if it contains valid commitment hash for the state transition data, verifies state transition validity and executes the state transition.

Token transfers require 1 Bitcoin transaction + 1 transaction for every smart chain the DAO operates on.

### Token locking for voting
To participate on DAO voting the tokens must first be locked, the tokens need to be locked till at least the end of the governance vote that the user wants to participate at - this is done to prevent people from sending tokens to other accounts and voting again with the very same tokens. The tokens can also be locked for longer than required periods to amplify their voting power, attaining the highest possible voting power when locked for 4 years.

Locking is done by sending the tokens to a bitcoin UTXO with a locking script containing OP\_CLTV (check locktime-verify) opcode, making the UTXO unspendable before some time in the future. A state transition is created, sending the tokens to the UTXO generated by commitment transaction (this is done so smart contracts can verify the UTXO locking script along with state transition simultaneusly), and also specifying the secp256k1 public key that will hold the voting rights on the smart chains.

### Initial token sale
DAO needs to select one chain - genesis smart chain, where the initial token distribution is created, this can include initial token sale, allocations for founding team, advisors and early investors. Once the distribution is set, the tokens are assigned to the bitcoin UTXOs specified by the token holders.

### Expansion to other chains
To expand to other chains, a snapshot of current token distribution is taken, merkle root of all the token holders is created and a DAO smart contract on other chain is created with this merkle root. Any DAO participant can independently verify that this merkle root was correctly calculated based on the allocations from genesis smart chain at the time of expansion to the other chain.

### Token trading
The tokens themsleves cannot be traded on the smart chains, as they are anchored in bitcoin blockchain. They can only be traded on bitcoin directly. Trading can be done either via atomic swaps, but those requires multiple transactions and are relatively slow, or via scriptless atomic swaps presented by Federico Tenga - [here](https://github.com/orgs/LNP-BP/discussions/125#discussioncomment-5728914). However scriptless atomic swaps do have their own set of drawbacks, which need to be addressed (the free option problem).
