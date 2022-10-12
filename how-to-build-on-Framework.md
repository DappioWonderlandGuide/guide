---
tags: devgitbook
---

#  How to Build on Framework


## Part 1: Add Raydium Farms

In this part, we will start from implementing Raydium farm list where user can zap-in and out their fund by just one click.

### Add `NavigatorProvider`

```bash
$ touch src/contexts/NavigatorProvider.tsx
```

Let's implement it by pasting the following code snippet:

```typescript!
// src/contexts/NavigatorProvider.tsx

import { useConnection } from "@solana/wallet-adapter-react";
import {
  createContext,
  FC,
  ReactNode,
  useContext,
  useEffect,
  useState,
} from "react";
import { raydium } from "@dappio-wonderland/navigator";

export interface NavigatorContextState {
  raydiumFarms: raydium.FarmInfoWrapper[];
  raydiumPoolSetWithLpMintKey: Map<string, raydium.PoolInfoWrapper>;
}

export const NavigatorContext = createContext<NavigatorContextState>(
  {} as NavigatorContextState
);

export function useNavigator(): NavigatorContextState {
  return useContext(NavigatorContext);
}

export const NavigatorProvider: FC<{ children: ReactNode }> = ({
  children,
}) => {
  const { connection } = useConnection();
  const [raydiumFarms, setRaydiumFarms] = useState<raydium.FarmInfoWrapper[]>(
    []
  );
  const [raydiumPoolSetWithLpMintKey, setRaydiumPoolSetWithLpMintKey] =
    useState<Map<string, raydium.PoolInfoWrapper>>(
      {} as Map<string, raydium.PoolInfoWrapper>
    );

  useEffect(() => {
    {
      const getAllFarmsWrappers = async () => {
        return (await raydium.infos.getAllFarmWrappers(
          connection
        )) as raydium.FarmInfoWrapper[];
      };

      getAllFarmsWrappers().then((wrappers) => {
        setRaydiumFarms(wrappers);
      });

      const getAllPoolWrappers = async () => {
        const poolWrappers = await raydium.infos.getAllPoolWrappers(connection);

        return new Map<string, raydium.PoolInfoWrapper>(
          poolWrappers.map((wrapper) => [
            wrapper.poolInfo.lpMint.toString(),
            wrapper as raydium.PoolInfoWrapper,
          ])
        );
      };

      getAllPoolWrappers().then((poolSetResult) => {
        setRaydiumPoolSetWithLpMintKey(poolSetResult);
      });
    }
  }, []);

  return (
    <NavigatorContext.Provider
      value={{
        raydiumFarms,
        raydiumPoolSetWithLpMintKey,
      }}
    >
      {children}
    </NavigatorContext.Provider>
  );
};
```

As you can see, you can get all the farms as simple as **one line of code!** (`raydium.infos.getAllFarmWrappers(connection)`)

### Update `ContextProvider`

Next, replace `ContextProvider.tsx` with the following code snippet. Here, we need to wrap children elements with `NavigatorProvider`:

```typescript!
// src/contexts/ContextProvider.tsx

import { WalletAdapterNetwork, WalletError } from "@solana/wallet-adapter-base";
import {
  ConnectionProvider,
  WalletProvider,
} from "@solana/wallet-adapter-react";
import { WalletModalProvider as ReactUIWalletModalProvider } from "@solana/wallet-adapter-react-ui";
import {
  PhantomWalletAdapter,
  SolflareWalletAdapter,
  SolletExtensionWalletAdapter,
  SolletWalletAdapter,
  TorusWalletAdapter,
  // LedgerWalletAdapter,
  // SlopeWalletAdapter,
} from "@solana/wallet-adapter-wallets";
import { Cluster, clusterApiUrl, ConnectionConfig } from "@solana/web3.js";
import { FC, ReactNode, useCallback, useMemo } from "react";
import { AutoConnectProvider, useAutoConnect } from "./AutoConnectProvider";
import { notify } from "../utils/notifications";
import {
  NetworkConfigurationProvider,
  useNetworkConfiguration,
} from "./NetworkConfigurationProvider";
import { NavigatorProvider } from "./NavigatorProvider";

const WalletContextProvider: FC<{ children: ReactNode }> = ({ children }) => {
  const { autoConnect } = useAutoConnect();
  const { networkConfiguration } = useNetworkConfiguration();
  const network = networkConfiguration as WalletAdapterNetwork;
  const endpoint = "https://rpc-mainnet-fork.epochs.studio";
  const config: ConnectionConfig = {
    wsEndpoint: "wss://rpc-mainnet-fork.epochs.studio/ws",
    commitment: "confirmed",
    confirmTransactionInitialTimeout: 300 * 1000,
  };

  const wallets = useMemo(
    () => [
      new PhantomWalletAdapter(),
      new SolflareWalletAdapter(),
      new SolletWalletAdapter({ network }),
      new SolletExtensionWalletAdapter({ network }),
      new TorusWalletAdapter(),
      // new LedgerWalletAdapter(),
      // new SlopeWalletAdapter(),
    ],
    [network]
  );

  const onError = useCallback((error: WalletError) => {
    notify({
      type: "error",
      message: error.message ? `${error.name}: ${error.message}` : error.name,
    });
    console.error(error);
  }, []);

  return (
    // TODO: updates needed for updating and referencing endpoint: wallet adapter rework
    <ConnectionProvider endpoint={endpoint} config={config}>
      <WalletProvider
        wallets={wallets}
        onError={onError}
        autoConnect={autoConnect}
      >
        <ReactUIWalletModalProvider>{children}</ReactUIWalletModalProvider>
      </WalletProvider>
    </ConnectionProvider>
  );
};

export const ContextProvider: FC<{ children: ReactNode }> = ({ children }) => {
  return (
    <>
      <NetworkConfigurationProvider>
        <AutoConnectProvider>
          <WalletContextProvider>
            <NavigatorProvider>{children}</NavigatorProvider>
          </WalletContextProvider>
        </AutoConnectProvider>
      </NetworkConfigurationProvider>
    </>
  );
};
```

