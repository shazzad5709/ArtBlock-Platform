# ArtBlock Platform

Platform ArtBlock is a way to create a creator based community that focuses on specific type of art. These communities are a form of DAO.

## Feautures  

1. Users can purchase platform native ERC20 ABX Tokens with a fixed price of ethers.
2. User can create new communities by speending ABX Token.
3. Each community will have their own native ERC20 token which can be exchanged with ABX Token.
4. Each community will have a Decentralized EXchange (DEX) which will have two assets - ABX Token & the community native token.
5. The liquidity pool follows Constant Product Automated Market Maker (CPAMM). The price curve for these AMMs abides by the x*y=k equation, where X and Y are the quantities of assets 1 and 2, and K is a constant.
6. Creators can publish their art in respective communities for approval by staking certain community native tokens. The arts can be of two category - exclusive & general.
7. Community members can be vote on the approval of the product with options for upvote & downvote. The voting is weighted.
8. Artists can host Dutch auctions for approved exclusive items.
9. Auctioned exclusive arts will be minted as Non-Transferable Token.
10. Approved general arts will be minted as NFT.
11. There is an ArtBlock marketplace where creators can sell their general art products in community native tokens & current owners can resell general arts.
12. There is a dynamic royalty system for original creators allowing them to set a percentage for each subsequent rsell. 

