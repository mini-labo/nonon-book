# NononFriendCard.sol

NononFriendCard.sol is a contract that dynamically constructs metadata based on stored user activity, and restricts
certain transfers in order to create a "soulbound" token.

### Image construction

On chain image composition is notoriously restricted due to the 24kb size limit of ethereum smart contracts, which
has historically limited the fidelity of images that can be saved directly on chain. However, off chain decentralized solutions such as IPFS
limit the ability to make NFTs that are both dynamic and decentralized, due to the need to save static content addressing hashes.

The nonon friend card leverages [`SSTORE2`](https://github.com/0xsequence/sstore2) to store the static portions of a larger svg image,
while injecting portions of an svg dynamically during the runtime of a call to the token contract's `tokenURI` method. 

In practice, this looks like below:

```solidity
function tokenURI(uint256 tokenId) public view virtual override returns (string memory) {
    if (!_exists(tokenId)) revert URIQueryForNonexistentToken();

    uint256 tokenPoints = points(tokenId);
    LevelImageData memory level = levelData(tokenPoints);
    string memory message = tokenMessage(tokenId);

    string memory baseUrl = "data:application/json;base64,";
    return string(
        abi.encodePacked(
            baseUrl,
            Base64.encode(
                bytes(
                    abi.encodePacked(
                        '{"name":"',
                        string.concat(TOKEN_NAME, level.suffix),
                        '",',
                        '"description":"',
                        message,
                        '",',
                        '"attributes":[{"trait_type":"points","max_value":',
                        _toString(level.cap),
                        ',"value":',
                        _toString(tokenPoints),
                        "}],",
                        '"image":"',
                        buildSvg(level.colorGradient, level.spriteIndex, level.spriteLength, message),
                        '"}'
                    )
                )
            )
        )
    );
}
```

a JSON object containing the token's metadata is dynamically constructed here based on the points data for the address
associated with the tokenId. The image is composed as a SVG string in below the `buildSvg()` function.

```solidity
// construct image svg
function buildSvg(string memory colorHex, uint16 spriteIndex, uint16 spriteLength, string memory message)
    internal
    view
    returns (string memory)
{
    string memory baseUrl = "data:image/svg+xml;base64,";
    bytes memory baseSvg = SSTORE2.read(baseSvgPointer);
    bytes memory spritesSvg = SSTORE2.read(spritesSvgPointer);
    bytes memory defs = SSTORE2.read(defsSvgPointer);

    return string(
        abi.encodePacked(
            baseUrl,
            Base64.encode(
                bytes(
                    abi.encodePacked(
                        '<svg xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 1080 1080"><path fill="rgba(255,255,255,0)" d="M0 0h1080v1080H0z" />',
                        '<path fill="url(#',
                        colorHex,
                        ')" d="M24 40a16 16 0 0 1 16-16h1000a16 16 0 0 1 16 16v914a16 16 0 0 1-16 16H114.5a24 24 0 0 0-17.6 7.7l-59 63.4a8 8 0 0 1-13.9-5.4V40Z" />',
                        baseSvg,
                        getSpriteSubstring(spritesSvg, spriteIndex, spriteLength),
                        '<text xml:space="preserve" fill="#009DF5" font-family="Courier" font-size="24" letter-spacing="0em" style="white-space:pre"><tspan x="144" y="1044.9">',
                        message,
                        "</tspan></text>",
                        defs,
                        "</svg>"
                    )
                )
            )
        )
    );
}
```

The svg image components that eventually comprise the nonon friend card image are stored in different contract addresses using `SSTORE2` as bytecode,
and that bytecode is read during runtime. The level that corresponds to a users total points contains information used to construct the image:


```solidity
struct Level {
  uint256 minimum;
  string name;
  string colorGradient;
  uint16 spriteIndex;
  uint16 spriteLength;
}
```

Each `Level` contains a `colorGradient` which is a simple ref to a more complex svg element stored statically and referenced during construction.
The `spriteIndex` and `spriteLength` contain references to the byte index and length of a given svg sprite in the set of sprites, which are all stored together
in a blob of data in a single `SSTORE2` write.

### Soulbinding

The mechanism for restricting transfers is very simple, leveraging the same `_beforeTokenTransfers` function from the ERC721A standard
used in [Nonon.sol](./Nonon.md):

```solidity
// prevent transfer (except mint and burn)
function _beforeTokenTransfers(address from, address to, uint256, uint256) internal pure override {
    if (from != address(0) && to != address(0)) {
        revert OnlyForYou();
    }
}
```

In accordance with ERC-5192, `event Locked(uint256 tokenId);` is also emitted on mint.

