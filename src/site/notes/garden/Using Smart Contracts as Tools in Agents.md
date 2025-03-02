---
{"dg-publish":true,"permalink":"/garden/using-smart-contracts-as-tools-in-agents/","created":"2025-02-27T13:58:51.602+01:00","updated":"2025-03-02T15:04:51.952+01:00"}
---

# Using Smart Contracts as Tools in Agents
This POC experiments with using smart contracts as tools for agents. A common problem with agents now is creating the the tools they interact with. They still require a lot of manual work, like signing up for the API service it's using, setting up billing and API keys. 

Smart contracts allow transparent pricing (each interaction is paid vs. complicated pricing tiers or credit systems) and can be fully automated (no account/ API keys are required). The documentation also can be structured easier.

In this example we are exploring creating a whole agentic system that only interacts with smart contracts as tools. We are using [Thirdweb](https://thirdweb.com) and the [Vercel AI SDK](https://sdk.vercel.ai/docs/introduction) for this.

---

# Log

## *1.3.2025 1:45am CET:* First experiments

Managed to create a function to turn a smart contract address into a fully usable OpenAI function format based on it's ABI. This confirms that the idea is doable.

The main concern now is: How do we execute the actual function? Right now we can only use the existing functionalities of Solidity, which limits the usability. But we need to be able to access a database and do everything a normal API would do. The easiest way would obviously just to call the actual API from the smart contract (while verifying the smart contract did the call) and only use the smart contract as "gateway".

There is already pre-built solutions to call API endpoints "from a smart contract". This is how they (according to my research) work:

1. The smart contract is called on-chain from a wallet
2. The backend of the provider, like [Chainlink](https://chain.link/), watches for the events (invocations) of the smart contract. You can see them on-chain at Etherscan. See this [example log](https://sepolia.etherscan.io/address/0x254a6c4eb7bE8D1AC3C07F14BEB4C83bca2aCa43#events).
3. Once a new event is logged, the provider knows it went through successfully and then calls the webhook that the user configured it to.

But just telling users to use Chainlink would be boring and rely on a thirdparty service, so we will implement this by hand. We will also have to send back the result to the agent for it to be used in reasoning. The best idea is to just provide a polling API endpoint to poll the response of the API/ tool based on the transaction ID returned by the smart contract call.

## *1.3.2025 11am CET:* Adding backend
The main logic of the backend is super basic. I was able to setup a simple event handler to watch a smart contract's event based on `watchContractEvents` in the Thirdweb SDK.  It allows listening and printing out all the events on a contract and gives us this data:
- `eventName` - The "function" called in the contract
- `arguments` - The returned values (eg. the random number generated and requestId)
- `address` - The contract address
- `transactionHash` - The has of the transaction to get more metadata

To now get things running, I made a simple flow to fetch a smart contract and save it as OpenAI function into a database.

![Screenshare of the flow](https://youtu.be/4bg53iihLNk)

With this info saved I made a small background worker that just listens for an event. It dynamically listens for all contracts that are saved in the database and then uses the contract address to match it to the stored contract. Then it stores an "invocation".

![Pasted image 20250302145026.png](/img/user/Pasted%20image%2020250302145026.png)
![Pasted image 20250302145042.png](/img/user/Pasted%20image%2020250302145042.png)

See the images above with an example. The lower images shows a script running that calls the contract and prints the returned random number, while the upper one is the mentioned background worker. If we check the database now we can see the contract (added before running the worker and test):

and we can see the invocation:

