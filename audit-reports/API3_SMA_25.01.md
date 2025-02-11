## **WEAKNESSES**

---

This section contains the list of discovered weaknesses.

# 1. MISSING UPPER LIMIT CHECKS

**SEVERITY**: Low

**PATH**: StakeableVestingFactory:deployStakeableVesting:L22-55

**REMEDIATION**: add constant variables with upper limits, which will be checked against the parameters in StakeableVestingFactory:deployStakeableVesting()

**STATUS**:

**DESCRIPTION**:

The function `deployStakeableVesting()` doesn't have an upper limit check for the `endTimestamp` variable.

**Code snippet**:

```
function deployStakeableVesting(
        address beneficiary,
        uint32 startTimestamp,
        uint32 endTimestamp,
        uint192 amount
    ) external override returns (address stakeableVesting) {
		[...]
}
```

---

# 2. ABUSE POSSSIBILITY

**SEVERITY**: Low

**PATH**: StakeableVesting:setBeneficiary():L105-109

**REMEDIATION**: add a check if \_beneficiary is not equal to owner

**STATUS**:

**DESCRIPTION**:

The Owner is able to set themselves as a beneficiary and do things on behalf of the real beneficiary. It is better to add a check that \_beneficiary isn’t equal to the owner at `setBeneficiary`.

**Code snippet**:

```
function setBeneficiary(address _beneficiary) external override onlyOwner {
        require(_beneficiary != address(0), "Beneficiary address zero");
        beneficiary = _beneficiary;
        emit SetBeneficiary(_beneficiary);
    }
```

---

# 3. FRONT-RUNNING VESTING CREATION

**SEVERITY**: Low

**PATH**: StakeableVestingFactory.sol:deployStakeableVesting:L39-71

**REMEDIATION**: use OpenZeppelin's Counter library for a unique number that is added to the salt in StakeableVestingFactory.sol:deployStakeableVesting and incremented afterwards

**STATUS**:

**DESCRIPTION**:

Deployment of a stakeable vesting contract can be front-ran because it only uses the function parameters as salt for `CREATE2`. Any user can copy the parameters from some other user's transaction and force their transaction to be processed first.
As a result, the transaction of the victim would revert and their vesting contract address 'stolen' (i.e. they would have to adjust their parameters).
The malicious actor can immediately call `StakeableVesting.sol:withdrawAsOwner` in the same transaction to retrieve their assets, effectively risking 0 assets.
The victim could be blocked from creating vesting contracts (albeit with a high gas cost to the attacker). Or, if the victim performed any actions on the precalculated address of the vesting contract, then that would fall into the hands of the attacker.

**Code snippet**:

```
function deployStakeableVesting(
    address beneficiary,
    uint32 startTimestamp,
    uint32 endTimestamp,
    uint192 amount
) external override returns (address stakeableVesting) {
    stakeableVesting = Clones.cloneDeterministic(
        stakeableVestingImplementation,
        keccak256(
            abi.encodePacked(
                beneficiary,
                startTimestamp,
                endTimestamp,
                amount
            )
        )
    );
    IERC20(api3Token).transferFrom(msg.sender, stakeableVesting, amount);
    IStakeableVesting(stakeableVesting).initialize(
        msg.sender,
        beneficiary,
        startTimestamp,
        endTimestamp,
        amount
    );
    emit DeployedStakeableVesting(
        msg.sender,
        beneficiary,
        startTimestamp,
        endTimestamp,
        amount
    );
}
```
