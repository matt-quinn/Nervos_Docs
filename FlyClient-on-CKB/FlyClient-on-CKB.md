# Transaction-based FlyClient on CKB

These are ideas to support a super-light client for CKB, allowing for sub-linear verification of CKB's proof of work (compared to verifying all block headers) without any consensus changes. The design makes use of an on-chain contract to implement a [FlyClient](https://eprint.iacr.org/2019/226.pdf) MMR and produces proofs of CKB transaction inclusion in about 1.5 megabytes.

The key advantage of this design is that it requires no changes to existing mining software or consensus rules and can be implemented immediately. On the negative side, it does rely on processing of transactions to keep contract data updated and is also subject to censorship attacks by miners. 

A soft fork could resolve these issues without changing the foundation of the system and also substantially decrease proof size (blocks can be immediately added to the MMR, eliminating SPV-style verification for blocks added within the most recent 4 epochs not yet available through `header_deps`).

Background information:

- [Introduction to Light Clients](https://medium.com/@m.quinn/introduction-to-light-clients-5f5dc122fe3c)
- [Introduction to super-light client: NIPoPoW](https://medium.com/@m.quinn/introduction-to-super-light-clients-nipopows-da8cf3914a8a)
- [Introduction to super-light client: FlyClient](https://medium.com/@m.quinn/introduction-to-super-light-clients-flyclient-12dbd28f72a9)



## 1. Super-light clients

A properly designed light client protocol will prevent adversaries from including invalid (fake) blocks that could be used to deceive a light client into following an alternate chain of the same length (or difficulty) as that of the honest chain. 

While SPV proofs accomplish this by requiring verifiers to check all headers, foundational research has created "super-light client" designs, which allow for verification of a blockchain's proof of work using only a fraction of the chain's block headers. 

Super-light clients allows for sub-linear verification of the work expended to produce a blockchain and significantly compress light client proof sizes, *without requiring any trust in a 3rd party*.



## 2. Choosing FlyClient over NIPoPoW

As the base of a super-light client, FlyClient is a straightforward choice over NIPoPoW for three reasons:

1. While NIPoPoW does not explicitly chain all blocks together as part of the mining protocol, FlyClient creates commitments to the blockchain's history.

2. When relying on NIPoPoW proofs for security, an incentive is introduced for attackers to bribe miners into discarding blocks that if confirmed would contribute to the quality of NIPoPoW proofs (lowering attack costs). 

   Block discarding not only lowers the effectiveness and security of NIPoPoW proofs, it also lowers overall chain security (difficulty adjustments will be based on less than actual hash rate).

3. Construction of NIPoPoW proofs for blockchains with variable difficulty is still a topic for future research. 



## 3. FlyClient on CKB (Transaction-based)

**3.1 FlyClient CKB info cell**

The implementation of FlyClient on CKB begins with an "anyone can unlock" cell that allows for a user to add to the MMR. MMR references can be found at 

FlyClient uses a "Difficulty MMR" to capture difficulty information at each MMR node and use it later to do difficulty-based sampling (so sampling is spread according to accumulated difficulty instead of number of blocks). More information about MMRs or "Merkle Mountain Ranges" can be found here:  [1](https://github.com/opentimestamps/opentimestamps-server/blob/master/doc/merkle-mountain-range.md), [2](https://github.com/mimblewimble/grin/blob/master/doc/mmr.md#structure), [3](https://github.com/zmitton/merkle-mountain-range), [4](https://youtu.be/zcpSHeVgoos?t=605).

The cell contains the following fields:

- byte32        	MMR_ROOT						//current MMR root

- struct              MMR_PEAK {

  ​    byte32        peakValue                   //root of sub-tree

  ​    uint128       accumulatedDifficulty      //accumulated difficulty of MMR nodes below peak

  }

- uint8               HIGHEST_PEAK                  //the highest peak of the mountain range

- uint64             PREVIOUS_BLOCK             //previous block processed by cell

A type script should enforce the following constraints:

1. The cell is used as input 1
2. The transaction contains 1 input cell and 1 output cell *or* 2 input cells and 2 output cells (no fee vs fee paid)
3. The `header_deps` referenced begin at (PREVIOUS_BLOCK+1)
4. Input 1's lock script is equal to lock script applied to output 1
5. Capacity of input 1 == capacity of output 1

The type script of this cell will allow any user to use `header_deps` to read the block(s) following PREVIOUS_BLOCK, add them to the MMR and store the accumulated difficulty value. 

The type script could require a delay of a certain number of blocks since the last commitment to reduce the frequency of commitment transactions.

MMR_ROOT is a linked list of all current peaks, as [recently recommended](https://github.com/zmitton/merkle-mountain-range/pull/1#issuecomment-489445900) by Peter Todd (inventor of MMR).



**3.2 FlyClient CKB proofs**

Process for delivering FlyClient CKB proofs:

1. A client will request the latest block header from full nodes;

2. Full node will deliver the latest MMR commitment transaction, along with a Merkle path proving its inclusion in a transaction root in a valid block;

3. Full node will deliver all block headers (SPV-style verification) between the block referenced in (2) and the latest block header;

   (`header_deps` can only read header information following 4 epochs of block confirmation)

4. Full node will deliver Merkle paths to show inclusion of the randomly sampled blocks in the MMR root contained in (2). Randomness is derived from a hash of the latest block header (FlyClient protocol).

This technique produces proofs of about 2 megabytes, more information about proof size is listed in Appendix A.



## 4. Deviations from FlyClient

**4.1 Verification of Difficulty Adjustments**

FlyClient prescribes that the block's time value be appended to the block header value as it is added to the MMR, so that difficulty transitions can be checked and [difficulty raising attacks](https://arxiv.org/abs/1312.7013) cannot be executed.

This design cannot be applied to CKB because CKB consensus rules utilize orphan rate in difficulty adjustment calculations (in addition to epoch time). This requires additional information to check difficulty transitions that are outside the scope of the FlyClient protocol.

Fortunately, difficulty raising attacks are impractical when difficulty is adjusted over epochs. Instead of a user "getting very lucky" and finding one extremely rare block, they must quickly find a very long series of blocks. More details about the impracticality of this attack can be found in [this video](https://youtu.be/0I9VTzIn7BQ?t=382).



**4.2 Checking MMR commitments in sampled blocks**

FlyClient prescribes that a proof be provided that each sampled block commits to a valid subtree of the latest MMR commitment, however this design stores MMR commitments in transactions. There is no certainty that a sampled block will contain a MMR commitment and the proposed design omits this aspect of FlyClient.

*Confirmation is needed that these deviations would not cause implementation or security issues.*



## Appendix

### Appendix A: Proof size estimates

An estimate for proof size summed from the data below is **2,284,182 bytes**. Proofs consist of 3 elements:

1. Previous commitment transaction, block header and transaction inclusion proof
2. Block headers between commitment transaction and current chain tip. 
3. Proofs of block inclusion in the most recent MMR commitment							



**A.1 Commitment transaction, block header and inclusion proof**

Block header that transaction was committed (208 bytes)

<u>Commitment transaction</u>

​    Input: Previous transaction hash (32 bytes)

​    Input: Previous outpoint index (4 bytes)

​    Input: Header_dep (32 bytes)

​    Output: MMR_ROOT (32 bytes)

​    Output: MMR_PEAKS: (varies) 1,296 bytes (27*(32+16)) max estimate (up to 134 million blocks)

​    Output: HIGHEST_PEAK (1 byte)

​    Output: PREVIOUS_BLOCK (8 bytes)

Transaction hash (32 bytes)

Transaction inclusion in commitment root proof (288 bytes) (9*32) max estimate (up to 512 transactions per block)

**A.1 total 1,725 bytes**									


**A.2 Intermediate block headers** 

`header_deps` can only be used after 4 epoch confirmations, max block length of epoch is 1800:

4 * 1800 = 7,200 block headers (upper bound)

7,200  * 208 = 1,497,600 bytes

**A.2 total: 1,497,600 bytes** (upper bound)



**A.3 FlyClient MMR commitment block inclusion proofs (max ~500 blocks)**

​     Sampled block headers: 104,000 bytes (500 * 208)

​     Block inclusion proofs: 432,000 bytes (500 * 27 * 32) (inclusion proofs up to 134 million blocks) 



![flyclient blocks queried](./flyclient-blocks-queried.PNG)

​	*Only "Queries" line and left y-axis (BLOCKS QUERIED) is relevant to A.3 (from FlyClient paper)*

**A.3 Total 538,000 bytes** (likely lower as upper leaves of MMR can be cached)



**Total A.1-A.3 = 1,553,125 bytes**



### Appendix B: Example - Example Difficulty MMR contract logic (pseudocode)

```c
struct      mmr_peak {
    byte32  peakValue;
    uint128 accumDifficulty;
}

//declare global variables
struct mmr_peak MMR_PEAKS[64];
byte32          MMR_ROOT
uint8           HIGHEST_PEAK;
uint64          PREVIOUS_BLOCK
    
main(){
    struct header {
        byte32  headerHash;
        uint128 difficulty;
        uint64  blockNumber;
    }
    struct header headerArray[];
    
     (load header_deps into headerArray)
    
    uint64 memBlockNumber = PREVIOUS_BLOCK + 1; //block sequence starts at next block
    
    for(uint16 i=0; i < _headerArray.length; i++){ //loop through headerArray and add to MMR
        if(headerArray[i].blockNumber != memBlockNumber){ //ensure correct sequence of blocks
            return 1;
        }
        else {
            _addToMMR(headerArray[i].headerHash, headerArray[i].difficulty, 0); //add to MMR
            memBlockNumber++; //continue block number sequence
        }   
    
    _bagPeaks();                                    //create linked list of peaks
    PREVIOUS_BLOCK = memBlockNumber;                //store last block number added to MMR
}

_addToMMR(byte32 _blockHash, uint128 _difficulty, uint8 _height){

    if(MMR_PEAKS[_height].peakValue != 0){           //value exists at this height
 
        //variables for combining existing peak with new leaf
        byte32 memBlockHash;
        uint128 memAccumDifficulty;
       
        //hash together existing peak and new leaf, add their difficulty values
        memBlockHash = blake2b(MMR_PEAKS[_height].peakValue, MMR_PEAKS[_height].accumDifficulty, _blockHash, _difficulty);
        memAccumDifficulty = MMR_PEAKS[_height].accumDifficulty + _difficulty;
        
        //clear existing peak data at this height
        MMR_PEAKS[_height].peakValue = 0;			 
        MMR_PEAKS[_height].accumulatedDifficulty = 0;
        
        //call recursively up the tree
        _addToMMR(memBlockHash, memAccumDifficulty, _height++);
        
    } else {                                        //value is blank at this level   
        
        MMR_PEAKS[_height].peakValue = _blockHash;  //store input value at this height
        MMR_PEAKS[_height].accumDifficulty = _difficulty; //store input value at this height
    
        if(_height > HIGHEST_PEAK){
            HIGHEST_PEAK = _height;                 //update HIGHEST_PEAK
            }
    }
}

_bagPeaks(){                                 //create linked list of peaks(MMR root)(ascending)
    byte32 memMMRvalue;   
    for(uint8 i=0; i <= HIGHEST_PEAK; i++){
        if(MMR_PEAKS[i] != 0) {
            memMMRvalue = blake2b(memMMRvalue, MMR_PEAKS[i].peakValue, MMR_PEAKS[i].accumDifficulty);
            }
        }
    MMR_ROOT = memMMRvalue;                   //set MMR Root
}
```
