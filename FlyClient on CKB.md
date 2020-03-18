# FlyClient on CKB

This document presents ideas to support a super-light client for CKB, allowing for sub-linear verification of CKB's proof of work (compared to verifying all block headers) without any consensus changes. The design makes use of an on-chain contract to implement FlyClient [[1]](https://eprint.iacr.org/2019/226.pdf) and produces proofs of CKB transaction inclusion in about 2 megabytes.

This document also includes a description of light clients, SPV proofs, an alternative super-light client approach (NIPoPoWs [[2]](https://eprint.iacr.org/2017/963.pdf)), proof size estimates and an idea for proof size reduction, as well as a contract-based MMR implementation.

The key advantage of this design is that it requires no changes to existing mining software or consensus rules, thus no coordination or polarization/delay due to a hard fork. It also allows CKB to continue following conventional Nakamoto consensus rules (commitment to prior block). 

Though this design relies on the processing of transactions to keep a contract's data updated, the shared benefit of a super-light client should provide the necessary incentives to ensure stakeholders keep this data updated.



## 1. Light Clients
Light client proofs can provide sufficient evidence of a transaction's inclusion in a blockchain without requiring operation of a full node that downloads and verifies the entirety of the blockchain's history. Light clients make the following security assumptions:

1. The combined hash power of all malicious miners is less than that of honest miners (honest majority assumption);
2. At least one full node the light client is connected to is honest.

### 1.1 SPV Light Clients in Bitcoin

The first light client implementation, the SPV proof, was described by Satoshi in the Bitcoin white paper [[3]](https://bitcoin.org/bitcoin.pdf). SPV proofs are generated through the following process:

1. Light client knows the genesis block hash;
2. Light client connects to a number of (untrusted) full nodes;
4. Full nodes provide all block headers following the genesis block;
4. Full node provides a Merkle path to prove the transaction is included in a valid block;
5. Light client verifies that each block header hash value meets the target difficulty.

Through this process, provers (full nodes) can provide verifiers (light clients) sufficient evidence to prove the following without requiring trust in a 3rd party:

1. Which chain tip represents the longest (heaviest/most difficult);
2. A transaction is included in a block in the chain. 

Bitcoin SPV proofs yield about a 1000x improvement in data transmission and storage requirements compared to downloading all block data. These proofs have worked well, trustless mobile wallets implement them and they are still in use today.

### 1.2 SPV proofs and later blockchains

SPV proofs for newer blockchains such as Ethereum or CKB grow orders of magnitude larger than those for Bitcoin, diminishing their convenience and utility due to the following:

1. Blocks are produced at much higher average rates (<10 seconds on CKB, 15 seconds on Ethereum, compared with 600 seconds in Bitcoin);
2. Larger block headers due to inclusion of additional fields (212 bytes in CKB, 508 bytes in Ethereum, compared with 80 bytes in Bitcoin). 



## 2. Going beyond SPV proofs

A properly designed light client protocol will prevent adversaries from including invalid (fake) blocks that could be used to deceive a light client into following an alternate chain of the same length (or difficulty) as that of the honest chain. 

While SPV proofs accomplish this by requiring verifiers to check all headers, foundational research has created "super-light client" designs, which allow for verification of a blockchain's proof of work using only a fraction of the chain's block headers. 

Super-light clients allows for sub-linear verification of the work expended to produce a blockchain and significantly compress light client proof sizes, without requiring any trust in a 3rd party.

### 2.1 Super-light client design: NIPoPoWs

NIPoPoWs [[2]](https://eprint.iacr.org/2017/963.pdf) or "Non-Interactive Proofs of Proof of Work" were introduced by Kiayias, Miller and Zindros in a 2018 paper that built on PoPoWs [[4]](https://fc16.ifca.ai/bitcoin/papers/KLS16.pdf) and an idea that originated in early Bitcoin Talk forum discussions [[5]](https://bitcointalk.org/index.php?topic=98986.0;all). The NIPoPoW construction allows for proofs that grow logarithmically with blockchain length and can be provided through a single message between prover and verifier.

**2.1.1 NIPoPoW operation**

In proof of work protocols, adding a new block to the blockchain requires a block header hash value that is less than a target value (target difficulty). Because the value of the block header hash is random: if a block header hash value is less the target difficulty, the probability that the block header hash value is also *n* bits smaller than the target difficulty is 1 / 2^*n*

Blocks that exceed target difficulty are referred to as "super-blocks" and rare super-blocks can be provided to a verifier to succinctly prove that a certain amount of work has been expended to produce the current chain tip. This property can then be used by super-light clients to identify the longest (heaviest/most difficult) chain without verifying all block headers.

**2.1.2 Construction a superchain**

The NIPoPoW protocol creates an authenticated data structure referred to as a "superchain" that is used by light clients to identify the longest (heaviest/most difficult) chain.

This structure contains interlinks of blocks exceeding the difficulty target, ordered on different levels according to how many bits they exceed the difficulty target by. All blocks exceeding target difficulty by 1 bit would be linked together on level 1, 2 bit on level 2, 3-bit on level 3 and so on (demonstrated below). A hash of the current value of the entire structure would be included in the block header of each block. 

![super chain](.\superchain.png)

**2.1.3 Attacking superchain quality**

The quality of the superchain can be attacked through selfish-mining [[6]](https://arxiv.org/abs/1311.0243). Whenever an honest participant has recently mined a rare superblock, an adversary can attempt to propagate a competing block faster, orphaning the superblock and in turn not contributing to the quality of the superchain. 

Miners can also be bribed to discard superblocks (thus making a competing superchain easier to construct). The NIPoPoW protocol prescribes that when superchain quality is attacked, provers should fall back to SPV proofs, which unfortunately limits the succinctness that super-light client proofs are designed to provide.

### 2.2 Super-light client design: FlyClient

FlyClient [[1]](https://eprint.iacr.org/2019/226.pdf) was introduced in a 2019 paper by Buuunz, Kiffer, Luu, and Zamani. FlyClient authors observe that in previous super-light client constructions, verifiers only download a small number of total blocks which are not necessarily chained together. This design allows a malicious prover to deceive a verifier by presenting a mix of invalid blocks and honest blocks.

FlyClient closes this attack vector by requiring that miners commit to the entire accumulated blockchain history at each block. With this commitment, a prover can only provide valid blocks at expected positions in the chain. A sampling of the blocks in this commitment are provided to super-light clients for verification.

**2.2.1 FlyClient operation**

FlyClient captures these block history commitments in an append-only data structure known as a Merkle Mountain Range or MMR [[7](https://github.com/opentimestamps/opentimestamps-server/blob/master/doc/merkle-mountain-range.md), [8](https://github.com/mimblewimble/grin/blob/master/doc/mmr.md#structure), [9](https://github.com/zmitton/merkle-mountain-range)] :

> *"At every block height i, the prover appends the hash of the previous block, Bi−1, to the most recent MMR and records the new MMR root, Mi, in the header of Bi. As a result, each MMR root stored at every block height can be seen as a commitment to the entire blockchain"*

FlyClient also prescribes that accumulated difficulty and accumulated time are included in each commitment to the MMR, meaning that each block header commits to both the sequence of all previous blocks as well as total accumulated difficulty and time values. 

By committing to difficulty and time data, difficulty transitions can be proven to have been executed correctly and verifiers can ensure that no difficulty raising attacks [[10]](https://arxiv.org/abs/1312.7013) have been executed. 

**2.2.2 FlyClient proving process**

Each prover begins by sending the header of the last block in its chain, which includes the latest MMR commitment (*Mi*). The veriﬁer then samples a number of random blocks (logarithmic to chain length) from the prover based on a sampling distribution. 

1. Randomness is derived from a hash of the current block header:

> *"To make our probabilistic sampling protocol noninteractive, we apply the Fiat-Shamir heuristic to the protocol described so far. The randomness is generated from the hash of the head of the chain. The veriﬁer now simply checks that the proof is correct and that the randomness was derived correctly."*

2. Sampling distribution is based on accumulated difficulty:

> *"we use the same sampling distribution g(x) but x now denotes the relative aggregate diﬃculty. For example, x = 1/2 refers to a point on the chain where half of the diﬃculty has been amassed, and g(1/2) is the probability that the block at that point is sampled by FlyClient."* 

For every sampled block, the prover provides the corresponding block header and its Merkle path proving inclusion in the MMR root. This proves the block is located at the correct height in the chain as committed to in the MMR root. Additionally, the verifier checks that the MMR root stored in each sampled block header commits to valid subtree in the *current* MMR commitment (*Mi*).

If the proof-of-work solution or inclusion proofs for any blocks are invalid, the verifier rejects the proof. Otherwise, the provided block header is accepted as the latest block of the honest chain. Transaction inclusion is proven in the same process as an SPV proof, with a Merkle path to a transaction root and then a proof of that block's inclusion in the current MMR commitment (*Mi*). 

**2.2.3 Replication of FlyClient proofs**

Block sampling in FlyClient can be based on randomness derived from the hash of the latest block header making FlyClient is non-interactive. Full nodes can calculate a single proof (at a particular block height) and share it with many light clients. These light clients can in turn share the proof with other light clients, all of which can independently verify the integrity of the proof they receive.

**2.2.4 Difficulty MMR**

A diﬃculty MMR is a variant of the MMR with identical properties, but such that each node contains additional data. MMR commitments are modified such that each node in the Merkle tree additionally contains the aggregate diﬃculty of all nodes below it. Every node can be written as (h,w,t,Dstart,Dnext,n,data)

- h is a λ bit output of a hash function, 
- w is an integer representing a total diﬃculty,
- t is a timestamp, 
- D is a diﬃculty target, 
- n is an integer representing the size of the subtree rooted by the node,
- data is an arbitrary string. 

> *Deﬁnition 7 (Diﬃculty MMR). A diﬃculty MMR is a variant of the MMR with identical properties, but such that each node contains additional data. Every node can be written as h,w,t,Dstart,Dnext,n,data, where h is a λ bit output of a hash function, w is an integer representing a total diﬃculty, t is a timestamp, D is a diﬃculty target, n is an integer representing the size of the subtree rooted by the node and data is an arbitrary string. In a block, i.e., a leaf node, n = 1 and t is the time it took to mine the current block (the blocks time stamp minus the previous block’s timestamp). w,Dstart is the current diﬃculty targets and Dnext is the diﬃculty of the next block computed using the diﬃculty adjustment function deﬁned Deﬁnition 1. Each non-leaf node is deﬁned as {H(lc,rc),lc.w+rc.w,lc.t+rc.t,lc.Dstart,rc.Dnext,lc.n+rc.n,⊥}, wherelc = LeftChild and rc = RightChild.*



## 3. Approaching a Super-light Client for CKB
### 3.1 Choosing FlyClient over NIPoPoW

As the base of CKB's super-light client, FlyClient is a straightforward choice over NIPoPoW for three reasons:

1. While NIPoPoW does not explicitly chain all blocks together as part of the mining protocol, FlyClient creates a commitment to the blockchain's history at each block.

2. When relying on NIPoPoW proofs for security, an incentive is introduced for attackers to bribe miners into discarding blocks that if confirmed would contribute to the quality of NIPoPoW proofs (lowering attack costs). 

   This block discarding not only lowers the effectiveness and security of NIPoPoW proofs, it also lowers overall chain security (difficulty adjustments will be based on less than the chain's actual hash rate).

3. Construction of NIPoPoW proofs for blockchains with variable difficulty is still a topic for future research. 

### 3.2 FlyClient Implementation Paths (General)

**3.2.1 Block Header Field**

The MMR required for FlyClient has been implemented in the block header of Beam [[11]](https://github.com/BeamMW/beam/blob/42ef470c322a230cfa45f7f268bf37987da92057/core/fly_client.cpp), Grin [[12]](https://github.com/mimblewimble/grin/issues/1555) and is proposed for ZCash [[13]](https://github.com/zcash/zips/pull/220) and Ethereum Classic [[14]](https://ecips.ethereumclassic.org/ECIPs/ecip-1055). Mining on a MMR commitment versus a prior block commitment does not seem to have produced any notable complications (outside of a small amount of additional complexity to handle short forks [[15]](https://github.com/mimblewimble/grin/issues/1555#issuecomment-425543073)).

**3.2.2 State Commitment - Soft Fork **

The FlyClient paper describes implementation through a soft fork, in which only blocks that contain a transaction committing to the MMR would be accepted.

**3.2.3 State Commitment - Velvet Fork **

In the FlyClient paper, implementation through a "velvet fork" is described for uncontentious deployment in existing blockchains. Similar to merge-mining, compliant miners would include FlyClient's MMR data in the coinbase input and blocks that do not include this data would still be accepted. 

FlyClient would treat blocks that do not include commitment data as part of the last committed block.

**3.2.4 State Commitment - Transactions **

A second non-contentious path is a commitment to the MMR value in contract state through transactions. This has been mentioned as a permissionless implementation of FlyClient for Ethereum Classic [[16]](https://youtu.be/zcpSHeVgoos?t=2512). 

In Ethereum, the latest 256 block headers are accessible from a contract, the MMR root required by FlyClient could be built by submitting a contract transaction that reads previous block headers and adds them to the MMR. This implementation method does however require ongoing, timely processing of transactions and significantly expands proof size (due to Ethereum's data model). 

### 3.3 FlyClient Implementation Paths (CKB)

**3.3.1 Block Header**

Implementing the MMR required by FlyClient in the CKB block header would require changes to consensus rules and a hard fork. 

With any hard fork, stakeholder coordination and forced upgrades are required. Though Ethereum and other chains have normalized hard forks (through aggressive development or the fight for ASIC-resistance), they do introduce an element of centralization into blockchain governance. 

Decisions that affect all stakeholders reinforce a role of certain orchestrators of a blockchain's development, an idea that is in conflict with the goal of neutral infrastructure sustained through shared ownership of a community.

**3.3.2 State Commitment - Soft Fork**

This path carries much of the coordination, upgrade and governance implications of the block header implementation, however it does allow any CKB client version to sync from genesis and verify the blockchain's history. 

**3.3.3 State Commitment - Velvet Fork**

CKB does not include a coinbase input, though this could be implemented through optional transactional commitments.

**3.3.4 State Commitment - Contract Transaction**

CKB's `header deps` functionality allows a CKB script to read data from past block headers. This capability allows for trustless, user-sourced creation of the data set needed to create FlyClient proofs and is the approach described in the next section.

## 3.4 FlyClient on CKB (Transaction-based)

**3.4.1 FlyClient CKB info cell**

The implementation of FlyClient on CKB begins with an "anyone can unlock" cell that allows for a user to add to the MMR. 

The cell contains the following fields:

- byte32        	MMR_ROOT						//captures current MMR root
- byte32[64]     MMR_PEAKS                       //captures all current peak values
- uint8               HIGHEST_PEAK                  //captures the highest peak of the mountain range
- uint64             PREVIOUS_BLOCK             //captures previous block processed by cell
- uint128           ACCUM_DIFFICULTY         //total difficulty accumulated as of PREVIOUS_BLOCK

The type script should enforce the following constraints:

1. The transaction contains 1 input cell and 1 output cell
2. The `header_deps` referenced begin at (PREVIOUS_BLOCK+1)
3. The input's lock script is equal to lock script applied to the output

The type script of this cell will allow any user to use `header_deps` to read the block(s) following PREVIOUS_BLOCK, add them to the MMR and store the accumulated difficulty value.

MMR_ROOT is a linked list of all current peaks, as recently recommended by Peter Todd [[17]](https://github.com/zmitton/merkle-mountain-range/pull/1#issuecomment-489445900).

**3.4.2 FlyClient CKB proofs**

Here is the process for delivering FlyClient CKB proofs:

1. A client will request the latest block header from full nodes;

2. Full node will deliver the latest MMR commitment transaction, along with a Merkle path proving its inclusion in a transaction root in a valid block;

3. Full node will deliver all block headers between the block that includes the MMR commitment transaction and latest block header;

   *(*`header_deps` *can only read header information 4 epochs after a block's confirmation)*

4. Full node will deliver a random sampled proof of block inclusion using randomness derived from a hash of the latest block header (as described in 2.2.2).

This technique produces proofs of about 2 megabytes, more information about proof size is listed in Appendix A.

**3.4.3 Deviation from FlyClient: Verification of Difficulty Adjustments**

FlyClient prescribes that the block's time value be appended the block header value as it is added to the MMR, so that difficulty transitions can be checked.

This design however cannot be applied to Nervos CKB because in addition to elapsed time, CKB consensus rules also utilize orphan rate in difficulty adjustment calculations. This requires additional information to check difficulty transitions that are outside the scope of the FlyClient protocol.

Fortunately, difficulty raising attacks [[10]](https://arxiv.org/abs/1312.7013) are impractical when difficulty is adjusted over epochs. Instead of a user "getting very lucky" and finding one extremely rare block, they must quickly find a very long series of blocks as opposed to just one. More details about the practicality of this attack can be found in [[18]](https://youtu.be/0I9VTzIn7BQ?t=382) (video).

*Confirmation is needed that this deviation would not cause implementation or security issues.*

**3.4.4 Deviation from FlyClient: Checking MMR commitments in sampled blocks**

FlyClient prescribes that a proof is provided that each sampled block commits to a valid subtree of the latest MMR commitment (as described in 2.2.2), however the design outlined in (3.4.1) stores MMR commitments in transactions. Thus there is no certainty that a sampled block will contain a MMR commitment and the proposed design omits this aspect of FlyClient.

*Confirmation is needed that this deviation would not cause security issues.*



## References

1. [FlyClient](https://eprint.iacr.org/2019/226.pdf) 
2. [NIPoPoW](https://eprint.iacr.org/2017/963.pdf)
3. [Bitcoin](https://bitcoin.org/bitcoin.pdf)
4. [Proofs of Proof of Work with sub-linear complexity](https://fc16.ifca.ai/bitcoin/papers/KLS16.pdf)
5. [Bitcoin Talk post - The High-Value-Hash Highway](https://bitcointalk.org/index.php?topic=98986.0;all)
6. [Majority is not Enough: Bitcoin Mining is Vulnerable](https://arxiv.org/abs/1311.0243)
7. [Merkle Mountain Ranges (Peter Todd)](https://github.com/opentimestamps/opentimestamps-server/blob/master/doc/merkle-mountain-range.md)
8. [Merkle Mountain Ranges (Grin)](https://github.com/mimblewimble/grin/blob/master/doc/mmr.md#structure)
9. [Merkle Mountain Range (zmitton)](https://github.com/zmitton/merkle-mountain-range)
10. [Theoretical Bitcoin Attacks with less than Half of the Computational Power](https://arxiv.org/abs/1312.7013)
11. [Beam FlyClient Implementation](https://github.com/BeamMW/beam/blob/42ef470c322a230cfa45f7f268bf37987da92057/core/fly_client.cpp) 
12. [implement Flyclient by Benedikt Bunz, Lucianna Kiffer, Loi Luu and Mahdi Zamani (Grin)](https://github.com/mimblewimble/grin/issues/1555)
13. [[ZIP 221] Adding MMR Proofs to Block Headers.. (ZCash)](https://github.com/zcash/zips/pull/220)
14. [ECIP 1055: Flyclient - Succint PoW Using Merkle Mountain Ranges (FlyClient)](https://ecips.ethereumclassic.org/ECIPs/ecip-1055)
15. [Short fork & MMR discourse (Grin)](https://github.com/mimblewimble/grin/issues/1555#issuecomment-425543073)
16. [FlyClient commitments to contract state through transactions (ETC)](https://youtu.be/zcpSHeVgoos?t=2512)
17. [Peter Todd suggesting storing MMR peaks in linked list](https://github.com/zmitton/merkle-mountain-range/pull/1#issuecomment-489445900) 
18. [David Vorick presentation detailing impracticality of a difficulty raising attack on Bitcoin](https://youtu.be/0I9VTzIn7BQ?t=382)



## Appendix

### Appendix A: Proof size estimates

An estimate for proof size summed from the data below is **2,283,943 bytes**. Proofs consist of 3 elements:

1. Previous commitment transaction, block header and transaction inclusion proof
2. Block headers between commitment transaction and current chain tip. 
3. Proofs of block inclusion in the most recent MMR commitment							

**A.1 Commitment transaction, block header and inclusion proof**

​	Block header transaction was committed in	

​																   212 bytes

​	Commitment transaction

​			Input 

​				Previous transaction hash:     32 bytes

​				Previous outpoint index: 	  	1 byte

​				Header_dep:							  32 bytes

​			Output

​				MMR_ROOT								 32 bytes

​				MMR_PEAKS (varies) 			   864 bytes (27*32) max estimate, up to 134 million blocks

​				ACCUM_DIFF								16 bytes

​				HIGHEST_PEAK							  1 byte

​				PREVIOUS_BLOCK						8 bytes

​	Transaction inclusion in commitment root proof	

​																	288 bytes (9*32) max estimate, up to 512 transactions per block

**A.1 total**										  		**1,486 bytes**									



**A.2 Intermediate block headers** 

`header_deps` can only be used after 4 epoch confirmations. (4) 4 hour epochs, min block time 7 seconds:

((4 *4 hours) * 3600 seconds) /  7 second min block time = 8,228 block headers (upper bound)



**A.2 total**		  8,228  * 212			 **1,744,457 bytes** (upper bound)



**A.3 FlyClient MMR commitment block Inclusion proofs (max ~500 blocks)**

Sampled block headers          			106,000 bytes 	(500 * 212)

Block inclusion proofs 						432,000 bytes	(500 * 27 * 32) (inclusion proofs up to 134 million blocks) 

**A.3 Total**								 				**538,000 bytes** (likely lower as upper leaves can be cached)

![flyclient blocks queried](.\flyclient blocks queried.PNG)

​	*Only "Queries" line and left y-axis (BLOCKS QUERIED) is relevant to A.3 (from FlyClient paper)*



### Appendix B: Proof compression (of A.2)

Appendix A shows that the largest portion of the CKB super-light client proof comes from SPV-style header validation of recent blocks (A.2) that is needed because CKB requires 4 epochs of confirmations to read a block header through `header_deps`, the blocks included in the MMR commitment will lag the chain tip by thousands of blocks.

The properties outlined in the NIPoPoW construction (2.1.1), however show us that work can be succinctly proven through super-blocks. For instance, if a logarithmic number of block headers can be used (14 instead of 8,228) the upper bound of this portion of the proof can be reduced from 1,744,457 bytes to 2,968 bytes, bringing total proof size under 1 megabyte. 

*Research is needed to determine the most effective way to utilize this property and to determine the exact parameters for proof optimization, as well as security considerations of this approach.*

*FlyClient does prescribe SPV-style validation of n recent blocks to account for short forks an attacker may be capable of generating.*



### Appendix C: Example - Example MMR contract logic (no difficulty)

```c
//TODO: add header_deps, difficulty reading for current header
//TODO: add difficulty logic
//TODO: add block height logic
byte32 		MMR_ROOT
byte32[64] 	MMR_PEAKS;
uint8 		HIGHEST_PEAK;

main(byte32[] _input){
	for(uint32 i=0; i < _input.length; i++){	//add each value to the MMR
		_mmrHash(_input[i], 0);
		}
	_bagPeaks();								//create linked list of peaks
}

_mmrHash(byte32 _input, uint8 _height){
	if(MMR_PEAKS[_height] != 0){				//value exists at this height
		byte32 memHashVal = MMR_PEAKS[_height];	//store current value at this height
        MMR_PEAKS[_height] = 0;					//clear current value at this height
        _mmrHash(blake2b(memHashVal,_input), _height++);	//recursive call up the tree
    } else {									//value is blank at this level   
		MMR_PEAKS[_height] = _input;			//store current value at this height
        if(_height > HIGHEST_PEAK){
			HIGHEST_PEAK = _height;				//update HIGHEST_PEAK
        	}
    }
}

_bagPeaks(){									//linked list of peaks(MMR root)(ascending)
	byte32 memMMRRoot;
	for(uint8 i=0; i <= HIGHEST_PEAK; i++){
		if(MMR_PEAKS[i] != 0) {
			memMMRRoot = blake2b(memMMRRoot,MMR_PEAKS[i]);
			}
		}
	MMR_ROOT = memMMRRoot;						//set MMR Root
}
```
