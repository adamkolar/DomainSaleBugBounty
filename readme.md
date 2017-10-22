# DomainSale contract bug bounty

  * https://www.reddit.com/r/ethdev/comments/719u9f/domainsale_bug_bounty_1eth/
  * Date: 2017-09-24
  * Reward: 1 ETH

## Contracts

[DomainSale.sol](./contracts/DomainSale.sol)

## Report

The way funds are transferred when the auction ends right now allows seller to set up a trap. If the previous owner of the name is a contract which throws in the fallback payable function, the auction can never be finished (because transfer propagates exceptions). This leads to the last bidder's funds being locked indefinitely and the name staying in the ownership of the DomainSale contract with the original resolver.
If the previous owner contract has a condition in the payable function, the seller can even try to extort additional money from the bidder to allow the auction to finish. The only safe way to bid on names the way the contract is set up right now, is to examine what's on previous owner (and startRefferer) address before placing a bid, which seems very impractical.
One way this could be resolved is to increase the internal balances of the parties to the auction when it is finished instead of transferring the funds.

```javascript
    /**
     * @dev finish an auction
     */
    function finish(string _name) deedValid(_name) public {
        Sale storage s = sales[_name];
        require(now > s.auctionEnds);

        // Obtain the previous owner from the deed
        Deed deed;
        (,deed,,,) = registrar.entries(sha3(_name));

        address previousOwner = deed.previousOwner();
        registrar.transfer(sha3(_name), s.lastBidder);
        Transfer(previousOwner, s.lastBidder, _name, s.lastBid);

        // Distribute funds to referrers
        transferFunds(s.lastBid, previousOwner, s.startReferrer, s.bidReferrer);

        // Finished with the sale information
        delete sales[_name];

        // As we're here, return any funds that the sender is owed
        withdraw();
    }
```

```javascript
    /**
     * @dev Transfer funds for a sale to the relevant parties
     */
    function transferFunds(uint256 amount, address seller, address startReferrer, address bidReferrer) internal {
        uint256 startReferrerFunds = amount * START_REFERRER_SALE_PERCENTAGE / 100;
        uint256 bidReferrerFunds = amount * BID_REFERRER_SALE_PERCENTAGE / 100;
        uint256 sellerFunds = amount - startReferrerFunds - bidReferrerFunds;
        seller.transfer(sellerFunds);
        startReferrer.transfer(startReferrerFunds);
        bidReferrer.transfer(bidReferrerFunds);
    }
```