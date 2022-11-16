---
title: Ricardian NFT Royalty
description: Ricardian contract extension to ERC-721
repo link: https://github.com/factoria-org/ricardian
author: skogard <skogard@protonmail.com>
status: Draft
type: Standards Track
category: ERC
created: 2022-11-11
---

## Abstract

An extension to the existing EIP-2981 NFT Royalty Standard that's aimed at providing a legal framework for NFT creators, in order to enforce their intellectual property rights properly, even when the royalty standard is not enforceable on-chain.


## Motivation

The EIP-2981 NFT Royalty Standard is NOT enforceable because it is "just a convention". As a result, several NFT marketplaces have started bypassing the standard. Some may even argue that this is not much different from stealing intellectual properties, similar to:

- Musicians sampling other musician's songs and not giving them royalty.
- Toy companies making money selling Disney character toys, WITHOUT paying Disney any royalty.

However, legally speaking this is NOT true, and technically none of this is illegal because the creators have never explicitly included a license in their NFT contract, because the EIP-2981 standard does not include any license document and is merely a "convention". 

This is analagous to an open source project publishing its code without a license. Any open source project that is published without a license document is regarded as public domain, and the publisher cannot easily claim to have rights to the project.

Likewise, it is no surprise NFT contracts without a proper license are considered "public domain", and anyone (such as NFT marketplaces) is free to do whatever they want with them. Therefore, disrespecting royalties is technically not illegal. And this spreads virally throughout the industry because all NFT marketplaces must think about the following problem:

- Other NFT marketplaces are doing it, and if they don't join, they will lose market share. Might as well join it instead of coming out as a loser in the wild wild west.
- Even the NFT marketplaces that WANT to support NFT creators' rights have to come up with a far-from-optimal solution because they can no longer rely on benevolence of the others. They must assume that intellectual properties without a proper license will not be enforced.



## Specification

> The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

Imagine EIP-2981 NFT royalty standard, but legally enforceable.

This proposal simply adds a single additional interface to the existing EIP-2981 to turn ANY NFT contract into a Ricardian contract. Basically, we introduce an additional method interface:

```
function licenseURI(uint tokenId) external view returns (string memory uri) {
  return "ipfs://..";
}
```

The URI string returned by `licenseURI()` may point to a license document that's legally binding, or may point to a document that simply describes a desired social contract expected by the asset holders and traders. Something along the lines of: "By owning this token, you agree that you will pay the royalty amount specified by the royaltyInfo() contract method for this tokenId."


### Examples

While it is possible to use mutable URIs such as `https://`, it is recommended that the `licenseURI()` returns either:

1. a content addressable URI such as GIT or IPFS URI, so that the creator cannot arbitrarily update them without leaving an on-chain evidence trail, but this is optional.
2. or on-chain rendered URI, programmatically generated in a view function (similar to how some NFT projects use on-chain generated SVG logic to implement tokenURIs)


#### 1. Content addressable URI

Content addressable URI may be any URI that cannot be changed, such as git URIs and IPFS URIs.

```
function licenseURI(uint tokenId) external view returns (string memory uri) {
  return "ipfs://..";
}
```


#### 2. Onchain URI

Onchain URIs may be generated programmatically by the smart contract logic. This is entirely left up to each contract implementation. For example:

```
import "@openzeppelin/contracts/utils/Base64.sol";
function licenseString(uint tokenId) internal view returns (string memory uri) {
  // some onchain logic taking the tokenId and deriving a license text and returns it.
}
function licenseURI(uint tokenId) external view returns (string memory uri) {
  return abi.encodePacked("data:application/json;base64,", Base64.encode(bytes(licenseString(tokenId))));
}
```


## Rationale

### Open source licensing model analogy

This scheme is better than blacklist driven approaches proposed by various NFT marketplaces because it takees the opposite approach.

Instead of each NFT contract having to follow some rules proposed by 3rd party marketplaces (or all kinds of future applications that deal with NFTs), the OWNER of the contract gets to decalre the license. If you think about it, this is exactly how it should work.

Using the open source project analogy, the blacklist based royalty enforcement scheme is similar to:

1. Open source projects don't have license documents
2. The projects are treated differently depending on which git hosting service they are hosted (such as GitHub, GitLab, etc.)

This model not only discourages the intellectual property creators to share their work, but is NOT at all scalable if we assume that there will be all kinds of third party applications and contracts that interact with NFTs in the future in many different ways.


### 3rd Party Implementation

From the contract owner's point of view, all they need to do is add just one additional method `licenseURI()`.

From the point of view of a 3rd party application (such as NFT marketplaces), they may take advantage of the onchain `licenseURI()` method from each NFT contract to automatically decide whether to list the NFTs on their platform or not.

In the following marketplace contract example, the `isListed()` method calls an NFT contract's `licenseURI()` to determine whether the license model of the said NFT contract is compatible with the marketplace's policy:

