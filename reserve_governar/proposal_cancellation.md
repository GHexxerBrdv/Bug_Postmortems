# [L] optimistic proposer can cancel the proposal after vetoed

## Description

The `ReserveOptimisticGovernor` automatically creates a standard(traditional) proposal with same perameters as the optimistic proposal after the veto threshold is reached for the optimistic proposal. However the `ReserveOptimisticGovernor` creates the standard proposal with the same optimistic proposer as the proposal creator.

```solidity
ProposalData memory proposalData = ProposalData(
    newProposalId,
    governor.proposalProposer(proposalId),
    optimisticProposal.targets,
    optimisticProposal.values,
    optimisticProposal.calldatas,
    newDescription
);

_saveProposal(proposalData, proposalCores[newProposalId], governor.votingDelay(), governor.votingPeriod());
```

However after transitioning to the standard proposal, the optimistic proposer can still cancel the proposal. since the cancle function requires the proposal creator to cancel the proposal (also admin can cancel the proposal). This allows proposal cancellation of standard proposal which is vetoed as optimistic proposal.