## Used Technology
 - [👷🏽‍♂️ Hardhat](https://www.rainbowkit.com/)
 - [🌈 RainbowKit](https://hardhat.org/)
 - [➬ WAGMI](https://wagmi.sh/)
 - [🌐 Next JS](https://nextjs.org/)
 - [🗺 Web3.storage]
 - [🗺 Etherscan](https://etherscan.io/)
 - [🕹 Typechain](https://github.com/dethcrypto/TypeChain)
 - [TailwindCSS](https://tailwindcss.com) – Utility-first CSS framework for rapid UI development
 - [TypeScript](https://www.typescriptlang.org/) – Static type checker for end-to-end typesafety
 - [Prettier](https://prettier.io/) – Opinionated code formatter for consistent code style
 - [ESLint](https://eslint.org/) – Pluggable linter for Next.js and TypeScript

## Prerequisite
```
Node js
npm
yarn
```
## Install dependencies
Install yarn:
```
npm i -g yarn
```
For contract dev:
Run this command on the root folder:
```
yarn
```

For Frontend dev:
Go to `frontend` folder and install node modules:
```
cd frontend
yarn

```

## Instruction
- Install a wallet like [Metamask](https://chrome.google.com/webstore/detail/metamask/nkbihfbeogaeaoehlefnkodbefgpgknn)
- Copy `.env.example` to `.env`
  * Mac or Linux
    * ```cp .env.example .env```
  * Windows
    * ```copy .env.example .env```
- Set the env variable in `.env` file on root level and on `frontend` folder:

Variable descriptions:

1. `RPC_NODE_API_KEY`: Get from [Alchemy site](https://auth.alchemy.com/signup/) after sign up and login
2. `PRIVATE_KEY`: Export private key from metamask, follow these [instructions](https://support.metamask.io/hc/en-us/articles/360015289632-How-to-export-an-account-s-private-key)
3. `ETHERSCAN_API_KEY`: Get from [etherscan](https://etherscan.io/login)

Frontend ENV Variable:
4. `NEXT_PUBLIC_ALCHEMY_API_KEY` : Same as `RPC_NODE_API_KEY` 
- Compile Contract:
```
npm run compile
```
- Run test:
```
npm run test
```
- Deploy
```
npm run deploy:<network>
```
- Verify on etherscan
```
npx hardhat verify --network sepolia <YOUR_CONTRACT_ADDRESS> <Paramaters>
```

## Some key code snippets
### Weighted Voting
```solidity
    function upvote(uint proposalId) external {
        ArtProposal storage _artProposal = artProposals[proposalId];
        require(_artProposal.closingTime < block.timestamp);
        
        uint weight = (1 ether * comToken.balanceOf(msg.sender) / comToken.totalSupply());
        _artProposal.upvotes += weight;
    }
```

### Dutch Auction
```solidity
    function createDutchAuction(Artwork memory _artwork, uint _start, uint _decrement, uint _interval, uint _minprice) external {
        Auction storage auction = auctions.push();
        auction.artwork = _artwork;
        auction.start = _start;
        auction.decrement = _decrement * 60; //decrement is in minutes, multiplied by 60 to convert to seconds
        auction.interval = _interval;
        auction.minprice = _minprice;
        auction.artist = msg.sender;
        auction.startTime = block.timestamp;

        auctionMap[auctionIndex++] = auction;
    }

    mapping (uint => Artwork) exclusiveArtIndex;

    function placeBid(uint _auctionId) external {
        Auction storage auction = auctionMap[_auctionId];
        uint currentPrice = auction.start - (auction.decrement * (block.timestamp - auction.start) / auction.interval);

        require (
            comToken.transferFrom(msg.sender, auction.artwork.artist, currentPrice)
        );

        uint id = exclusiveNTT.safeMint(msg.sender);
        exclusiveArtIndex[id] = auction.artwork;
        artNFT.burn(auction.artwork.nftTokenId);
    }
```

### Non-Transferrable Exclusive Auctioned Art
```solidity
    function _beforeTokenTransfer(
        address from,
        address to,
        uint256 tokenId,
        uint256 batchSize
    ) internal virtual override {
        require(from == address(0), "Err: token transfer is BLOCKED");
        super._beforeTokenTransfer(from, to, tokenId, batchSize);
    }
```

### Automated Dynamic Royalty
```solidity
   function buyFromMarket (uint _id) external {
        Sellpost storage _sellpost = sellposts[getSellpost(_id)];
        require(comToken.transferFrom(msg.sender, artNFT.ownerOf(_sellpost.artwork.nftTokenId), _sellpost.price - (_sellpost.price * _sellpost.artwork.royalty / 100 )), "Community Token transfer to owner 
         failed!");
        require(comToken.transferFrom(msg.sender, _sellpost.artwork.artist, _sellpost.price * _sellpost.artwork.royalty / 100 ), "Community Token transfer to artist failed!");

        _sellpost.isSold = true;
    }
```

### Adding Liquidity in DEX
```solidity
    function addLiquidity(uint256 _abxAmount, uint256 _comAmount)
        external
        returns (uint256 shares)
    {
        uint256 abxReserve = getAbxReserve();
        uint256 comReserve = getComReserve();

        if (abxReserve > 0 || comReserve > 0) {
            require(
                abxReserve * _comAmount == comReserve * _abxAmount
            );
        }

        abxToken.transferFrom(msg.sender, address(this), _abxAmount);
        comToken.transferFrom(msg.sender, address(this), _comAmount);

        if (totalSupply == 0) {
            shares = _sqrt(_abxAmount * _comAmount);
        } else {
            shares = _min(
                (_abxAmount * totalSupply) / abxReserve,
                (_comAmount * totalSupply) / comReserve
            );
        }
        require(shares > 0);
        _mint(msg.sender, shares);
    }
```

### Swapping in DEX
```solidity
    function swap(address _tokenIn, uint256 _amountIn)
        internal
        returns (uint256 amountOut)
    {
        require(
            _tokenIn == address(abxToken) || _tokenIn == address(comToken)
        );
        require(_amountIn > 0, "amount in = 0");

        bool isAbxToken = _tokenIn == address(abxToken);
        (
            IERC20 tokenIn,
            IERC20 tokenOut,
            uint256 reserveIn,
            uint256 reserveOut
        ) = isAbxToken
                ? (abxToken, comToken, getAbxReserve(), getComReserve())
                : (comToken, abxToken, getComReserve(), getAbxReserve());

        tokenIn.transferFrom(msg.sender, address(this), _amountIn);

        uint256 amountInWithFee = (_amountIn * 997) / 1000;
        amountOut =
            (reserveOut * amountInWithFee) /
            (reserveIn + amountInWithFee);

        tokenOut.transfer(msg.sender, amountOut);
    }
```





For example for `ArtBlockPlatform` contract:
```
npx hardhat verify --network sepolia 0xAECD7dFD9d5ED08EA916B052D90A75366B963A61 "Hello world"
```

