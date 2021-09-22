# gnosis
## acitons
2. send transaction
- contract
  - method 
    - execTransaction  
  - source code 
    - https://rinkeby.etherscan.io/address/0xd9db270c1b5e3bd161e8c8503c55ceabee709552#code
  - core code 
    ```
     /**
     * @dev Checks whether the signature provided is valid for the provided data, hash. Will revert otherwise.
     * @param dataHash Hash of the data (could be either a message hash or transaction hash)
     * @param data That should be signed (this is passed to an external validator contract)
     * @param signatures Signature data that should be verified. Can be ECDSA signature, contract signature (EIP-1271) or approved hash.
     * @param requiredSignatures Amount of required valid signatures.
     */
    function checkNSignatures(
        bytes32 dataHash,
        bytes memory data,
        bytes memory signatures,
        uint256 requiredSignatures
    ) public view {
        // Check that the provided signature data is not too short
        require(signatures.length >= requiredSignatures.mul(65), "GS020");
        // There cannot be an owner with address 0.
        address lastOwner = address(0);
        address currentOwner;
        uint8 v;
        bytes32 r;
        bytes32 s;
        uint256 i;
        for (i = 0; i < requiredSignatures; i++) {
            (v, r, s) = signatureSplit(signatures, i);
            if (v == 0) {
                // If v is 0 then it is a contract signature
                // When handling contract signatures the address of the contract is encoded into r
                currentOwner = address(uint160(uint256(r)));

                // Check that signature data pointer (s) is not pointing inside the static part of the signatures bytes
                // This check is not completely accurate, since it is possible that more signatures than the threshold are send.
                // Here we only check that the pointer is not pointing inside the part that is being processed
                require(uint256(s) >= requiredSignatures.mul(65), "GS021");

                // Check that signature data pointer (s) is in bounds (points to the length of data -> 32 bytes)
                require(uint256(s).add(32) <= signatures.length, "GS022");

                // Check if the contract signature is in bounds: start of data is s + 32 and end is start + signature length
                uint256 contractSignatureLen;
                // solhint-disable-next-line no-inline-assembly
                assembly {
                    contractSignatureLen := mload(add(add(signatures, s), 0x20))
                }
                require(uint256(s).add(32).add(contractSignatureLen) <= signatures.length, "GS023");

                // Check signature
                bytes memory contractSignature;
                // solhint-disable-next-line no-inline-assembly
                assembly {
                    // The signature data for contract signatures is appended to the concatenated signatures and the offset is stored in s
                    contractSignature := add(add(signatures, s), 0x20)
                }
                require(ISignatureValidator(currentOwner).isValidSignature(data, contractSignature) == EIP1271_MAGIC_VALUE, "GS024");
            } else if (v == 1) {
                // If v is 1 then it is an approved hash
                // When handling approved hashes the address of the approver is encoded into r
                currentOwner = address(uint160(uint256(r)));
                // Hashes are automatically approved by the sender of the message or when they have been pre-approved via a separate transaction
                require(msg.sender == currentOwner || approvedHashes[currentOwner][dataHash] != 0, "GS025");
            } else if (v > 30) {
                // If v > 30 then default va (27,28) has been adjusted for eth_sign flow
                // To support eth_sign and similar we adjust v and hash the messageHash with the Ethereum message prefix before applying ecrecover
                currentOwner = ecrecover(keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", dataHash)), v - 4, r, s);
            } else {
                // Default is the ecrecover flow with the provided data hash
                // Use ecrecover with the messageHash for EOA signatures
                currentOwner = ecrecover(dataHash, v, r, s);
            }
            require(currentOwner > lastOwner && owners[currentOwner] != address(0) && currentOwner != SENTINEL_OWNERS, "GS026");
            lastOwner = currentOwner;
        }
    }
    ```
1. create safe
- contract 
  - method
    - createProxyWithNonce
  - out 
    - GnosisSafeProxy  
      - https://rinkeby.etherscan.io/address/0xCa59c83995260E2B9D917acB29B5510E7B3fEc3c#code 
  - source code 
    - https://rinkeby.etherscan.io/address/0xa6b71e26c5e0845f74c812102ca7114b6a896ab2#code
    - core code 
      ```
      /// @dev Allows to create new proxy contact using CREATE2 but it doesn't run the initializer.
      ///      This method is only meant as an utility to be called from other methods
      /// @param _singleton Address of singleton contract.
      /// @param initializer Payload for message call sent to new proxy contract.
      /// @param saltNonce Nonce that will be used to generate the salt to calculate the address of the new proxy contract.
      function deployProxyWithNonce(
          address _singleton,
          bytes memory initializer,
          uint256 saltNonce
      ) internal returns (GnosisSafeProxy proxy) {
          // If the initializer changes the proxy address should change too. Hashing the initializer data is cheaper than just concatinating it
          bytes32 salt = keccak256(abi.encodePacked(keccak256(initializer), saltNonce));
          bytes memory deploymentData = abi.encodePacked(type(GnosisSafeProxy).creationCode, uint256(uint160(_singleton)));
          // solhint-disable-next-line no-inline-assembly
          assembly {
              proxy := create2(0x0, add(0x20, deploymentData), mload(deploymentData), salt)
          }
          require(address(proxy) != address(0), "Create2 call failed");
      }

      /// @dev Allows to create new proxy contact and execute a message call to the new proxy within one transaction.
      /// @param _singleton Address of singleton contract.
      /// @param initializer Payload for message call sent to new proxy contract.
      /// @param saltNonce Nonce that will be used to generate the salt to calculate the address of the new proxy contract.
      function createProxyWithNonce(
          address _singleton,
          bytes memory initializer,
          uint256 saltNonce
      ) public returns (GnosisSafeProxy proxy) {
          proxy = deployProxyWithNonce(_singleton, initializer, saltNonce);
          if (initializer.length > 0)
              // solhint-disable-next-line no-inline-assembly
              assembly {
                  if eq(call(gas(), proxy, 0, add(initializer, 0x20), mload(initializer), 0, 0), 0) {
                      revert(0, 0)
                  }
              }
          emit ProxyCreation(proxy, _singleton);
      }
      ```
