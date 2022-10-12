---
tags: devgitbook
---

# Architecture


There are two modules in Universal Rabbit Hole: **Navigator** and **Gateway**:

![](https://hackmd.io/_uploads/rJbWYMd-o.jpg)

#### Navigator (Reader)

Navigator is a Typescript client for instantiating various of kinds of DeFi protocols. You can use it as a standalone dependency in your own project or together with [Dappio Gateway](https://guide.dappio.xyz/the-universal-rabbit-hole).

#### Gateway (Writer)

Gateway is a **CaaS (Compasaility-as-a-Service)** that standardizes inter-protocol interaction on Solana to unlock the potential of composability. It has an universal interface for various Solana DeFi elements (Pool / Farm / MoneyMarket / Vault / Leveraged Farm / ...) and serves as a **common knowledge base** that helps Solana community learn and improve.

### Anatomy of Gateway

![](https://hackmd.io/_uploads/Skbcueoyi.jpg)

Let's take a closer look on Gateway module. There are 4 different components in order to make Gateway function:
  - **Builder**: Off-chain component that helps composing different DeFi actions
  - **Protocol(s)**: Off-chain component that packages the specific instruction set of each protocol
  - **Gateway**: On-chain program that receives and dispatches all the transactions, manages state, distributes fees
  - **Adapter(s)**: On-chain program that connects base program and Gateway program

While **Builder** and **Protocols** are packaged into a npm module, **Gateway (program)** and **Adapters** are Solana programs deployed by Dappio (sometimes third-party) to be invoked to interact with Base programs

### Workflow of Gateway

![](https://hackmd.io/_uploads/Bkvs9tU1i.png)

We can separate these series of operations into 2 distinct sections: **off-chain part** and **on-chain part**.

#### Off-chain Part

- User assembles actions by invoking action setter and providing proper parameters
- Builder composes transactions

#### On-chain Part

- User sends transactions directly to Gateway program
- Gateway dispatches transactions to their corresponding adapters through CPI (cross-program invocation)
- Adapter invokes Base program through another CPI