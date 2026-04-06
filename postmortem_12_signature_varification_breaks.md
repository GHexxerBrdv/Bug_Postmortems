Welcome to another postmortem. in this postmortem i will explain how incorrect assumption can lead to unexpected validation bypass in signature validation.

This bus is found in recent public audit contest on Code4Rena, called intution. However this seems very informative bug ot easy one. but honestly speaking when the code base is complex and big this type of simple and easy bug might be overlooked and could be missed.

To understand the bug first let's see following function

```solidity
function _extractValidUntilAndValidAfterFromSignature(bytes calldata signature)
        internal
        pure
        returns (uint48 validUntil, uint48 validAfter, bytes memory rawSignature)
    {
        uint256 signatureLength = signature.length;
        if (signatureLength == 65) {
            return (0, 0, signature);
        }
        //>/ this check ensures the signature should be of 65 length or 77 length. otherwise revert
        if (signatureLength != 77) {
            revert AtomWallet_InvalidSignatureLength(signatureLength);
        }

        uint256 metaOffset = signatureLength - 12;
        rawSignature = signature[:metaOffset]; //>/ extract the first 65 bytes of signature as raw signature

        bytes memory meta = signature[metaOffset:]; //>/ extract the last metadata bytes from the signature
        uint256 word;
        assembly {
            word := mload(add(meta, 32)) //>/ load the last metadata part as 32 bytes in memory
        }
        uint96 packed = uint96(word >> 160); //>/ shift right by 160 bits (20 bytes = 32 - 12) to get 12 bytes metadata
        validUntil = uint48(packed >> 48); //>/ take first 48 bits as cat it
        validAfter = uint48(packed); //>/ take last 48 bits as validAfter
    }
```

for your insights, the protocol has hardcoded the signature size. it could be either 65 bytes or 77 bytes long. otherwise the function revert. Above function is used to extract the validUntil and ValitAfter timestamp from the signature.

However the above function is called by another function which was performing the signature validation. as shown below.

```solidity
function _validateSignature(
        PackedUserOperation calldata userOp,
        bytes32 userOpHash
    )
        internal
        virtual
        override
        returns (uint256 validationData)
    {
        (uint48 validUntil, uint48 validAfter, bytes memory signature) =
            _extractValidUntilAndValidAfterFromSignature(userOp.signature);

        bytes32 hash = keccak256(abi.encodePacked("\x19Ethereum Signed Message:\n32", userOpHash)); // @audit-qa use oppenzeppelin MessageHashUtils instead

        (address recovered, ECDSA.RecoverError recoverError, bytes32 errorArg) = ECDSA.tryRecover(hash, signature);
        //>/ What if NoError has returned as Error?
        if (recoverError == ECDSA.RecoverError.InvalidSignatureLength) {
            revert AtomWallet_InvalidSignatureLength(uint256(errorArg));
        } else if (recoverError == ECDSA.RecoverError.InvalidSignatureS) {
            revert AtomWallet_InvalidSignatureS(errorArg);
        } else if (recoverError == ECDSA.RecoverError.InvalidSignature) {
            return _packValidationData(true, validUntil, validAfter);
        }

        bool sigFailed = recovered != owner();
        return _packValidationData(sigFailed, validUntil, validAfter);
    }
```

The catch here is that the above function is not validating the ValidUntil and ValidAfter timestamp. That means anyone can create a signature where these two timestamps can be anything and the validation just pass as long as the 65 bytes signature derive the right owner as the caller. 

This is the bug here. which is very simple by in large code base where there are multiple files to look out and thing out of the box, anyone can overlook this part and miss the bug. However,

> you always need to remeber that what are the assumption this function is making and what must be doen by the function, and what are thisngs that the function is not doing correctly. there is where bugs live.