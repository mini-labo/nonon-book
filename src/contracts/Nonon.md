# Nonon.sol

Nonon.sol is powered by the [scatter.art](https://scatter.art) Archetype base, built on top of the ERC721A standard.

As a token contract, Nonon.sol largely sticks to the standard established by Archtype.

However, The nonon contract has a special addition that leverages ERC721A's `_beforeTokenTransfers` function to
enable tracking of nonon movement across wallets.

```solidity
function _beforeTokenTransfers(address from, address to, uint256 startTokenId, uint256 quantity)
    internal
    override
{
    IFriendCard friendCard = IFriendCard(friendCardAddress);

    if (to != address(0) && !friendCard.hasToken(to)) {
        friendCard.mintTo(to);
    }

    friendCard.registerTokenMovement(from, to, startTokenId, quantity);
}
```

This function is run as part of both ERC721A's `_mint` and `transferFrom` functions, allowing this single override
to catch all token transfer cases.
The `registerTokenMovement` function in the friend card contract update received and sent bitmaps that contain
token transfer records for a given address. Since we enforce one token per address for the soulbound friend card, we
can use the wallet address as an equivalent to a token. This has the positive side effect of allowing a user to preserve their progress
if they burn their soulbound friendship card and later are granted another one when they want to engage with the nonon collection again.

```solidity
// add ID for associated sequential tokens to appropriate lists
function registerTokenMovement(address from, address to, uint256 collectionTokenStartId, uint256 quantity)
    external
    onlyCollection
{
    if (from != address(0)) {
        if (to != from) {
            sentBitmap[from].setBatch(collectionTokenStartId, quantity);
        }
    }

    if (to != address(0)) {
        receivedBitmap[to].setBatch(collectionTokenStartId, quantity);
    }

    emit BatchMetadataUpdate(1, type(uint256).max);
}
```

While there is a tradeoff in the form of marginal gas cost increase per transfer in the Nonon token collection,
this mechanism allows the friend point mechanism to be added to an ERC721A with only a single function override and extra storage variable.