Another change we made is that: **it configs the connection object to mainnet-fork RPC**, which is a [`solana-mf` (mainnet-fork)](https://github.com/DappioWonderland/solana) validator developed by Dappio. `solana-mf` restores the entire mainnet account data locally. Developers can build and test with mainnet account data on a tesing environmant. This is especially handy for testing program that needs to interact with different programs:


### Add `raydiumFarms`

Create a file `raydiumFarms.tsx` in `pages` folder:

```bash
$ touch src/pages/raydiumFarms.tsx
```

Implement this page by pasting the following code snippet:

```typescript!
// src/pages/raydiumFarms.tsx

import { useNavigator } from "contexts/NavigatorProvider";
import { NextPage } from "next";
import Head from "next/head";
import { useEffect, useState } from "react";
import { raydium as protocol } from "@dappio-wonderland/navigator";
import { Farm } from "components/RaydiumFarm";

export const RaydiumFarms: NextPage = (props) => {
  const { raydiumFarms, raydiumPoolSetWithLpMintKey } = useNavigator();
  const [farmsWithPool, setFarmsWithPool] = useState<protocol.FarmInfoWrapper[]>(
    []
  );

  useEffect(() => {
    const farmsWithPool = raydiumFarms.filter((farm) => {
      return raydiumPoolSetWithLpMintKey.size > 0
        ? raydiumPoolSetWithLpMintKey.has(
            farm.farmInfo.poolLpTokenAccount.mint.toString()
          )
        : false;
    });
    setFarmsWithPool(farmsWithPool);
  }, [raydiumPoolSetWithLpMintKey]);

  return (
    <div>
      <Head>
        <title>Solana Scaffold</title>
        <meta name="description" content="Farms" />
      </Head>
      <div className="md:hero mx-auto p-4">
        <div className="md:hero-content flex flex-col">
          <h1 className="text-center text-5xl font-bold text-transparent bg-clip-text bg-gradient-to-tr from-[#9945FF] to-[#14F195]">
            Raydium Farms
          </h1>
          {/* CONTENT GOES HERE */}
          <div className="overflow-x-auto">
            <table className="table w-full">
              <thead>
                <tr>
                  <th>Farm ID</th>
                  <th>LP Token</th>
                  <th>APY</th>
                  <th></th>
                </tr>
              </thead>
              <tbody>
                {farmsWithPool
                  .sort((a, b) =>
                    a.farmInfo.farmId
                      .toString()
                      .localeCompare(b.farmInfo.farmId.toString())
                  )
                  .map((farm) => (
                    <Farm
                      key={farm.farmInfo.farmId.toString()}
                      farm={farm}
                      pool={raydiumPoolSetWithLpMintKey.get(
                        farm.farmInfo.poolLpTokenAccount.mint.toString()
                      )}
                    ></Farm>
                  ))}
              </tbody>
            </table>
          </div>
          <div className="text-center"></div>
        </div>
      </div>
    </div>
  );
};

export default RaydiumFarms;
```

At this point, you might notice that the component `RaydiumFarm` is missing. Let's implement it in the next step.

### Add `RaydiumFarm`

Next, let's implement the Farm component.

```bash
$ touch src/components/RaydiumFarm.tsx
```

Paste the following code snippet to `RaydiumFarm.tsx`:

```typescript!
// src/components/RaydiumFarm.tsx

import { FC, useCallback, useEffect, useState } from "react";
import { PublicKey } from "@solana/web3.js";
import { notify } from "utils/notifications";
import { useConnection, useWallet } from "@solana/wallet-adapter-react";
import useUserSOLBalanceStore from "stores/useUserSOLBalanceStore";
import { AnchorWallet } from "utils/anchorWallet";
import * as anchor from "@project-serum/anchor";
import { raydium as protocol } from "@dappio-wonderland/navigator";
import {
  AddLiquidityParams,
  GatewayBuilder,
  HarvestParams,
  RemoveLiquidityParams,
  StakeParams,
  SupportedProtocols,
  SwapParams,
  UnstakeParams,
  WSOL,
} from "@dappio-wonderland/gateway";

interface FarmProps {
  farm: protocol.FarmInfoWrapper;
  pool: protocol.PoolInfoWrapper;
}

const protocolType = SupportedProtocols.Raydium;

export const Farm: FC<FarmProps> = (props: FarmProps) => {
  const [apr, setApr] = useState(0);
  const { connection } = useConnection();
  const wallet = useWallet();
  const { getUserSOLBalance } = useUserSOLBalanceStore();

  // Get Farm
  const farm = props.farm;
  const farmInfo = farm.farmInfo;
  const farmId = farmInfo.farmId.toString();
  const lpMint = farmInfo.poolLpTokenAccount.mint.toString();
  const pool = props.pool;
  const poolInfo = pool.poolInfo;
  useEffect(() => {
    // NOTICE: We mocked LP price and reward price here just for demo
    const aprs = farm.getAprs(5, 1, 2);
    const apr = aprs.length > 1 ? aprs[1] : aprs[0];
    setApr(apr);
  }, []);

  const zapIn = useCallback(async () => {
    if (!wallet.publicKey) {
      console.error("error", "Wallet not connected!");
      notify({
        type: "error",
        message: "error",
        description: "Wallet not connected!",
      });
      return;
    }

    const provider = new anchor.AnchorProvider(
      connection,
      new AnchorWallet(wallet),
      anchor.AnchorProvider.defaultOptions()
    );
    const zapInAmount = 10000; // WSOL Amount

    // SOL to tokenA
    const swapParams1: SwapParams = {
      protocol: SupportedProtocols.Jupiter,
      fromTokenMint: new PublicKey(WSOL),
      toTokenMint: poolInfo.tokenAMint,
      amount: zapInAmount,
      slippage: 1,
    };
    // tokenA to tokenB
    const swapParams2: SwapParams = {
      protocol: SupportedProtocols.Jupiter,
      fromTokenMint: poolInfo.tokenAMint,
      toTokenMint: poolInfo.tokenBMint,
      amount: 0, // Notice: amount needs to be updated later
      slippage: 1,
    };
    const addLiquidityParams: AddLiquidityParams = {
      protocol: protocolType,
      poolId: poolInfo.poolId,
    };
    const stakeParams: StakeParams = {
      protocol: protocolType,
      farmId: farmInfo.farmId,
      version: farmInfo.version,
    };

    const gateway = new GatewayBuilder(provider);

    // 1st Swap
    await gateway.swap(swapParams1);
    const minOutAmount1 = gateway.params.swapMinOutAmount.toNumber();

    // 2nd Swap
    swapParams2.amount = minOutAmount1 / 2;
    await gateway.swap(swapParams2);
    const minOutAmount2 = gateway.params.swapMinOutAmount.toNumber();

    // Add Liquidity
    addLiquidityParams.tokenInAmount = minOutAmount2;
    await gateway.addLiquidity(addLiquidityParams);

    // Stake
    await gateway.stake(stakeParams);

    await gateway.finalize();
    const txs = gateway.transactions();

    const recentBlockhash = (await connection.getLatestBlockhash()).blockhash;

    txs.forEach((tx) => {
      tx.recentBlockhash = recentBlockhash;
      tx.feePayer = wallet.publicKey;
    });

    const signTxs = await provider.wallet.signAllTransactions(txs);

    console.log("======");
    console.log("Txs are sent...");
    for (let tx of signTxs) {
      let sig: string = "";
      try {
        sig = await connection.sendRawTransaction(tx.serialize(), {
          skipPreflight: true,
          commitment: "confirmed",
        } as unknown as anchor.web3.SendOptions);
        await connection.confirmTransaction(sig, connection.commitment);

        notify({
          type: "success",
          message: "Transaction is executed successfully!",
          txid: sig,
        });
      } catch (error: any) {
        notify({
          type: "error",
          message: `Transaction failed!`,
          description: error?.message,
          txid: sig,
        });
        console.log(
          "NOTICE: paste the output to Transaction Inspector in Solana Explorer for debugging"
        );
        console.log(tx.serializeMessage().toString("base64"));
        console.error("error", `Transaction failed! ${error?.message}`, sig);
        break;
      }
    }
    console.log("Txs are executed");
    console.log("======");

    getUserSOLBalance(wallet.publicKey, connection);
  }, [wallet.publicKey, connection, getUserSOLBalance]);

  const zapOut = useCallback(async () => {
    if (!wallet.publicKey) {
      console.error("error", "Wallet not connected!");
      notify({
        type: "error",
        message: "error",
        description: "Wallet not connected!",
      });
      return;
    }

    const provider = new anchor.AnchorProvider(
      connection,
      new AnchorWallet(wallet),
      anchor.AnchorProvider.defaultOptions()
    );

    // Get share amount
    const ledgerKey = await protocol.infos.getFarmerId(
      farmInfo,
      provider.wallet.publicKey,
      farmInfo.version
    );
    const ledger = (await protocol.infos.getFarmer(
      connection,
      ledgerKey,
      farmInfo.version
    )) as protocol.FarmerInfo;
    const shareAmount = ledger.amount;
    const { tokenAAmount, tokenBAmount } = await pool.getTokenAmounts(
      shareAmount
    );

    const harvestParams: HarvestParams = {
      protocol: protocolType,
      farmId: farmInfo.farmId,
      version: farmInfo.version,
    };
    const unstakeParams: UnstakeParams = {
      protocol: protocolType,
      farmId: farmInfo.farmId,
      shareAmount,
      version: farmInfo.version,
    };
    const removeLiquidityParams: RemoveLiquidityParams = {
      protocol: protocolType,
      poolId: poolInfo.poolId,
    };
    // tokenB to tokenA
    const swapParams1: SwapParams = {
      protocol: SupportedProtocols.Jupiter,
      fromTokenMint: poolInfo.tokenBMint,
      toTokenMint: poolInfo.tokenAMint,
      amount: tokenBAmount, // swap coin to pc
      slippage: 3,
    };

    // tokenA to SOL
    const swapParams2: SwapParams = {
      protocol: SupportedProtocols.Jupiter,
      fromTokenMint: poolInfo.tokenAMint,
      toTokenMint: new PublicKey(WSOL),
      amount: 0, // Notice: This amount needs to be updated later
      slippage: 10,
    };

    const gateway = new GatewayBuilder(provider);

    await gateway.harvest(harvestParams);
    await gateway.unstake(unstakeParams);
    await gateway.removeLiquidity(removeLiquidityParams);

    // 1st Swap
    await gateway.swap(swapParams1);
    const minOutAmount = gateway.params.swapMinOutAmount.toNumber();
    swapParams2.amount = minOutAmount + tokenAAmount;
    // 2nd Swap
    await gateway.swap(swapParams2);

    await gateway.finalize();
    const txs = gateway.transactions();

    const recentBlockhash = (await connection.getLatestBlockhash()).blockhash;

    txs.forEach((tx) => {
      tx.recentBlockhash = recentBlockhash;
      tx.feePayer = wallet.publicKey;
    });

    const signTxs = await provider.wallet.signAllTransactions(txs);

    console.log("======");
    console.log("Txs are sent...");
    for (let tx of signTxs) {
      let sig: string = "";
      try {
        sig = await connection.sendRawTransaction(tx.serialize(), {
          skipPreflight: true,
          commitment: "confirmed",
        } as unknown as anchor.web3.SendOptions);
        await connection.confirmTransaction(sig, connection.commitment);

        notify({
          type: "success",
          message: "Transaction is executed successfully!",
          txid: sig,
        });
      } catch (error: any) {
        notify({
          type: "error",
          message: `Transaction failed!`,
          description: error?.message,
          txid: sig,
        });
        console.log(
          "NOTICE: paste the output to Transaction Inspector in Solana Explorer for debugging"
        );
        console.log(tx.serializeMessage().toString("base64"));
        console.error("error", `Transaction failed! ${error?.message}`, sig);
        break;
      }
    }
    console.log("Txs are executed");
    console.log("======");

    getUserSOLBalance(wallet.publicKey, connection);
  }, [wallet.publicKey, connection, getUserSOLBalance]);

  return (
    <tr>
      <th>
        {farmId.slice(0, 5)}...{farmId.slice(farmId.length - 5)}
      </th>
      <td>
        {lpMint.slice(0, 5)}...{lpMint.slice(lpMint.length - 5)}
      </td>
      <td>{apr.toFixed(2) + "%"}</td>
      <td>
        <button className="btn btn-info" onClick={zapIn}>
          Zap In
        </button>
        &nbsp;
        <button className="btn btn-warning" onClick={zapOut}>
          Zap Out
        </button>
      </td>
    </tr>
  );
};
```

You may notice that this is the place where Gateway kicks in:
- There are 2 functions: Zap-in and Zap-out
- Inside each functions, various types of parameters are declared. (`SwapParams`, `AddLiquidityParams`, `StakeParams`, etc)
- Actions are assembled by simply **a few lines of code!** For example:

```typescript!
const gateway = new GatewayBuilder(provider);

await gateway.swap(swapParams1);

// ...

await gateway.swap(swapParams2);

// ...

await gateway.addLiquidity(addLiquidityParams);

// ...

await gateway.stake(stakeParams);

await gateway.finalize();
const txs = gateway.transactions();
```

### Update `ContentContainer`

Replace `ContentContainer` with the following code snippet. This will append a new link to `RaydiumFarms` page:

```typescript!
// src/components/ContentContainer.tsx

import { FC } from "react";
import Link from "next/link";
export const ContentContainer: FC = (props) => {
  return (
    <div className="flex-1 drawer h-52">
      {/* <div className="h-screen drawer drawer-mobile w-full"> */}
      <input id="my-drawer" type="checkbox" className="grow drawer-toggle" />
      <div className="items-center  drawer-content">{props.children}</div>

      {/* SideBar / Drawer */}
      <div className="drawer-side">
        <label htmlFor="my-drawer" className="drawer-overlay"></label>
        <ul className="p-4 overflow-y-auto menu w-80 bg-base-100">
          <li>
            <h1>Menu</h1>
          </li>
          <li>
            <Link href="/">
              <a>Home</a>
            </Link>
          </li>
          <li>
            <Link href="/basics">
              <a>Basics</a>
            </Link>
          </li>
          <li>
            <Link href="/raydiumFarms">
              <a>Raydium Farms</a>
            </Link>
          </li>
        </ul>
      </div>
    </div>
  );
};
```

### Compile and Run

```bash
$ yarn dev
```

Then open http://localhost:3000 (default).

> **Notice: Sometimes the Jupiter aggregator could fail due to the outdated mainnet-fork snapshot.** When it happens, you might have to remove some AMM liquidity pools directly from the dependencies (a.k.a `node_modules`) to mitigate this error.

## Part 2: Add Orca Farms

In this part, we will continue to add Orca Farms. You will soon discover that **the interfaces are exactly identical no matter which protocol you are trying to access.**

### Update `NavigatorProvider`

Replace `NavigatorPrivider.tsx` with the following code snippet:

```typescript!
// src/contexts/NavigatorPrivider.tsx

import { useConnection } from "@solana/wallet-adapter-react";
import {
  createContext,
  FC,
  ReactNode,
  useContext,
  useEffect,
  useState,
} from "react";
// Added from Part 2
import { raydium, orca } from "@dappio-wonderland/navigator";

export interface NavigatorContextState {
  raydiumFarms: raydium.FarmInfoWrapper[];
  raydiumPoolSetWithLpMintKey: Map<string, raydium.PoolInfoWrapper>;
  // *************************
  // Added from Part 2 (Begin)
  orcaFarms: orca.FarmInfoWrapper[];
  orcaPoolSetWithLpMintKey: Map<string, orca.PoolInfoWrapper>;
  // Added from Part 2 (End)
  // *************************
}

export const NavigatorContext = createContext<NavigatorContextState>(
  {} as NavigatorContextState
);

export function useNavigator(): NavigatorContextState {
  return useContext(NavigatorContext);
}

export const NavigatorProvider: FC<{ children: ReactNode }> = ({
  children,
}) => {
  const { connection } = useConnection();
  const [raydiumFarms, setRaydiumFarms] = useState<raydium.FarmInfoWrapper[]>(
    []
  );
  const [raydiumPoolSetWithLpMintKey, setRaydiumPoolSetWithLpMintKey] =
    useState<Map<string, raydium.PoolInfoWrapper>>(
      {} as Map<string, raydium.PoolInfoWrapper>
    );
  // *************************
  // Added from Part 2 (Begin)
  const [orcaFarms, setOrcaFarms] = useState<orca.FarmInfoWrapper[]>([]);
  const [orcaPoolSetWithLpMintKey, setOrcaPoolSetWithLpMintKey] = useState<
    Map<string, orca.PoolInfoWrapper>
  >({} as Map<string, orca.PoolInfoWrapper>);
  // Added from Part 2 (End)
  // *************************
  
  useEffect(() => {
    {
      const getAllFarmsWrappers = async () => {
        return (await raydium.infos.getAllFarmWrappers(
          connection
        )) as raydium.FarmInfoWrapper[];
      };

      getAllFarmsWrappers().then((wrappers) => {
        setRaydiumFarms(wrappers);
      });

      const getAllPoolWrappers = async () => {
        const poolWrappers = await raydium.infos.getAllPoolWrappers(connection);

        return new Map<string, raydium.PoolInfoWrapper>(
          poolWrappers.map((wrapper) => [
            wrapper.poolInfo.lpMint.toString(),
            wrapper as raydium.PoolInfoWrapper,
          ])
        );
      };

      getAllPoolWrappers().then((poolSetResult) => {
        setRaydiumPoolSetWithLpMintKey(poolSetResult);
      });
    }
    // *************************
    // Added from Part 2 (Begin)
    {
      const getAllFarmsWrappers = async () => {
        return (await orca.infos.getAllFarmWrappers(
          connection
        )) as orca.FarmInfoWrapper[];
      };

      getAllFarmsWrappers().then((wrappers) => {
        setOrcaFarms(wrappers);
      });

      const getAllPoolWrappers = async () => {
        const poolWrappers = await orca.infos.getAllPoolWrappers(connection);

        return new Map<string, orca.PoolInfoWrapper>(
          poolWrappers.map((wrapper) => [
            wrapper.poolInfo.lpMint.toString(),
            wrapper as orca.PoolInfoWrapper,
          ])
        );
      };

      getAllPoolWrappers().then((poolSetResult) => {
        setOrcaPoolSetWithLpMintKey(poolSetResult);
      });
    }
    // Added from Part 2 (End)
    // *************************
  }, []);

  return (
    <NavigatorContext.Provider
      value={{
        raydiumFarms,
        raydiumPoolSetWithLpMintKey,
        // *************************
        // Added from Part 2 (Begin)
        orcaFarms,
        orcaPoolSetWithLpMintKey,
        // Added from Part 2 (End)
        // *************************
      }}
    >
      {children}
    </NavigatorContext.Provider>
  );
};
```

Notice the highlighting comments in the code snippet above.

### Add `orcaFarms`

Copy from `raydiumFarms`:

```bash!
$ cp src/pages/raydiumFarms.tsx src/pages/orcaFarms.tsx
```

Replace `ocraFarms.tsx` with the following code snippet:

```typescript!
// src/pages/orcaFarms.tsx

import { useNavigator } from "contexts/NavigatorProvider";
import { NextPage } from "next";
import Head from "next/head";
import { useEffect, useState } from "react";
import { orca } from "@dappio-wonderland/navigator";
import { Farm } from "../components/OrcaFarm";

export const OrcaFarms: NextPage = (props) => {
  const { orcaFarms, orcaPoolSetWithLpMintKey } = useNavigator();
  const [farmsWithPool, setFarmsWithPool] = useState<orca.FarmInfoWrapper[]>(
    []
  );

  useEffect(() => {
    setFarmsWithPool(
      orcaFarms.filter((farm) => {
        return orcaPoolSetWithLpMintKey.size > 0
          ? orcaPoolSetWithLpMintKey.has(farm.farmInfo.baseTokenMint.toString())
          : false;
      })
    );
  }, [orcaFarms]);

  return (
    <div>
      <Head>
        <title>Solana Scaffold</title>
        <meta name="description" content="Farms" />
      </Head>
      <div className="md:hero mx-auto p-4">
        <div className="md:hero-content flex flex-col">
          <h1 className="text-center text-5xl font-bold text-transparent bg-clip-text bg-gradient-to-tr from-[#9945FF] to-[#14F195]">
            Orca Farms
          </h1>
          {/* CONTENT GOES HERE */}
          <div className="overflow-x-auto">
            <table className="table w-full">
              <thead>
                <tr>
                  <th>Farm ID</th>
                  <th>LP Token</th>
                  <th>APY</th>
                  <th></th>
                </tr>
              </thead>
              <tbody>
                {farmsWithPool
                  .sort((a, b) =>
                    a.farmInfo.farmId
                      .toString()
                      .localeCompare(b.farmInfo.farmId.toString())
                  )
                  .map((farm) => (
                    <Farm
                      key={farm.farmInfo.farmId.toString()}
                      farm={farm}
                      pool={orcaPoolSetWithLpMintKey.get(
                        farm.farmInfo.baseTokenMint.toString()
                      )}
                    ></Farm>
                  ))}
              </tbody>
            </table>
          </div>
          <div className="text-center"></div>
        </div>
      </div>
    </div>
  );
};

export default OrcaFarms;
```

Except for the difference in protocol name (Raydium v.s Orca), the interface is almost identical. This means that **you can access all the protocols supported by Universal Rabbit Hole with the same universal interface!**

### Add `OrcaFarm`

Copy from `RaydiumFarm` component:

```bash!
$ cp src/components/RaydiumFarm.tsx src/components/OrcaFarm.tsx
```

Replace if with the following code snippet:

```typescript!
// src/components/OrcaFarm.tsx

import { FC, useCallback, useEffect, useState } from "react";
import { PublicKey } from "@solana/web3.js";
import { notify } from "utils/notifications";
import { useConnection, useWallet } from "@solana/wallet-adapter-react";
import useUserSOLBalanceStore from "stores/useUserSOLBalanceStore";
import { AnchorWallet } from "utils/anchorWallet";
import * as anchor from "@project-serum/anchor";
import { orca as protocol } from "@dappio-wonderland/navigator";
import {
  AddLiquidityParams,
  GatewayBuilder,
  HarvestParams,
  RemoveLiquidityParams,
  StakeParams,
  SupportedProtocols,
  SwapParams,
  UnstakeParams,
  WSOL,
} from "@dappio-wonderland/gateway";

interface FarmProps {
  farm: protocol.FarmInfoWrapper;
  pool: protocol.PoolInfoWrapper;
}

const protocolType = SupportedProtocols.Orca;

export const Farm: FC<FarmProps> = (props: FarmProps) => {
  const [apr, setApr] = useState(0);
  const { connection } = useConnection();
  const wallet = useWallet();
  const { getUserSOLBalance } = useUserSOLBalanceStore();

  // Get Farm
  const farm = props.farm;
  const farmInfo = farm.farmInfo;
  const farmId = farmInfo.farmId.toString();
  const lpMint = farmInfo.baseTokenMint.toString();
  const pool = props.pool;
  const poolInfo = pool.poolInfo;
  useEffect(() => {
    // NOTICE: We mocked LP price and reward price here just for demo
    const aprs = farm.getAprs(5, 1, 2);
    const apr = aprs.length > 1 ? aprs[1] : aprs[0];
    setApr(apr);
  }, []);

  const zapIn = useCallback(async () => {
    if (!wallet.publicKey) {
      console.error("error", "Wallet not connected!");
      notify({
        type: "error",
        message: "error",
        description: "Wallet not connected!",
      });
      return;
    }

    const provider = new anchor.AnchorProvider(
      connection,
      new AnchorWallet(wallet),
      anchor.AnchorProvider.defaultOptions()
    );
    const zapInAmount = 10000; // WSOL Amount

    // WSOL to tokenA
    const swapParams1: SwapParams = {
      protocol: SupportedProtocols.Jupiter,
      fromTokenMint: new PublicKey(WSOL),
      toTokenMint: poolInfo.tokenAMint,
      amount: zapInAmount,
      slippage: 1,
    };
    // tokenA to tokenB
    const swapParams2: SwapParams = {
      protocol: SupportedProtocols.Jupiter,
      fromTokenMint: poolInfo.tokenAMint,
      toTokenMint: poolInfo.tokenBMint,
      amount: 0, // Notice: amount needs to be updated later
      slippage: 1,
    };
    const addLiquidityParams: AddLiquidityParams = {
      protocol: protocolType,
      poolId: poolInfo.poolId,
    };
    const stakeParams: StakeParams = {
      protocol: protocolType,
      farmId: farmInfo.farmId,
    };

    const gateway = new GatewayBuilder(provider);

    // 1st Swap
    await gateway.swap(swapParams1);
    const minOutAmount1 = gateway.params.swapMinOutAmount.toNumber();

    // 2nd Swap
    swapParams2.amount = minOutAmount1 / 2;
    await gateway.swap(swapParams2);
    const minOutAmount2 = gateway.params.swapMinOutAmount.toNumber();

    // Add Liquidity
    addLiquidityParams.tokenInAmount = minOutAmount2;
    await gateway.addLiquidity(addLiquidityParams);

    // Stake
    await gateway.stake(stakeParams);

    await gateway.finalize();
    const txs = gateway.transactions();

    const recentBlockhash = (await connection.getLatestBlockhash()).blockhash;

    txs.forEach((tx) => {
      tx.recentBlockhash = recentBlockhash;
      tx.feePayer = wallet.publicKey;
    });

    const signTxs = await provider.wallet.signAllTransactions(txs);

    console.log("======");
    console.log("Txs are sent...");
    for (let tx of signTxs) {
      let sig: string = "";
      try {
        sig = await connection.sendRawTransaction(tx.serialize(), {
          skipPreflight: true,
          commitment: "confirmed",
        } as unknown as anchor.web3.SendOptions);
        await connection.confirmTransaction(sig, connection.commitment);

        notify({
          type: "success",
          message: "Transaction is executed successfully!",
          txid: sig,
        });
      } catch (error: any) {
        notify({
          type: "error",
          message: `Transaction failed!`,
          description: error?.message,
          txid: sig,
        });
        console.log(
          "NOTICE: paste the output to Transaction Inspector in Solana Explorer for debugging"
        );
        console.log(tx.serializeMessage().toString("base64"));
        console.error("error", `Transaction failed! ${error?.message}`, sig);
        break;
      }
    }
    console.log("Txs are executed");
    console.log("======");

    getUserSOLBalance(wallet.publicKey, connection);
  }, [wallet.publicKey, connection, getUserSOLBalance]);

  const zapOut = useCallback(async () => {
    if (!wallet.publicKey) {
      console.error("error", "Wallet not connected!");
      notify({
        type: "error",
        message: "error",
        description: "Wallet not connected!",
      });
      return;
    }

    const provider = new anchor.AnchorProvider(
      connection,
      new AnchorWallet(wallet),
      anchor.AnchorProvider.defaultOptions()
    );

    // Get share amount
    const ledgerKey = await protocol.infos.getFarmerId(
      farmInfo,
      provider.wallet.publicKey
    );
    const ledger = (await protocol.infos.getFarmer(
      connection,
      ledgerKey
    )) as protocol.FarmerInfo;
    const shareAmount = ledger.amount;
    const { tokenAAmount, tokenBAmount } = await pool.getTokenAmounts(
      shareAmount
    );
    const harvestParams: HarvestParams = {
      protocol: protocolType,
      farmId: farmInfo.farmId,
    };
    const unstakeParams: UnstakeParams = {
      protocol: protocolType,
      farmId: farmInfo.farmId,
      shareAmount,
    };
    const removeLiquidityParams: RemoveLiquidityParams = {
      protocol: protocolType,
      poolId: poolInfo.poolId,
    };
    // tokenB to tokenA
    const swapParams1: SwapParams = {
      protocol: SupportedProtocols.Jupiter,
      fromTokenMint: poolInfo.tokenBMint,
      toTokenMint: poolInfo.tokenAMint,
      amount: tokenBAmount, // swap coin to pc
      slippage: 3,
    };

    // tokenA to WSOL
    const swapParams2: SwapParams = {
      protocol: SupportedProtocols.Jupiter,
      fromTokenMint: poolInfo.tokenAMint,
      toTokenMint: new PublicKey(WSOL),
      amount: 0, // Notice: This amount needs to be updated later
      slippage: 10,
    };

    const gateway = new GatewayBuilder(provider);

    await gateway.harvest(harvestParams);
    await gateway.unstake(unstakeParams);
    await gateway.removeLiquidity(removeLiquidityParams);

    // 1st Swap
    await gateway.swap(swapParams1);
    const minOutAmount = gateway.params.swapMinOutAmount.toNumber();
    swapParams2.amount = minOutAmount + tokenAAmount;
    // 2nd Swap
    await gateway.swap(swapParams2);

    await gateway.finalize();
    const txs = gateway.transactions();

    const recentBlockhash = (await connection.getLatestBlockhash()).blockhash;

    txs.forEach((tx) => {
      tx.recentBlockhash = recentBlockhash;
      tx.feePayer = wallet.publicKey;
    });

    const signTxs = await provider.wallet.signAllTransactions(txs);

    console.log("======");
    console.log("Txs are sent...");
    for (let tx of signTxs) {
      let sig: string = "";
      try {
        sig = await connection.sendRawTransaction(tx.serialize(), {
          skipPreflight: true,
          commitment: "confirmed",
        } as unknown as anchor.web3.SendOptions);
        await connection.confirmTransaction(sig, connection.commitment);

        notify({
          type: "success",
          message: "Transaction is executed successfully!",
          txid: sig,
        });
      } catch (error: any) {
        notify({
          type: "error",
          message: `Transaction failed!`,
          description: error?.message,
          txid: sig,
        });
        console.log(
          "NOTICE: paste the output to Transaction Inspector in Solana Explorer for debugging"
        );
        console.log(tx.serializeMessage().toString("base64"));
        console.error("error", `Transaction failed! ${error?.message}`, sig);
        break;
      }
    }
    console.log("Txs are executed");
    console.log("======");

    getUserSOLBalance(wallet.publicKey, connection);
  }, [wallet.publicKey, connection, getUserSOLBalance]);

  return (
    <tr>
      <th>
        {farmId.slice(0, 5)}...{farmId.slice(farmId.length - 5)}
      </th>
      <td>
        {lpMint.slice(0, 5)}...{lpMint.slice(lpMint.length - 5)}
      </td>
      <td>{apr.toFixed(2) + "%"}</td>
      <td>
        <button className="btn btn-info" onClick={zapIn}>
          Zap In
        </button>
        &nbsp;
        <button className="btn btn-warning" onClick={zapOut}>
          Zap Out
        </button>
      </td>
    </tr>
  );
};
```

Here, you can still feel the power of Gateway: The only change you need to make, is changing protocol type from `Raydium` to `Orca`! The rest of the code are almost identical.

### Update `ContentContainer`

Let's add another link for `orcaFarms`:

```typescript!
// src/components/ContentContainer.tsx 

// #L32
<li>
  <Link href="/orcaFarms">
    <a>Orca Farms</a>
  </Link>
</li>
```

### Compile and Run

```
$ yarn dev
```

Then open http://localhost:3000 (default).

## Part 3: Add Tulip Vaults

In the final part of the guide, we will demonstrate how simple it is to **compose** 2 different protocols into one continuous operation.

### Update `NavigatorProvider`

Let's start from replacing `NavigatorProvider.tsx` with the following code snippet:

```typescript!
// src/contexts/NavigatorProvider.tsx

import { useConnection } from "@solana/wallet-adapter-react";
import {
  createContext,
  FC,
  ReactNode,
  useContext,
  useEffect,
  useState,
} from "react";
// Added from Part 3
import { raydium, orca, tulip } from "@dappio-wonderland/navigator";

export interface NavigatorContextState {
  raydiumFarms: raydium.FarmInfoWrapper[];
  raydiumPoolSetWithLpMintKey: Map<string, raydium.PoolInfoWrapper>;
  orcaFarms: orca.FarmInfoWrapper[];
  orcaPoolSetWithLpMintKey: Map<string, orca.PoolInfoWrapper>;
  // Added from Part 3
  tulipVaults: tulip.VaultInfoWrapper[];
}

export const NavigatorContext = createContext<NavigatorContextState>(
  {} as NavigatorContextState
);

export function useNavigator(): NavigatorContextState {
  return useContext(NavigatorContext);
}

export const NavigatorProvider: FC<{ children: ReactNode }> = ({
  children,
}) => {
  const { connection } = useConnection();
  const [raydiumFarms, setRaydiumFarms] = useState<raydium.FarmInfoWrapper[]>(
    []
  );
  const [raydiumPoolSetWithLpMintKey, setRaydiumPoolSetWithLpMintKey] =
    useState<Map<string, raydium.PoolInfoWrapper>>(
      {} as Map<string, raydium.PoolInfoWrapper>
    );
  const [orcaFarms, setOrcaFarms] = useState<orca.FarmInfoWrapper[]>([]);
  const [orcaPoolSetWithLpMintKey, setOrcaPoolSetWithLpMintKey] = useState<
    Map<string, orca.PoolInfoWrapper>
  >({} as Map<string, orca.PoolInfoWrapper>);

  // Added from Part 3
  const [tulipVaults, setTulipVaults] = useState<tulip.VaultInfoWrapper[]>([]);

  useEffect(() => {
    {
      const getAllFarmsWrappers = async () => {
        return (await raydium.infos.getAllFarmWrappers(
          connection
        )) as raydium.FarmInfoWrapper[];
      };

      getAllFarmsWrappers().then((wrappers) => {
        setRaydiumFarms(wrappers);
      });

      const getAllPoolWrappers = async () => {
        const poolWrappers = await raydium.infos.getAllPoolWrappers(connection);

        return new Map<string, raydium.PoolInfoWrapper>(
          poolWrappers.map((wrapper) => [
            wrapper.poolInfo.lpMint.toString(),
            wrapper as raydium.PoolInfoWrapper,
          ])
        );
      };

      getAllPoolWrappers().then((poolSetResult) => {
        setRaydiumPoolSetWithLpMintKey(poolSetResult);
      });
    }
    {
      const getAllFarmsWrappers = async () => {
        return (await orca.infos.getAllFarmWrappers(
          connection
        )) as orca.FarmInfoWrapper[];
      };

      getAllFarmsWrappers().then((wrappers) => {
        setOrcaFarms(wrappers);
      });

      const getAllPoolWrappers = async () => {
        const poolWrappers = await orca.infos.getAllPoolWrappers(connection);

        return new Map<string, orca.PoolInfoWrapper>(
          poolWrappers.map((wrapper) => [
            wrapper.poolInfo.lpMint.toString(),
            wrapper as orca.PoolInfoWrapper,
          ])
        );
      };

      getAllPoolWrappers().then((poolSetResult) => {
        setOrcaPoolSetWithLpMintKey(poolSetResult);
      });
    }
    // *************************
    // Added from Part 3 (Begin)
    {
      const getAllVaultsWrappers = async () => {
        return (await tulip.infos.getAllVaultWrappers(
          connection
        )) as tulip.VaultInfoWrapper[];
      };
  
      getAllVaultsWrappers().then((wrappers) => {
        setTulipVaults(wrappers);
      });
    }
    // Added from Part 3 (End)
    // *************************
  }, []);

  return (
    <NavigatorContext.Provider
      value={{
        raydiumFarms,
        raydiumPoolSetWithLpMintKey,
        orcaFarms,
        orcaPoolSetWithLpMintKey,
        // Added from Part 3
        tulipVaults,
      }}
    >
      {children}
    </NavigatorContext.Provider>
  );
};
```

What we do in above code snippet is fetching vaults from Tulip by **one line of code** (`tulip.infos.getAllVaultWrappers`)

### Add `tulipVaults`

Copy from `raydiumFarms`:

```
$ cp src/pages/raydiumFarms.tsx src/pages/tulipVaults.tsx
```

Replace `tulipVaults.tsx` with the following code snippet:

```typescript!
// src/pages/tulipVaults.tsx

import { useNavigator } from "contexts/NavigatorProvider";
import { NextPage } from "next";
import Head from "next/head";
import { useEffect, useState } from "react";
import { tulip as protocol } from "@dappio-wonderland/navigator";
import { Vault } from "components/TulipVault";

export const TulipVaults: NextPage = (props) => {
  const { tulipVaults, raydiumPoolSetWithLpMintKey } = useNavigator();
  const [vaultsWithPool, setVaultsWithPool] = useState<
    protocol.VaultInfoWrapper[]
  >([]);

  useEffect(() => {
    const vaultsWithPool = tulipVaults.filter((vault) => {
      return raydiumPoolSetWithLpMintKey.size > 0
        ? raydiumPoolSetWithLpMintKey.has(
            vault.vaultInfo.base.underlyingMint.toString()
          )
        : false;
    });
    setVaultsWithPool(vaultsWithPool);
  }, [raydiumPoolSetWithLpMintKey]);

  return (
    <div>
      <Head>
        <title>Solana Scaffold</title>
        <meta name="description" content="Farms" />
      </Head>
      <div className="md:hero mx-auto p-4">
        <div className="md:hero-content flex flex-col">
          <h1 className="text-center text-5xl font-bold text-transparent bg-clip-text bg-gradient-to-tr from-[#9945FF] to-[#14F195]">
            Tulip Vaults
          </h1>
          {/* CONTENT GOES HERE */}
          <div className="overflow-x-auto">
            <table className="table w-full">
              <thead>
                <tr>
                  <th>Vault ID</th>
                  <th>LP Token</th>
                  <th></th>
                </tr>
              </thead>
              <tbody>
                {vaultsWithPool
                  .sort((a, b) =>
                    a.vaultInfo.vaultId
                      .toString()
                      .localeCompare(b.vaultInfo.vaultId.toString())
                  )
                  .map((vault) => (
                    <Vault
                      key={vault.vaultInfo.vaultId.toString()}
                      vault={vault}
                      pool={raydiumPoolSetWithLpMintKey.get(
                        vault.vaultInfo.base.underlyingMint.toString()
                      )}
                    ></Vault>
                  ))}
              </tbody>
            </table>
          </div>
          <div className="text-center"></div>
        </div>
      </div>
    </div>
  );
};

export default TulipVaults;
```

Since we're going to implement deposit to and withdraw from Tulip vualt, `farm` will no longer be used.

As a result:
- We replace `RaydiumFarm` to `TulipVault`, `raydiumFarm` to `tulipVault`, `Farm` to `Vault`, and `farm` to `vault`.
- We change the properties from `poolLpTokenAccount.mint` to `base.underlyingMint`.

### Add `TulipVault`

Copy from `RaydiumFarm`:

```
$ cp src/components/RaydiumFarm.tsx src/components/TulipVault.tsx
```

Replace `TulipVault.tsx` with the following code snippet:

```typescript!
// src/components/TulipVault.tsx

import { FC, useCallback } from "react";
import { PublicKey } from "@solana/web3.js";
import { notify } from "utils/notifications";
import { useConnection, useWallet } from "@solana/wallet-adapter-react";
import useUserSOLBalanceStore from "stores/useUserSOLBalanceStore";
import { AnchorWallet } from "utils/anchorWallet";
import * as anchor from "@project-serum/anchor";
import { raydium, tulip } from "@dappio-wonderland/navigator";
import {
  AddLiquidityParams,
  GatewayBuilder,
  RemoveLiquidityParams,
  DepositParams,
  SupportedProtocols,
  SwapParams,
  WithdrawParams,
  WSOL,
} from "@dappio-wonderland/gateway";

interface VaultProps {
  vault: tulip.VaultInfoWrapper;
  pool: raydium.PoolInfoWrapper;
}

export const Vault: FC<VaultProps> = (props: VaultProps) => {
  const { connection } = useConnection();
  const wallet = useWallet();
  const { getUserSOLBalance } = useUserSOLBalanceStore();

  // Get Farm
  const vault = props.vault;
  const vaultInfo = vault.vaultInfo;
  const vaultId = vaultInfo.vaultId.toString();
  const lpMint = vaultInfo.base.underlyingMint.toString();
  const pool = props.pool;
  const poolInfo = pool.poolInfo;

  const zapIn = useCallback(async () => {
    if (!wallet.publicKey) {
      console.error("error", "Wallet not connected!");
      notify({
        type: "error",
        message: "error",
        description: "Wallet not connected!",
      });
      return;
    }

    const provider = new anchor.AnchorProvider(
      connection,
      new AnchorWallet(wallet),
      anchor.AnchorProvider.defaultOptions()
    );
    const zapInAmount = 10000; // WSOL Amount

    // WSOL to tokenA
    const swapParams1: SwapParams = {
      protocol: SupportedProtocols.Jupiter,
      fromTokenMint: new PublicKey(WSOL),
      toTokenMint: poolInfo.tokenAMint,
      amount: zapInAmount,
      slippage: 1,
    };
    // tokenA to tokenB
    const swapParams2: SwapParams = {
      protocol: SupportedProtocols.Jupiter,
      fromTokenMint: poolInfo.tokenAMint,
      toTokenMint: poolInfo.tokenBMint,
      amount: 0, // Notice: amount needs to be updated later
      slippage: 1,
    };
    const addLiquidityParams: AddLiquidityParams = {
      protocol: SupportedProtocols.Raydium,
      poolId: poolInfo.poolId,
    };
    const depositParams: DepositParams = {
      protocol: SupportedProtocols.Tulip,
      vaultId: vaultInfo.vaultId,
      depositAmount: 0, // Notice: amount will auto-update in gateway state after add liquidity
    };

    const gateway = new GatewayBuilder(provider);

    // 1st Swap
    await gateway.swap(swapParams1);
    const minOutAmount1 = gateway.params.swapMinOutAmount.toNumber();

    // 2nd Swap
    swapParams2.amount = minOutAmount1 / 2;
    await gateway.swap(swapParams2);
    const minOutAmount2 = gateway.params.swapMinOutAmount.toNumber();

    // Add Liquidity
    addLiquidityParams.tokenInAmount = minOutAmount2;
    await gateway.addLiquidity(addLiquidityParams);

    // Stake
    await gateway.deposit(depositParams);

    await gateway.finalize();
    const txs = gateway.transactions();

    const recentBlockhash = (await connection.getLatestBlockhash()).blockhash;

    txs.forEach((tx) => {
      tx.recentBlockhash = recentBlockhash;
      tx.feePayer = wallet.publicKey;
    });

    const signTxs = await provider.wallet.signAllTransactions(txs);

    console.log("======");
    console.log("Txs are sent...");
    for (let tx of signTxs) {
      let sig: string = "";
      try {
        sig = await connection.sendRawTransaction(tx.serialize(), {
          skipPreflight: false,
          commitment: "confirmed",
        } as unknown as anchor.web3.SendOptions);
        await connection.confirmTransaction(sig, connection.commitment);

        notify({
          type: "success",
          message: "Transaction is executed successfully!",
          txid: sig,
        });
      } catch (error: any) {
        notify({
          type: "error",
          message: `Transaction failed!`,
          description: error?.message,
          txid: sig,
        });
        console.log(
          "NOTICE: paste the output to Transaction Inspector in Solana Explorer for debugging"
        );
        console.log(tx.serializeMessage().toString("base64"));
        console.error("error", `Transaction failed! ${error?.message}`, sig);
        break;
      }
    }
    console.log("Txs are executed");
    console.log("======");

    getUserSOLBalance(wallet.publicKey, connection);
  }, [wallet.publicKey, connection, getUserSOLBalance]);

  const zapOut = useCallback(async () => {
    if (!wallet.publicKey) {
      console.error("error", "Wallet not connected!");
      notify({
        type: "error",
        message: "error",
        description: "Wallet not connected!",
      });
      return;
    }

    const provider = new anchor.AnchorProvider(
      connection,
      new AnchorWallet(wallet),
      anchor.AnchorProvider.defaultOptions()
    );

    const depositorId = tulip.infos.getDepositorId(
      vaultInfo.vaultId,
      wallet.publicKey
    );
    const depositor = (await tulip.infos.getDepositor(
      connection,
      depositorId
    )) as tulip.DepositorInfo;

    const shareAmount = Math.floor(Number(depositor.shares) / 10);

    const withdrawParams: WithdrawParams = {
      protocol: SupportedProtocols.Tulip,
      vaultId: vaultInfo.vaultId,
      withdrawAmount: shareAmount,
    };
    const removeLiquidityParams: RemoveLiquidityParams = {
      protocol: SupportedProtocols.Raydium,
      poolId: poolInfo.poolId,
    };

    const { tokenAAmount: coinAmount } = pool.getTokenAmounts(shareAmount);
    // tokenA to tokenB
    const swapParams1: SwapParams = {
      protocol: SupportedProtocols.Jupiter,
      fromTokenMint: poolInfo.tokenAMint,
      toTokenMint: poolInfo.tokenBMint,
      amount: coinAmount, // swap coin to pc
      slippage: 3,
    };

    // tokenB to WSOL
    const swapParams2: SwapParams = {
      protocol: SupportedProtocols.Jupiter,
      fromTokenMint: poolInfo.tokenBMint,
      toTokenMint: new PublicKey(WSOL),
      amount: 0, // Notice: This amount needs to be updated later
      slippage: 3,
    };

    const gateway = new GatewayBuilder(provider);

    await gateway.withdraw(withdrawParams);
    await gateway.removeLiquidity(removeLiquidityParams);

    // 1st Swap
    await gateway.swap(swapParams1);
    const minOutAmount = gateway.params.swapMinOutAmount.toNumber();
    swapParams2.amount = minOutAmount;

    // 2nd Swap
    if (!poolInfo.tokenBMint.equals(WSOL)) {
      await gateway.swap(swapParams2);
    }

    await gateway.finalize();
    const txs = gateway.transactions();

    const recentBlockhash = (await connection.getLatestBlockhash()).blockhash;

    txs.forEach((tx) => {
      tx.recentBlockhash = recentBlockhash;
      tx.feePayer = wallet.publicKey;
    });

    const signTxs = await provider.wallet.signAllTransactions(txs);

    console.log("======");
    console.log("Txs are sent...");
    for (let tx of signTxs) {
      let sig: string = "";
      try {
        sig = await connection.sendRawTransaction(tx.serialize(), {
          skipPreflight: false,
          commitment: "confirmed",
        } as unknown as anchor.web3.SendOptions);
        await connection.confirmTransaction(sig, connection.commitment);

        notify({
          type: "success",
          message: "Transaction is executed successfully!",
          txid: sig,
        });
      } catch (error: any) {
        notify({
          type: "error",
          message: `Transaction failed!`,
          description: error?.message,
          txid: sig,
        });
        console.log(
          "NOTICE: paste the output to Transaction Inspector in Solana Explorer for debugging"
        );
        console.log(tx.serializeMessage().toString("base64"));
        console.error("error", `Transaction failed! ${error?.message}`, sig);
        break;
      }
    }
    console.log("Txs are executed");
    console.log("======");

    getUserSOLBalance(wallet.publicKey, connection);
  }, [wallet.publicKey, connection, getUserSOLBalance]);

  return (
    <tr>
      <th>
        {vaultId.slice(0, 5)}...{vaultId.slice(vaultId.length - 5)}
      </th>
      <td>
        {lpMint.slice(0, 5)}...{lpMint.slice(lpMint.length - 5)}
      </td>
      <td>
        <button className="btn btn-info" onClick={zapIn}>
          Zap In
        </button>
        &nbsp;
        <button className="btn btn-warning" onClick={zapOut}>
          Zap Out
        </button>
      </td>
    </tr>
  );
};
```

You will be surprised by the extreme simplicity of the interface. **To compose 2 actions from 2 protocols, the only thing you have to do is specifying the protocol type and action.** That's it!

```typescript!
const gateway = new GatewayBuilder(provider);

await gateway.swap(swapParams1);

// ...

await gateway.swap(swapParams2);

// ...

await gateway.addLiquidity(addLiquidityParams);

// ...

await gateway.deposit(depositParams);

await gateway.finalize();
const txs = gateway.transactions();
```

### Update `ContentContainer`

Let's add a link for `tulipVaults`:

```typescript!
// src/components/ContentContainer.tsx 

// #L39
<li>
  <Link href="/tulipVaults">
    <a>Tulip Vaults</a>
  </Link>
</li>
```

### Compile and Run

```bash
$ yarn dev
```

Then open http://localhost:3000 (default).

## Summary

- The core logic is less than 100 lines of code
  - `RaydiumFarm`
  - `OrcaFarm`
  - `TulupVault`
- See how simple it is to use gateway to make your own aggregator
- What's next?
  - Develop higher level DeFi composability
  - Explore other supported protocols
  - Compose!

