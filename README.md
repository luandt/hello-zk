# Hello-zk 
A hello-world zk using Circom to build a circuit and can verify locally or with Solidity

Note: This repo has been using [PaulBerg template](https://github.com/PaulRBerg/foundry-template/generate)

## Getting Started
The tutorial is using Circom version 2 for writing circuits and snarkjs to generate proofs and locally verify proofs.

Refer to [Circom installation guide](https://docs.circom.io/getting-started/installation/#installing-circom) for more details


### Writing a circuit
Let take a simple example that someone want to prove that thay know the input of this arithmatic formula (`a * b = 55`) but do not want to disclose those values of a and b
```
pragma circom 2.0.0;

/*This circuit template checks that c is the multiplication of a and b.*/  

template Multiplier2 () {  

   // Declaration of signals.  
   signal input a;  
   signal input b;  
   signal output c;  

   // Constraints.  
   c <== a * b;  
}
```

### Compile circuit
Save to multiplier2.circom file under circom, then run the below command to compile the circuit
```
circom multiplier2.circom --r1cs --wasm --sym --c
```

### Create input file
Create a input file with json format like below (already existed in the repo)
``` 
{"a": "5", "b": "11"} 
```

### Compute witnesses
Make sure to be located in `multipliers_js` directory to execute the command
```
node generate_witness.js multiplier2.wasm ../input.json witness.wtns
```

### Proving the circuit using the Groth16 zk-SNARK protocol
#### Trusted setup, phase 1
First, we start a new "powers of tau" ceremony:
```
snarkjs powersoftau new bn128 12 pot12_0000.ptau -v
```
Then, we contribute to the ceremony:
```
snarkjs powersoftau contribute pot12_0000.ptau pot12_0001.ptau --name="First contribution" -v
```
#### Trusted setup, phase 2
Run below command to start the generation
```
snarkjs powersoftau prepare phase2 pot12_0001.ptau pot12_final.ptau -v
```

Next, generate a .zkey file that will contain the proving and verification keys together with all phase 2 contributions. 
```
snarkjs groth16 setup multiplier2.r1cs pot12_final.ptau multiplier2_0000.zkey
```
Contribute to the phase 2 of the ceremony:
```
snarkjs zkey contribute multiplier2_0000.zkey multiplier2_0001.zkey --name="1st Contributor Name" -v
```
Export the verification key:
```
snarkjs zkey export verificationkey multiplier2_0001.zkey verification_key.json
```

### Generating a Proof
```
snarkjs groth16 prove multiplier2_0001.zkey witness.wtns proof.json public.json
```

This command generates a Groth16 proof and outputs two files:
- proof.json: it contains the proof.
- public.json: it contains the values of the public inputs and outputs.

### Verifying a Proof
```
snarkjs groth16 verify verification_key.json public.json proof.json
```

### Verifying from a Smart Contract
First, we need to generate the Solidity code using the command:
```
snarkjs zkey export solidityverifier multiplier2_0001.zkey verifier.sol
```
This contract can be deploy on a testnet network such as sepolia or holeski to verify on-chain

After deploying on-chain, the `view` function `verifyProof` can be call to check whether proof is correct or not (returning true is correct otherwise is incorrect)

o facilitate the call, you can use snarkJS to generate the parameters of the call by typing:
```
snarkjs generatecall
```
The output has hex format which must be converted to decimal compatible with arguments type in the smart contract.

Note: I deployed and verified a contract at [0x87035f7005350500e0282B55F8a430e8286F7936](https://sepolia.etherscan.io/address/0x87035f7005350500e0282b55f8a430e8286f7936)

I also added a script to verify on-chain from the above contract in the folder `script/Deploy.s.sol`
Run this command to see the result (it will take several seconds to make an rpc call)
```
forge script --chain sepolia script/Deploy.s.sol:verifyProofOC --rpc-url $SEPOLIA_RPC_URL -vvvv
```

You can generate your proof and re-test by changing the value in this function
```javascript
verifier.verifyProof([21344529039867642738510397377876146259038501554931325406214543597454379755439,8976648817476389217708538658785254934268439161015719064362999030809393764999],[[17403079811870541704844424248523753069906068604490748359863959777843324819475,9692872315583656082798304049253911348579573220592590130503699873713971147478],[21805756855415707546531957795313603862597671569597417504655696766330683494810,21197175524545537519555648600584355203435948667399476861668394613100750130601]],[3849134695378743302609165268144292894969238300245668850466502672023257017945,8376787217373855464599860663895279030958888153224591414693039755943740304369],
[uint(55)]);
```

Feel free to add any circuit to test by your self.
