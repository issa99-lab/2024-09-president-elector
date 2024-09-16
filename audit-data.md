H-2

## Summary

The constructor does not check if the voters array is non-empty, allowing a malicious president to initialize the contract with no voters.

## Vulnerability Details

A malicious president can deploy the contract with an empty voters array, which would result in no votes being cast during voter rankings, effectively making the initial president permanent.

## Impact

This allows a malicious initial president to stay in power indefinitely, as no votes can occur if there are no voters.

## Tools Used

Manual code review

## Recommendations

Add a check in the constructor to ensure the voters array is not empty

```javascript
 constructor(address[] memory voters) EIP712("RankedChoice", "1") {
  @>+   require(voters.length > 0, "No voters added");
        VOTERS = voters;
        i_presidentalDuration = 1460 days;
        s_currentPresident = msg.sender;
        s_voteNumber = 0;
    }
```

H-3

## Summary

The `s_previousVoteEndTimeStamp` is never initialized in the constructor, causing the `selectPresident()` function to always revert.

## Vulnerability Details

Since `s_previousVoteEndTimeStamp` defaults to 0, the condition (block.timestamp - s_previousVoteEndTimeStamp <= i_presidentalDuration) at `selectPresident()` will always evaluate as true on the first call, leading to the function reverting with the error `RankedChoice__NotTimeToVote`()

## Impact

The president selection process cannot complete, preventing a new president from being selected.

Add this at `RankedChoice.sol`:

```javascript
+ function getPreviousEndTime() external view returns (uint256) {
  return s_previousVoteEndTimeStamp;
  }
```

POC. Add this at your test suits:

```javascript
function testSelectPresident() public {
      assert(rankedChoice.getCurrentPresident() != candidates[0]);

      orderedCandidates = [
          candidates[0],
          candidates[1],
          candidates[2],
          candidates[3]
      ];
      uint256 startingIndex = 0;
      uint256 endingIndex = 60;
      for (uint256 i = startingIndex; i < endingIndex; i++) {
          vm.prank(voters[i]);
          rankedChoice.rankCandidates(orderedCandidates);
      }

      startingIndex = endingIndex + 1; //61
      endingIndex = 100;
      orderedCandidates = [
          candidates[3],
          candidates[1],
          candidates[0],
          candidates[2]
      ];
      for (uint256 i = startingIndex; i < endingIndex; i++) {
          vm.prank(voters[i]);
          rankedChoice.rankCandidates(orderedCandidates);
      }

      vm.warp(block.timestamp + rankedChoice.getPreviousEndTime());

      rankedChoice.selectPresident();
      assertEq(rankedChoice.getCurrentPresident(), candidates[0]);
  }
```

## Tools Used

Foundry

## Recommendations

Initialize `s_previousVoteEndTimeStamp` in the constructor, with the previous timestamp that was set after the selection of a president.

```javascript
constructor(address[] memory voters, uint256 _previousVoteEndTimeStamp) EIP712("RankedChoice", "1") {
+  s_previousVoteEndTimeStamp = _previousVoteEndTimeStamp;
}
```

H-3

## Summary

The s_candidateList is dynamically modified based on voter input, allowing voters to add unauthorized candidates.

## Vulnerability Details

The current implementation allows voters to add candidates dynamically to the candidate list during the election process at `selectPresident()`, which leads to unauthorized or unintended candidates being added to the election.

## Impact

The voting process can be compromised by allowing voters to include candidates who should not be part of the election.

## Tools Used

Manual code review

## Recommendations

Predefine a list of valid candidates during the election setup (constructor) to ensure only authorized candidates are part of the selection process.

Remove: ln 94&94

RE-CHECK THIS!!!

## Summary

The function `isInArray` contains an unbounded for loop that can cause transactions to revert due to exceeding gas limits.

## Vulnerability Details

If the voters array is very large, the transaction may run out of gas before reaching the end of the loop, causing the function to revert and preventing the voter from casting their vote.

## Impact

This issue can prevent legitimate voters from casting their votes, especially when the voter is located at the end of the array.

## Tools Used

Manual code review + Foundry

## Recommendations

Instead of looping through the array, use a mapping for voter validation to ensure O(1) lookups.

- Set a max length of voters at constructor. Loop while filling the mapping

```javascript
+   mapping(address => bool) private isVoter;

```

H-4

## Summary

A voter can rank the same candidate multiple times, thus manipulating the voting process.

## Vulnerability Details

There is no validation to prevent a voter from ranking the same candidate in multiple positions in their ordered list, giving a single candidate an unfair advantage.

## Impact

Malicious voters can skew election results by ranking the same candidate multiple times, which undermines the fairness of the ranked-choice voting process.

## Tools Used

Foundry

## Recommendations

Add validation to ensure that each candidate is ranked only once per voter:

6
Lack of Feedback in selectPresident

## Summary

The selectPresident function does not return the address of the selected president.

## Vulnerability Details

Currently, the function only updates internal state variables without providing any external feedback on the selected winner, which can make the election process less transparent.

## Impact

Users cannot easily determine the election outcome without querying state variables.

## Tools Used

Manual review

## Recommendations

Modify the selectPresident function to return the address of the newly selected president:

```javascript
+  function selectPresident() external view returns (address){

+return s_currentPresident;
 }
```

7
Missing NatSpec Documentation

## Summary

The contract lacks NatSpec comments, which makes it difficult to understand the purpose and behavior of certain functions.

## Vulnerability Details

The absence of NatSpec comments increases the difficulty for auditors and users to fully understand the contract's functionality and assumptions.

## Impact

It could lead to misunderstandings or improper use of the contract.

## Tools Used

Manual Review

## Recommendations

Add NatSpec comments to all functions and variables, explaining their purpose, parameters, and return values.

8

## Summary

Key functions such as rankCandidates and selectPresident do not emit events, making it difficult to track important actions on-chain.

## Vulnerability Details

Without events, there is no way to easily track when a vote is cast or when a president is selected, which can hinder transparency.

## Impact

It becomes harder for external users or applications to monitor important changes in the contract.

## Tools Used

Manual code review

## Recommendations

Add events for important actions, such as votes and president selections:

11

## Summary

## Vulnerability Details

## Impact

## Tools Used

## Recommendations
