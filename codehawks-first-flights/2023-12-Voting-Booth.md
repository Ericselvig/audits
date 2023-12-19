# Voting Booth
The code under review can be found in [2023-12-Voting-Booth](https://github.com/Cyfrin/2023-12-Voting-Booth/tree/main).

## Findings Summary
| ID | Description | Severity |
| :-: | - | :-: |
| [H-01](#h-1-in-votingbooth_distributerewards-rewardpervoter-is-miscalculated-causing-eth-to-be-stuck-and-voters-to-receive-less-reward-than-expected) | In `VotingBooth::_distributeRewards`, `rewardPerVoter` is miscalculated, causing ETH to be stuck and voters to receive less reward than expected | High |

## [H-1] In VotingBooth::_distributeRewards, rewardPerVoter is miscalculated, causing ETH to be stuck and voters to receive less reward than expected.

### Description
After the quorum has been reached, if the `for` votes are greater than the `against` votes, the ETH reward must be distributed among the `for` voters, but in `VootingBooth::_distributeRewards` the `rewardPerVoter` calculation wrongly divides the total reward by all voters instead of just `for` voters, causing ETH to be stuck in the contract forever.<br>
Additionally, the `for` voters will receive less reward.

```javascript
    function _distributeRewards() private {
        // get number of voters for & against
        uint256 totalVotesFor = s_votersFor.length;
        uint256 totalVotesAgainst = s_votersAgainst.length;
        uint256 totalVotes = totalVotesFor + totalVotesAgainst;

        // rewards to distribute or refund. This is guaranteed to be
        // greater or equal to the minimum funding amount by a check
        // in the constructor, and there is intentionally by design
        // no way to decrease or increase this amount. Any findings
        // related to not being able to increase/decrease the total
        // reward amount are invalid
        uint256 totalRewards = address(this).balance;

        // if the proposal was defeated refund reward back to the creator
        // for the proposal to be successful it must have had more `For` votes
        // than `Against` votes
        if (totalVotesAgainst >= totalVotesFor) {
            // proposal creator is trusted to create a proposal from an address
            // that can receive ETH. See comment before declaration of `s_creator`
            _sendEth(s_creator, totalRewards);
        }
        // otherwise the proposal passed so distribute rewards to the `For` voters
        else {
@>          uint256 rewardPerVoter = totalRewards / totalVotes;

            for (uint256 i; i < totalVotesFor; ++i) {
                // proposal creator is trusted when creating allowed list of voters,
                // findings related to gas griefing attacks or sending eth
                // to an address reverting thereby stopping the reward payouts are
                // invalid. Yes pull is to be preferred to push but this
                // has not been implemented in this simplified version to
                // reduce complexity & help you focus on finding the
                // harder to find bug

                // if at the last voter round up to avoid leaving dust; this means that
                // the last voter can get 1 wei more than the rest - this is not
                // a valid finding, it is simply how we deal with imperfect division
                if (i == totalVotesFor - 1) {
                    rewardPerVoter = Math.mulDiv(totalRewards, 1, totalVotes, Math.Rounding.Ceil);
                }
                _sendEth(s_votersFor[i], rewardPerVoter);
            }
        }
    }
```

### Impact
The ETH reward distributed among the `for` voters will be less than expected, and ETH will be stuck in the contract forever.

### Proof of Concept
1. Create a proposal with a reward of 10 ETH and 5 voters.
2. Voter 1 votes `for` the proposal.
3. Voter 2 votes `for` the proposal.
4. Voter 3 votes `against` the proposal.
5. Voters 1 and 2 should each receive 5 ETH, but they receive less.
6. The voting booth contract should have 0 ETH, but this is not the case as proven in the below PoC

<details>
<summary>PoC</summary>
Add the following to `VotingBoothTest.t.sol` file:

```javascript
    function testAllEthIsTransferedFromBooth() public {
        // first voter votes `for`
        vm.prank(address(0x1));
        booth.vote(true);

        // second voter votes `for`
        vm.prank(address(0x2));
        booth.vote(true);

        // third voter votes `against`
        vm.prank(address(0x3));
        booth.vote(false);
        
        console.log("booth balance: ", address(booth).balance); // 3333333333333333333
        console.log("voter 1 balance: ", address(0x1).balance); // 3333333333333333333
        console.log("voter 2 balance: ", address(0x2).balance); // 3333333333333333334
    }
```
</details>

### Recommendation
The easiest way to resolve this issue would be to calculate the `rewardPerVoter` as `totalRewards / totalVotesFor` instead of `totalRewards / totalVotes`.

```diff
-   uint256 rewardPerVoter = totalRewards / totalVotes;
+   uint256 rewardPerVoter = totalRewards / totalVotesFor;
```
