# Demurrage Mechanism in Circles: Developer's Guide

## Overview

Circles implements demurrage as a systematic reduction of token balances over time, applied to all token balances in the system, affecting both personal and group currencies. The system extends the ERC1155 multi-token standard to incorporate this demurrage mechanism.

Additionally, Circles allows wrapping ERC1155 tokens (in a demurrage representation) into an ERC20 contract. For each Circles identifier (personal or group), two ERC20 contracts can be created: one for demurrage representation and one for inflationary representation.

### Key Features of Demurrage in Circles

1. **Annual Rate**: 7% reduction in token balances per year
2. **Daily Application**: Calculated and applied on a per-day basis
3. **Universal**: Affects all Circles tokens equally

## ERC1155 Interface and Demurrage

Circles extends the standard ERC1155 interface to handle demurrage. Key functions include:

### Balance Queries

```js
function balanceOf(address account, uint256 id) public view returns (uint256)
```
Returns the current demurraged balance of `account` for token `id`. This function internally calls `balanceOfOnDay` with the current day.

```js
function balanceOfOnDay(address account, uint256 id, uint64 day)
    public view returns (uint256 balance, uint256 discountCost)
```
Returns the demurraged balance for `account` and `id` as of the specified `day`, along with the `discountCost` (amount of tokens "burned" due to demurrage).

### Transfers

```js
function safeTransferFrom(address from, address to, uint256 id, uint256 value, bytes memory data) public
```
Transfers `value` amount of token `id` from `from` to `to`, applying demurrage before transfer to both sender and receiver if applicable.

### Token ID Convention

Token IDs in Circles are derived from addresses:
```js
function toTokenId(address _avatar) public pure returns (uint256) {
    return uint256(uint160(_avatar));
}
```

## Demurrage Calculation

### Balance Reduction

The system reduces token balances daily by a factor calculated as:
```sh
Daily Factor = (1 - 0.07)^(1/365.25) ≈ 0.99980813
```
This results in a compound 7% reduction over a full year.

### Practical Effects

- A balance of 100 Circles reduces to approximately 99.98081 Circles after one day
- Over 365 days, 100 Circles reduces to about 93 Circles

## Implementation Considerations

1. **On-Demand Calculation**: Balance reductions are computed when balances are accessed (through view functions) or computed and updated when modified (through transfers).
2. **Token Burning**: The reduction in balance burns the amount that gets demurraged, effectively removing these tokens from circulation.
3. **Gas Efficiency**: On-demand calculation avoids the need for daily update transactions.
4. **Precision**: 128-bit fixed-point arithmetic ensures high accuracy in calculations, allowing for daily updates.

## Development Guidelines

1. Always use `balanceOf()` or `balanceOfOnDay()` to get current, demurraged balances.
2. On the hub contract, all amounts are to be understood as demurraged amounts on the current day.
3. Be aware that token balances decrease over time, even without explicit transfers or actions.
3. If your application expects static (ie. inflationary) balances or to interface with external systems:
    - Use the provided `InflationaryOperator` contract to interact with ERC1155 Circles.
    - Alternatively, wrap the (group) Circles you want to interact with in an inflationary ERC20 contract, where you can treat them as regular ERC20 tokens.

## Time and Day Calculation in Circles

Understanding how Circles handles time is crucial for working with demurraged balances. The system uses a concept of "days" since a fixed starting point to calculate demurrage.

### The `day` Function

```js
function day(uint256 timestamp) internal view returns (uint64) {
    return uint64((timestamp - inflationDayZero) / 1 days);
}
```

This function converts a Unix timestamp to a day number used in demurrage calculations. Key points:

1. **Inflation Day Zero**: This is the starting point for all time calculations in Circles. It's set to October 15, 2020, at 00:00:00 UTC (Unix timestamp: 1602720000).

2. **Day Calculation**: Days are calculated as the number of whole days passed since Inflation Day Zero.

3. **Usage**: This day number is used in all demurrage calculations to determine how much a balance has decreased over time.

### Demurrage vs Inflationary Balances

- **Demurrage Balances**: These are time-dependent. The actual balance at any given moment is a function of the stored balance, the last update day, and the current day.
- **Inflationary Balances**: These are static in time. They represent an equivalent way of representing the 7% p.a. "inflation", but through an ever-increasing money supply. In the inflationary representation, Circles mints slightly more tokens per hour each day than the previous day, to offset the existing Circles already in circulation.

Note that the ground-truth for Circles is the demurrage representation, where exactly one Circle is minted per person per hour, offset by the daily burning of Circles at a rate equivalent to 7% p.a.

### Practical Implications

1. When querying balances, the current day is always used to calculate the up-to-date demurraged balance.

2. For future projections, you can specify a different day using `balanceOfOnDay()`.

3. When converting between demurrage and inflationary representations, the day parameter determines the point in time for the conversion.

## Helper Functions for Inflationary Conversion

Circles provides two helper functions to convert between demurraged and inflationary representations:

```js
function convertInflationaryToDemurrageValue(uint256 _amount, uint64 _day)
    public view returns (uint256)
```
Converts an inflationary amount to its demurraged equivalent as of the specified day.

```js
function convertDemurrageToInflationaryValue(uint256 _amount, uint64 _dayUpdated)
    public view returns (uint256)
```
Converts a demurraged amount as on the day provided to its inflationary equivalent.

## Conclusion

Understanding the demurrage mechanism is crucial for developers working with Circles. Always consider the time-dependent nature of token balances and use the provided functions to ensure accurate balance calculations and transfers.