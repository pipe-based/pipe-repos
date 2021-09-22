# gnosis
## acitons
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
