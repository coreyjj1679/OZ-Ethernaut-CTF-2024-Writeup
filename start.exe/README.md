## start.exe

### description

This transaction seems to be the start of something big. Can you figure out what it is? https://sepolia.etherscan.io/tx/0x73fcb6eec33280c39a696b8db0f7b3f71f789c28ef722e0c716f9c8cef6aa040

### solution

The flag is hided in the input data.

```typescript
const flag =
  "0x4f5a4354467b304e335f4731344e545f4c3341505f4630525f4d344e4b314e44";
const flagAcsii = await ethers.toUtf8String(flag);
console.log(flagAcsii);
```
