# [M] Veto threshold can never be reached

## Description

In `ReserveOptimisticGovernor` the the veto threshold is calculated as following:

```solidity
uint256 _vetoThreshold = vetoThreshold(proposalId);

if (_vetoThreshold == ProposalLib.TRANSITIONED_VETO_THRESHOLD) {
    // special-case for transitioned proposals
    return ProposalState.Defeated;
}

// {tok}
uint256 pastSupply = token().getPastTotalSupply(snapshot); //>/ get past total supply at vote starting timestamp

if (pastSupply == 0) {
    return ProposalState.Canceled;
}

// {tok} = D18{1} * {tok} / D18{1}

@> uint256 vetoThresholdTok = (_vetoThreshold * pastSupply) / 1e18;
```

The catch here is that the protocol supports two types of proposals:

1. optimistic proposal
2. traditional proposal

there is different share accounting for optimistic shares vs traditional shares. To vote on optimistic proposals user must have optimistic shares, user must have to delegate their shares to optimistic shares. However delegating shares to optimistic shares are complately optional.

As we can see in the code above it is using `pastSupply` which is the total supply at the time the proposal started. However it reads the total supply of shares which are ever minted. means it is not considering total supply of optimistic shares, instead it is considering the trditional shares. Since optimistic shares are completely optional, there is a possibality that user do not delegate their shares to optimistic shares. However only the optimistic shares holder can vote on optimistic proposals.

This will result in never reaching veto threshold proposal. Which means the proposal will never be defeated in such cases.

## Root Cause

> [!NOTE]
> The protocol was considering total supply of shares that is ever minted to determine the veto threshold for optimistic proposals, However there is another accounting mechanism for the optimistic shares. instead of using total supply of optimistic shares, it is using the total supply of shares that are ever minted (represents the standard shares used to vote on standerd proposals). Which can be way more then total supply of optimistic shares (because optimistic delegation is completely optional).