```
interface IRicardian {
  function licenseURI(uint256 tokenId) external view returns (string memory);
}
function isListed(
  addres collection,
  uint tokenId
) pubilc view returns (bool) {
  string memory license = IRicardian(collection).licenseURI(tokenId);
  return license == "ipfs://bafkreiaxmbyabctzurpssf4wcki3lk3zuxpqdwwdc4dfdmwsq";
}
```

This is much more scalable than each individual NFT creator implementing some special logic in their contract.

This is more decentralized, as it's up to each marketplace to decide whether to respect the license or not, and each marketplace can be held accountable for their decisions.

### Why a new onchain interface instead of inside metadata

We could imagine including the license URI in the NFT metadata, like the following:

```json
{
  "name": "licensed NFT",
  "description": "This is an example metadata for an NFT with a license",
  "image": "ipfs://...",
  "licenmse": "ipfs://..."  // The licenseURI
}
```

The problem with this is, the `tokenURI` for each metadata will be different for every single NFT item, and there is no way to tell which license is being used by an item just by looking at `tokenURI`. The only way to detect the license is by:

1. First fetching the metadata at `tokenURI`
2. Second, parsing the metadata to get the `license` attribute

This is:

1. Inefficient and messy: because it requires offchain fetching
2. Cannot be implemented onchain: because the "fetching" happens OUTSIDE of the blockchain. You can't build smart contract functions that take advantage of this approach.

On the other hand, if we implement the license onchain using a designated onchain `licenseURI()` interface, the only thing that needs to be checked is the `licenseURI`. This is 100% onchain therefore can be used in smart contracts as well as offchain indexers and apps.

Also, just like how open source licensing models have generally converged to a small number of popular ones such as MIT, Apache, GPL, etc., we expect a handful of popular NFT licenses to emerge, which means there will be only a handful of known license URIs to track if you are building an onchain app that talks to NFT contracts. Assuming that we are using content addressable URIs, implementing license onchain would be as simple as filtering based on known URIs, something like:

```
interface IRicardian {
  function licenseURI(uint256 tokenId) external view returns (string memory);
}
mapping (string => bool) private supported; // a mapping with supported licenseURIs as keys and `true` as value
function isListed(
  addres collection,
  uint tokenId
) pubilc view returns (bool) {
  string memory licenseURI = IRicardian(collection).licenseURI(tokenId);
  return supported[licenseURI];
}
```


### Not just about negative reinforcement

The whole discussion around NFT royalties has been centered around whether it is right or not right to respect royalties. This is not the right question to ask. Instead, disagreements in views should be respected. Instead of making value judgments, all we need is a way for creators to communicate what they exactly want, and a deterministic way for 3rd parties to interpret and make decisions.

If a marketplace disagrees with an NFT creator's view, they should be able to simply not list them on their marketplace. This is not necessarily a bad thing as long as there's a way for the NFT creator to publicly broadcast their intent.

In the future there will be various ways of interacting with NFTs, and not all of them will be for respecting NFT royalties, and not all NFT types will need royalty enforcement. Also, some NFTs may want much more flexible and dynamic (such as programmable license document) license. Therefore a blacklist base approach will only fragment the ecosystem and suppress innovation. With the current proposal, creators can easily declare intent and each 3rd party app (such as marketplaces) can make independent decisions, such as only listing NFTs that are compatible with their policy. The NFT creators who do not wish for their NFTs to be listed on certain marketplaces should NOT have to declare "please don't list on this, this, and this marketplace", but simply declare "My NFTs should be treated in this, this, and this way. Otherwise you are breaching the license.".


### Extensible

While this is titled as "Ricardian NFT Royalty", this same approach can be applied to:

1. All kinds of licensing model (not just royalty): The license can cover issues other than just royalty.
2. ALL kinds of smart contracts: Not just ERC721, but could even be for ERC20 and more.

Basically it's a simple method that turns any smart contract into a Ricardian contract.


## Backwards Compatibility

This proposal is completely optional and backward compatible, which is why it's called an "extension". This is an opt-in based solution. If you don't implement it, it simply means the NFT has no specific license and you are OK with anyone treating it as a public domain digital object (just like an open source project without an explicit license document). However if you do implement it, you are explicitly declaring the terms under which the items should be treated and traded on the market.

Furthermore, the whole point of this proposal is to suggest a solution that DOES NOT break backwards compatibility. It provides a way to respect EIP-2981 royalty standard WITHOUT having to force NFT creators to add specialized logic catering to each and every NFT marketplace, which is impossible in many cases because sometimes they are mutually exclusive.


## Security Considerations

There are no security considerations related directly to the implementation of this standard.


## Copyright

Copyright and related rights waived via [CC0](https://eips.ethereum.org/LICENSE).

