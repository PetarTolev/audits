# [M-01] Use safeMint instead of mint in the FootiumClub contract

## Summary

Use `safeMint` instead of `mint` in `FootiumClub::safeMint`.

## Vulnerability Detail

The function name `safeMint` in the FootiumClub contract is misleading because it is actually using the `mint` function (not the safe version), especially since all other Footium contracts use the `safeMint` function. It's important to note that if the `to` address is a contract that does not support the ERC721 standard, the NFT can become frozen in that contract.

This information is documented in ERC721Upgradeable.sol by OpenZeppelin:

Ref: [https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/token/ERC721/ERC721Upgradeable.sol#L257-L291](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/token/ERC721/ERC721Upgradeable.sol#L257-L291)

```solidity
/**
 * @dev Mints `tokenId` and transfers it to `to`.
 *
 * WARNING: Usage of this method is discouraged, use {_safeMint} whenever possible
 *
 * Requirements:
 *
 * - `tokenId` must not exist.
 * - `to` cannot be the zero address.
 *
 * Emits a {Transfer} event.
 */
function _mint(address to, uint256 tokenId) internal virtual {
```

## Impact

The potential impact is the permanent loss (or 'stuck') of NFTs for users of the protocol. This could occur if the NFTs are minted to a contract that does not support the ERC721 standard or if they are mistakenly minted to the wrong address. Hence, this issue is considered to have a medium severity.

## Code Snippet

https://github.com/sherlock-audit/2023-04-footium/blob/main/footium-eth-shareable/contracts/FootiumClub.sol#L65

```solidity
  _mint(to, tokenId);
```

## Tool used

Manual Review

## Recommendation

To ensure ERC721 implementation support, use `safeMint`instead of`mint`.

# [M-02] Code does not handle ERC20 tokens with special transfer implementation

## Summary

Calls to the `ERC20::transfer` method should always be checked.

## Vulnerability Detail

Some `ERC20` tokens do not revert on failure in `transfer`, but instead return `false` (e.g. [ZRX](https://etherscan.io/address/0xe41d2489571d322189246dafa5ebde1f4699f498#code)).

Other tokens (e.g. [USDT](https://arbiscan.io/token/0xFd086bC7CD5C481DCC9C85ebE478A1C0b69FCbb9#code)) don't correctly implement the `ERC20` standard and their `transfer` function return void instead of boolean. Calling these functions with the correct `ERC20` function signatures will always revert.

## Impact

The impact is potentially **permanently lost (stuck)** tokens, but it needs a special `ERC20` (e.g. [ZRX](https://etherscan.io/address/0xe41d2489571d322189246dafa5ebde1f4699f498#code)) token to be used, hence Medium severity.

## Code Snippet

[https://github.com/sherlock-audit/2023-04-footium/blob/main/footium-eth-shareable/contracts/FootiumPrizeDistributor.sol#L130](https://github.com/sherlock-audit/2023-04-footium/blob/main/footium-eth-shareable/contracts/FootiumPrizeDistributor.sol#L130)

```solidity
File: FootiumPrizeDistributor.sol
/**
 * @notice Claims ERC20 tokens from the contract
 * @param _to Prize receiver account address
 * @param _token ERC20 token contract address
 * @param _amount ERC20 token amount
 * @param _proof Merkle proof for `erc20MerkleRoot`
 * @dev The sender must provide a merkle proof for `erc20MerkleRoot`
 * Emits a {ClaimERC20} event.
 */
function claimERC20Prize(
      address _to,
    IERC20Upgradeable _token,
    uint256 _amount,
    bytes32[] calldata _proof
) external whenNotPaused nonReentrant {
      if (_to != msg.sender) {
          revert InvalidAccount();
    }

    if (
          !MerkleProofUpgradeable.verify(
              _proof,
            erc20MerkleRoot,
            keccak256(abi.encodePacked(_token, _to, _amount))
        )
    ) {
          revert InvalidERC20MerkleProof();
    }

    uint256 value = _amount - totalERC20Claimed[_token][_to];

    if (value > 0) {
          totalERC20Claimed[_token][_to] += value;
        _token.transfer(_to, value);
    }

    emit ClaimERC20(_token, _to, value);
}
```

[https://github.com/sherlock-audit/2023-04-footium/blob/main/footium-eth-shareable/contracts/FootiumEscrow.sol#L110](https://github.com/sherlock-audit/2023-04-footium/blob/main/footium-eth-shareable/contracts/FootiumEscrow.sol#L110)

```solidity
File: FootiumEscrow.sol
/**
 * @notice Transfers ERC20 tokens to `to` address.
 * @param erc20Contract ERC20 contract address.
 * @param to Token receiver address.
 * @param amount Token amount to transfer.
 * @dev only the club owner address allowed.
 */
function transferERC20(
      IERC20 erc20Contract,
    address to,
    uint256 amount
) external onlyClubOwner {
      erc20Contract.transfer(to, amount);
}
```

## Tool used

Manual Review

## Recommendation

To avoid this issue, it has become a common practice to use OpenZeppelin's `SafeERC20` library to handle such tokens. `SafeERC20` provides a set of `ERC20` wrapper functions that handle tokens with non-standard `transfer` implementations. These functions check the return value of the `transfer` function and revert if it returns `false`, preventing tokens from getting stuck in the contract.
