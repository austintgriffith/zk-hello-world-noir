# ZK Hello World - Merkle Tree

---
## Checkpoint 0: Environment
- install dependencies
Before you begin, you need to install the following tools:

- [Node (>= v18.17)](https://nodejs.org/en/download/)
- Yarn ([v1](https://classic.yarnpkg.com/en/docs/install/) or [v2+](https://yarnpkg.com/getting-started/install))
- [Git](https://git-scm.com/downloads)
- [Nargo](https://noir-lang.org/docs/getting_started/quick_start#installation)
- [bb](https://noir-lang.org/docs/getting_started/quick_start#proving-backend)

Then download the challenge to your computer and install dependencies by running:

```sh
npx create-eth@0.1.0 -e calvbore/zk-hello-world-noir zk-hello-world
cd zk-hello-world
```

> in the same terminal, start your local network (a blockchain emulator in your computer):

```sh
yarn chain
```

> in a second terminal window, 🛰 deploy your contract (locally):

```sh
cd zk-hello-world
yarn deploy
```

> in a third terminal window, start your 📱 frontend:

```sh
cd zk-hello-world
yarn start
```

📱 Open http://localhost:3000 to see the app.

> 👩‍💻 Rerun `yarn deploy` whenever you want to deploy new contracts to the frontend. If you haven't made any contract changes, you can run `yarn deploy --reset` for a completely fresh deploy.

---
## Checkpoint 1: Smart Contract Merkle Tree
First, install the `zk-kit` dependency in your solidity workspace.

```
yarn workspace @se-2/hardhat add @zk-kit/lean-imt.sol
```

You'll need to import the `InternalLeanIMT` Contract and `LeanIMTData` struct from zk-kit into `YourContract.sol`.

```
import { InternalLeanIMT, LeanIMTData } from "@zk-kit/lean-imt.sol/LeanIMT.sol";
```

Instantiate a merkle tree data variable in `YourContract.sol`, make sure to use the `InternalLeanIMT` contract library for the struct!

```
using InternalLeanIMT for LeanIMTData;
LeanIMTData public mt;
```

Create an event to track all the added members, we want to keep this so we can recreate the merkle tree at any time.

```
event NewLeaf(uint256 indexed index, uint256 indexed leaf);
```

Write function that accepts ETH and gives membership in exchange. It should take in one `uint256` value and emit the `NewLeaf` event. Test it in the frontend with the `Debug Contracts` tab.

```
function insert(uint256 _hashedSecret) public payable {
    if (premium) require(msg.value > 0);
    mt._insert(_hashedSecret);
    emit NewLeaf(mt.size-1, _hashedSecret);
}
```

Add a couple convenience functions. One to see the root of the contract's merkle tree and another to get the index of a leaf after it has been committed.

```
function getRoot() public view returns (uint256) {
	return mt._root();
}

function getLeafIndex(uint256 _leaf) public view returns (uint256) {
	return mt._indexOf(_leaf);
}
```

Before you can deploy the contract you will need to modify the deployment script (`packages/hardhat/deploy/00_deploy_your_contract.ts`) so that the libraries will work properly. Deploy the `PoseidonT3` contract, it is a dependency of the `InternalLeanIMT` library that is used.

```
const poseidon = await deploy("PoseidonT3", {
	from: deployer,
	log: true,
	autoMine: true
});
```

Pass its `address` as a library to `YourContract`'s deployment function.

```
await deploy("YourContract", {
	from: deployer,
	// Contract constructor arguments
	args: [deployer],
	log: true,
	// autoMine: can be passed to the deploy function to make the deployment process faster on local networks by
	// automatically mining the contract deployment transaction. There is no effect on live networks.
	autoMine: true,
	libraries: {
		PoseidonT3: poseidon.address,
	}
});
```

Now `YourContract` Should deploy correctly.

---
## Checkpoint 2: Noir Merkle Proof Verification Circuit
The first thing we need to do for our circuit is add `zk-kit` merkle tree as dependency to `packages/nargo/circuits/your_circuit/nargo.toml`.

```
[dependencies]
binary_merkle_root = { git = "https://github.com/privacy-scaling-explorations/zk-kit.noir", tag = "binary-merkle-root-v0.0.1", directory = "packages/binary-merkle-root" }
```

Import it into `packages/nargo/circuits/your_circuit/src/main.nr`. Right at the top of the file add the import with all the traits that are needed. While you're up here import the `poseidon` hash function from the standard library. 

```
use binary_merkle_root::binary_merkle_root; 
use std::hash::poseidon::bn254::hash_2;
```

Define the `hasher` function that will be used inside the merkle tree calculations. It will be a simple wrapper around the `poseidon` `hash` function.

```
fn hasher(leaves: [Field; 2]) -> Field {
	hash_2(leaves)
}
```

Set all of the inputs necessary for the `main` function.
- `secret` along with `salt` are used to by the prover to claim that they know a secret value hashed behind one of leaves of the merkle tree managed by `YourContract`. 
- `salt` a random value hashed with `secret` to generate a merkle tree leaf. In another design this could be used as a nullifier.
- `indexes` will encode the position of the merkle tree leaf.
- `siblings` is an array forming the hash path from the merkle leaf to the merkle root. Arrays in noir must be a fixed length.
- `pub_root` will be compared to the merkle root of the tree stored in `YourContract`.
- `depth` will be the depth of the tree in `YourContract`
- `msg` is the message we want to set `YourContract`'s greeting to. This value doesn't actually do anything inside of the circuit, but it is a good practice to tie any values that change the state of a smart contract directly to the proof giving validity to those changes otherwise the proof could be used to front run the authentic contract call with different values. Strings in noir must be a fixed length.

```
fn main(
	secret: Field,
	salt: Field,
	indexes: Field,
	siblings: [Field; 4],
	pub_root: pub Field,
	depth: pub u32,
	msg: pub str<10>
) {
//...
}
```

Generate the merkle `leaf` by feeding `secret` and `salt` to `hasher`.

```
let leaf = hasher([secret, salt]);
```

Then you'll need to figure out the index where the empty elements in the siblings array begin.

```
let mut siblings_num = 0;
for i in 0..siblings.len() {
	if siblings[i] != 0 {
		siblings_num += 1;
	}
}
```

Calculate the merkle root. You'll need to convert the index to bits, first. Then pass the `hasher`, `leaf`, `siblings_num`, `index_bits`, and `siblings` to the `binary_merkle_root` that you imported earlier.

```
let index_bits: [u1; 4] = indexes.to_le_bits();

let bin_root = binary_merkle_root(hasher, leaf, siblings_num, index_bits, siblings);
```

Now `assert` that `pub_root` and the calculated `bin_root` are the same value.

```
assert(pub_root == bin_root);
```

For safe measure we can add assertions that `depth` is less than the length of `siblings`, and that `msg` is in fact equal to itself.

```
assert(depth <= siblings.len());
assert(msg == msg);
```

## Checkpoint 3: Circuit Smart Contract Verifier

Run `yarn nargo:compile` in your terminal and you'll find that a new solidity file in your contracts directory. Import the new contract into `YourContract.sol` and inherit `UltraVerifier` in `YourContract`.

```
import "./your_circuitVerifier.sol";

contract YourContract is UltraVerifier {
//...
}
```

Write a function that takes in the a proof and public values for `your_circuit`.

```
function setGreetingAnon(bytes calldata _proof, bytes32[] calldata _publicInputs) public {
//...
}
```

Parse `_publicInputs` into values you can use.

```
uint256 proved_root = uint256(_publicInputs[0]);
uint256 proved_depth = uint256(_publicInputs[1]);

bytes32 message;
bytes32[] memory messageChars = _publicInputs[2:12];
for (uint256 i; i<messageChars.length; i++) {
    message = message | messageChars[i] << 8*(messageChars.length-i-1);
}
```

Check that the `proved_root` and `proved_depth` values match those stored by `YourContract`.

```
require(proved_root == mt.root(), "Root does not match");
require(proved_depth == mt.depth, "Depth does not match");
```

And check the validity of the proof.

```
bool validity = this.verify(_proof, _publicInputs);
require(validity == true, "Invalid proof");
```

After the proof is found to be valid set the greeting message of `YourContract` to the message tied to the proof.

```
greeting = message;
totalCounter += 1;
```

As this function stands anyone could dig through old transactions and replay a proof to reset the greeting to the value tied to the proof. It is good practice to track if a proof has been used before. Add a `mapping` from `bytes32` to a `bool` in `YourContract`.

```
mapping(bytes32 => bool) public isProved;
```

calculate the `keccack256` hash of `_proof` and `_publicInputs` right at the top of the anonymous greeting setter function and at the end set the mapping to `true`.

```
bytes32 proofHash = keccak256(abi.encode(_proof, _publicInputs));
//...
isProved[proofHash] = true;
```

Then add a check at the beginning of the function to see if the proof has been used before.

```
bytes32 proofHash = keccak256(abi.encode(_proof, _publicInputs));
require(isProved[proofHash] == false, "Proof has been used");
//...
```

`yarn deploy` the updated contract to your local chain.

---
## Checkpoint 4: Frontend Secret Committer

---
## Checkpoint 5: Frontend Secret Prover

---
