1. At constructor, we are not checking if the voters array has voters. A malicious initial president may set 0 voters which would result in no votes when the time comes.
   A check should happen --> that would also check for 0 address voters

2. Function isArray() is looping through an array (UNBOUNDED FOR LOOP). If the voter is at the end of the array and gas ends before he's reached, tx will always revert..and he won't be able to rankCandidates() or vote
   Use a mapping to verify voter eg.bool

3. The orderedCandidates array isn't being hashed correctly, so the hash that the voter signs off-chain will not match the hash used for signature recovery on-chain. This will cause the signature verification (ECDSA.recover) to fail.
   Current Issue: abi.encode(TYPEHASH, orderedCandidates) will not properly handle the array encoding in the EIP-712 context.
   When encoding an array in EIP-712, you typically need to hash the array itself separately. EIP-712 does not directly support arrays in the abi.encode call. Instead, you need to hash the elements of the array and then include that in the final abi.encode.
   The correct implementation should hash the orderedCandidates array separately and then include that hash in the final struct hash:

4. Unrestricted Access: As it stands, anyone can call rankCandidatesBySig(). This could potentially allow malicious actors to submit votes using valid signatures, but not necessarily as intended. Without restrictions, any external party could invoke the function with valid signatures, potentially leading to abuse or unintended votes.

Restrict Access:
Ensure that rankCandidatesBySig can only be called by a trusted party or relayer. You could use an access control mechanism (e.g., Ownable from OpenZeppelin) to limit who can call this function, or implement logic to check that the caller is a known relayer or has proper authorization.

7. The `selectPresident()` function will always revert because `s_previousVoteEndTimeStamp` is never initialized in the constructor,thus the function will never work.
   Since `s_previousVoteEndTimeStamp` defaults to 0, the condition `block.timestamp - s_previousVoteEndTimeStamp <= i_presidentalDuration` will always evaluate to true during the first call, causing the function to revert.
   Winner cant ever be selected
   To fix this, ensure s_previousVoteEndTimeStamp is properly initialized in the constructor or via an appropriate initialization function.

8. The selectPresident function is crucial for determining the winning candidate in an election. However, there are significant issues related to the management of the s_candidateList that can lead to unexpected behavior during the election process.
   Lack of Proper Initialization:The s_candidateList is not initialized in the constructor, allowing voters to add candidates dynamically during the election process. This results in a situation where the ordered candidate list can contain duplicates, which violates the intended constraints.

9. Potential for Invalid Candidates:The absence of a controlled candidate list undermines the integrity of the voting mechanism. The current implementation does not enforce a limit on candidates in `s_candidateList`.
   Since voters can include any candidate, the list may contain candidates who should not be part of the election. This could lead to incorrect or skewed election outcomes, as the list may not accurately reflect the valid pool of candidates.

10. A voter can vote \*10 times for the same person, thus manipulating the election process
11. Ideally, the selectPresident function should return the address of the winning candidate. Currently, the function only updates internal state variables without providing feedback on the result of the election.
    To address this, you can modify the selectPresident function to return the address of the winner:
    // Return the winning address
    return s_currentPresident;

12. No natspecs!!
13. No events during important functions like rankCandidates(), selectPresident(),
14. We need to pass in the s_previousVoteEndTimeStamp at constructor. Check if it's passed <election period>. Revert if it has
    the require(block.timestamp< s_previousVoteEndTimeStamp + 1460days, "Cant rank, this specific election is over"),
    the check should be at function \_rankCandidates()

others::

1. // Time is not tracked at the time an initial president comes in at the constructor. If we dont add a current timestamp, time will eventually pass and a malicious president will make sure that they overstays his tenure
   TRACKED FROM THE TIME THE PRESIDENT IS SELECTED
   //
