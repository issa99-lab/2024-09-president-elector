H-1
Zero Voters on Initialization

## Summary

The constructor does not check if the voters array is non-empty or has zero addresses, allowing a malicious president to initialize the contract with no voters.

## Vulnerability Details

Zero Address Inclusion:
The constructor does not check if any of the voters or candidates in the lists are the zero address (0x0). Allowing 0x0 addresses in the election process can lead to unintended behavior, such as invalid votes or corrupted election rounds, since 0x0 is not a valid address for voting.

Empty Voter List:
A malicious president can deploy the contract with an empty voters array, which would result in no votes being cast during voter rankings, effectively making the initial president permanent.
If an empty voter list is passed, it can lead to undefined behavior, such as the voting process not functioning properly or the recursion terminating prematurely.

## Impact

Protocol Instability: The voting process is central to the protocol. If the list of voters is empty at initialization, it could prevent the election from occurring at all. This halts the core functionality of the protocol.

Invalid Votes: Allowing 0x0 addresses could enable invalid votes to be tallied, which would compromise the fairness of the election process.

## Tools Used

Manual code review

## Recommendations

Add a check in the constructor to ensure the voters array is not empty

```javascript
 constructor(address[] memory voters) EIP712("RankedChoice", "1") {
   @>++  uint256 votersLength = voters.length;
 + require(votersLength  > 0, "No voters added");
  + for (i=0; i<votersLength ; i++){
 + require(voterList[i] != address(0), "Voter cannot be zero address");
  }
        VOTERS = voters;
        i_presidentalDuration = 1460 days;
        s_currentPresident = msg.sender;
        s_voteNumber = 0;
    }
```

H-2
No Time Control at Initialization

## Summary

The `s_previousVoteEndTimeStamp` is never initialized in the constructor, causing the `selectPresident()` function to always revert.

## Vulnerability Details

Without initializing `s_previousVoteEndTimeStamp`, it defaults to 0. As a result, the condition in `selectPresident()` that checks if enough time has passed since the last vote ln 76-79:

```javascript
require(block.timestamp - s_previousVoteEndTimeStamp <=
  i_presidentalDuration, "RankedChoice__NotTimeToVote");
```

will always evaluate to true on the first call, leading to the function reverting with the `RankedChoice__NotTimeToVote` error, since the `previousVoteEndTimeStamp` isn't initialized thus empty.

## Impact

This bug prevents the president selection process from being completed successfully, as the contract will always revert on the first attempt to select a president. As a result, no new president can be chosen, disrupting the election process.

## POC

Add this at your test suits:

```Solidity
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

      vm.warp(block.timestamp + rankedChoice.getDuration());

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
Dynamic Candidate List in `selectPresident()`

## Summary

The `s_candidateList` is dynamically modified based on voter input, allowing voters to add unauthorized candidates.

## Vulnerability Details

The current implementation allows voters to add candidates dynamically to the candidate list during the election process at `selectPresident()`, which leads to unauthorized or unintended candidates being added to the election.

## Impact

The voting process can be compromised by allowing voters to include candidates who should not be part of the election.

## Tools Used

Manual code review

## Recommendations

Predefine a list of valid candidates during the election setup (constructor) to ensure only authorized candidates are part of the selection process.

Remove: ln 94&94
Add specific candidate list at constructor with checks on zero addresses and duplicates

H-4
Potential for Voter Manipulation

## Summary

A voter can rank the same candidate multiple times, thus manipulating the voting process.

## Vulnerability Details

There is no validation to prevent a voter from ranking the same candidate in multiple positions in their ordered list, giving a single candidate an unfair advantage.

## Impact

Malicious voters can skew election results by ranking the same candidate multiple times, which undermines the fairness of the ranked-choice voting process.

## Tools Used

Foundry

## Recommendations

To ensure that each candidate is ranked only once per voter in the `_rankCandidates` function, we can add a check to see if any candidate is duplicated within the `orderedCandidates` array. This can be done by using a temporary `mapping(address => bool)` inside the function to track if a candidate has already been ranked by the voter.

```Javascript
function _rankCandidates(
        address[] memory orderedCandidates,
        address voter
    ) internal {
        // Checks
        if (orderedCandidates.length > MAX_CANDIDATES) {
            revert RankedChoice__InvalidInput();
        }
        if (!_isInArray(VOTERS, voter)) {
            revert RankedChoice__InvalidVoter();
        }

@>++        // Temporary mapping to track if a candidate has already been ranked
+       mapping(address => bool) memory rankedAlready;

+       // Ensure all ranked candidates are valid and not duplicated
+      for (uint256 i = 0; i < orderedCandidates.length; i++) {
+           address candidate = orderedCandidates[i];

+          // Check if the candidate is valid
+          if (!s_isValidCandidate[candidate]) {
+              revert RankedChoice__InvalidCandidate();
+          }

+       // Check if this candidate has already been ranked by the voter
+     if (rankedAlready[candidate]) {
+         revert RankedChoice__DuplicateRanking();  // Revert if duplicate ranking
+      }

+        // Mark candidate as ranked
+       rankedAlready[candidate] = true;
+       }

        // Internal Effects
        s_rankings[voter][s_voteNumber] = orderedCandidates;
    }

