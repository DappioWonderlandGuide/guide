---
tags: devgitbook
---

# Setup

Let's begin implementing a simple aggregator from a clean slate. Here, we use [`dapp-scaffold`](https://github.com/solana-labs/dapp-scaffold), which handles a bunch of boilerplates already:

### Clone [`dapp-scaffold`](https://github.com/solana-labs/dapp-scaffold)

```bash
$ git clone git@github.com:solana-labs/dapp-scaffold.git
```

### Update `package.json`

Next, install all the dependencies:

```bash
$ yarn add @dappio-wonderland/navigator
$ yarn add @dappio-wonderland/gateway
$ yarn add @project-serum/anchor@^0.24.2
$ yarn add @solana/web3.js@^1.36.0
```

### Add resolution for `@jup-ag/core`

We have to lock the version of Jupiter core to `v1.0.0-beta.26` to avoid breaking changes, and also fix `@solana/wallet-adapter-react` to `0.15.18` which is workable version for connecting to phantom wallet:

```typescript!
// package.json

{
  "name": "universal-rabbit-hole-example",
  "version": "0.1.0",
  "license": "MIT",
  "private": false,
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "next lint"
  },
  "dependencies": {
    "@dappio-wonderland/gateway": "^0.2.5",
    "@dappio-wonderland/navigator": "^0.2.5",
    "@heroicons/react": "^1.0.5",
    "@project-serum/anchor": "^0.24.2",
    "@solana/wallet-adapter-base": "^0.9.3",
    "@solana/wallet-adapter-react": "0.15.18",
    "@solana/wallet-adapter-react-ui": "^0.9.5",
    "@solana/wallet-adapter-wallets": "^0.15.3",
    "@solana/web3.js": "^1.36.0",
    "@tailwindcss/typography": "^0.5.0",
    "daisyui": "^1.24.3",
    "immer": "^9.0.12",
    "next": "12.0.8",
    "next-compose-plugins": "^2.2.1",
    "next-transpile-modules": "^9.0.0",
    "react": "17.0.2",
    "react-dom": "17.0.2",
    "zustand": "^3.6.9"
  },
  "devDependencies": {
    "@types/node": "17.0.10",
    "@types/react": "17.0.38",
    "autoprefixer": "^10.4.2",
    "eslint": "8.7.0",
    "eslint-config-next": "12.0.8",
    "postcss": "^8.4.5",
    "tailwindcss": "^3.0.15",
    "typescript": "4.5.4"
  },
  "resolutions": {
    "@jup-ag/core": "1.0.0-beta.26",
    "@solana/wallet-adapter-react": "0.15.18"
  }
}
```

Install dependency by running

```
$ yarn
```

### Update `next.config.js`

Since some of the dependencies are using `fs`, we need to config next to make the compiler work. 

Replace `next.config.js` with the following code snippet:

```javascript!
// next.config.js

/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  webpack: (config, { isServer }) => {
    if (!isServer) {
      config.resolve.fallback.fs = false;
    }
    return config;
  },
};

module.exports = nextConfig;
```

### Add `AnchorWallet`

`AnchorWallet` is used as an interface that wraps the native wallet with a anchor wallet interface since Dappio Gateway uses [anchor](https://github.com/coral-xyz/anchor) internally:

```bash
$ touch src/utils/anchorWallet.tsx
```

```typescript!
// src/utils/anchorWallet.tsx

import { PublicKey, Transaction, Keypair } from "@solana/web3.js";
import { WalletContextState } from "@solana/wallet-adapter-react";

export class AnchorWallet {
  readonly payer: Keypair;
  readonly publicKey: PublicKey;

  constructor(readonly wallet: WalletContextState) {
    // Never used dummy payer
    this.payer = Keypair.generate();
    this.publicKey = wallet.publicKey ? wallet.publicKey : PublicKey.default;
  }

  async signTransaction(tx: Transaction): Promise<Transaction> {
    return await this.wallet.signTransaction!(tx);
  }

  async signAllTransactions(txs: Transaction[]): Promise<Transaction[]> {
    return await this.wallet.signAllTransactions!(txs);
  }
}
```