```

## Summary

The function `isInArray` when used at `_rankCandidates()`, function contains an unbounded for loop that can cause transactions to revert due to exceeding gas limits .
<https://github.com/Cyfrin/2024-09-president-elector/blob/fccb8e2b6a32404b4664fa001faa334f258b4947/src/RankedChoice.sol#L167>

## Vulnerability Details

The current implementation of `_isInArray` iterates over the entire array of voters to check if a given address is in the array. If the array size is very large (since there is no limit set for the number of voters), the loop may exceed the gas limits, causing the transaction to revert. This prevents the last voters from casting their votes and could disrupt the voting process.

## Impact

Transaction Failure: Transactions can fail if the loop runs out of gas, particularly affecting voters located towards the end of the array.
Disruption in Voting Process: Valid voters may be unable to cast their votes if the function reverts due to gas limits.

## Tools Used

Manual code review + Foundry.

## POC

If we had 30,000 voters, the transaction will revert at set Up.
Add this to your test suite:

```Solidity
contract RankedChoiceTest is Test {
    address[] voters;
    address[] candidates;

    uint256 constant MAX_VOTERS = 30000;
    uint256 constant MAX_CANDIDATES = 4;
    uint256 constant VOTERS_ADDRESS_MODIFIER = 100;
    uint256 constant CANDIDATES_ADDRESS_MODIFIER = 200;

    RankedChoice rankedChoice;

    address[] orderedCandidates;

    function setUp() public {
        for (uint256 i = 0; i < MAX_VOTERS; i++) {
            voters.push(address(uint160(i + VOTERS_ADDRESS_MODIFIER)));
        }
        rankedChoice = new RankedChoice(voters);

        for (uint256 i = 0; i < MAX_CANDIDATES; i++) {
            candidates.push(address(uint160(i + CANDIDATES_ADDRESS_MODIFIER)));
        }
    }
```

## Recommendations

Use a Mapping: Replace the array with a mapping for voter validation, which provides O(1) lookup time and avoids gas issues associated with large arrays.

Set Maximum Limits: Implement a maximum length for the voter array and check this limit in the constructor to prevent excessive sizes.

Update Code: Modify the constructor and `_rankCandidates()` function to use a mapping instead of an array for voter validation. Here’s an example:

```javascript
+   mapping(address => bool) private isVoter;
+ uint256 private constant MAX_VOTERS = 500;


 constructor(address[] memory voters) EIP712("RankedChoice", "1") {
   @>++  uint256 votersLength = voters.length;
 + require(votersLength  > 0 && votersLength  < = MAX_VOTERS , "Invalid voter number");
+  for (i=0; i<votersLength ; i++){
+  require(voterList[i] != address(0), "Voter cannot be zero address");
+  isVoter[voterList[i]] = true;
  }}

 function _rankCandidates(
        address[] memory orderedCandidates,
        address voter
    ) internal {
        // Checks
        if (orderedCandidates.length > MAX_CANDIDATES) {
            revert RankedChoice__InvalidInput();
        }
 --       if (!_isInArray(VOTERS, voter)) {
 ++         if(!isVoter[voter]){
            revert RankedChoice__InvalidVoter();

 } ";
        }
        s_rankings[voter][s_voteNumber] = orderedCandidates;
    }

```

````

M-2
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

Modify the `selectPresident()` function to return the address of the newly selected president:

```javascript
+ln75  function selectPresident() external view returns (address){

+ln113 return s_currentPresident;
 }
````

L-1
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

L-2
Lack of Events for Key Actions

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
