# @biconomy/sdk Testing Framework

## Testing Setup

> **Note**:  
> - Tests now must be run with node version >= v22

### Network Agnostic Tests
- Tests are executed against locally deployed ephemeral Anvil chains (each with a different ID) with relevant contracts pre-deployed for each test.
- Bundlers for testing are instantiated using [prool](https://github.com/wevm/prool), currently utilizing alto instances. We plan to switch to Biconomy's bundlers when they become available via `prool`.

### Deployment Configuration
A custom script `bun run fetch:deployment` is provided to search for the bytecode of deployed contracts from a customizable location (default: `../../nexus/deployments`). This folder is **auto-generated** in Nexus whenever a new Hardhat deployment is made, ensuring that the SDK remains up-to-date with the latest contract changes.

The script performs the following:
- **ABIs**: Moved to `./src/__contracts/{name}Abi.ts`
- **Addresses**: Moved to `./src/addresses.ts`
- **Additional Fixtures**: Copied to `tests__/contracts`

The script accepts a number of args from the command line:
  - nexusDeploymentPath (default: `"../node_modules/nexus/deployments"`)
  - chainName (default: `"anvil-55000"`)
  - forSrc (default: `["K1ValidatorFactory", "Nexus", "K1Validator"]`);

Example usage:
```bash
bun run fetch:deployment:raw --chainName=anvil-52878 -forSrc=K1Validator -forSrc=Nexus --nexusDeploymentPath=../../nexus/deployments
bun run lint --apply-unsafe
```

> **Note**:  
> - Do not edit these files manually; they will be overridden if/when a new Nexus deployment occurs.
> - Avoid hardcoding important addresses (e.g., `const k1ValidatorAddress = "0x"`). Use `./src/addresses.ts` instead.

## Network Scopes for Tests

To prevent tests from conflicting with one another, tests can be scoped to different networks in different ways.

### Global Scope
- Use by setting `const NETWORK_TYPE: TestFileNetworkType = "COMMON_LOCALHOST"` at the top of the test file.
- Suitable when you're sure that tests in the file will **not** conflict with other tests using the common localhost network.

### Local Scope
- Use by setting `const NETWORK_TYPE: TestFileNetworkType = "FILE_LOCALHOST"` for test files that may conflict with others.
- Networks scoped locally are isolated to the file in which they are used.
- Tests within the same file using a local network may conflict with each other. If needed, split tests into separate files or use the Test Scope.

### Test Scope
- A network is spun up *only* for the individual test in which it is used. Access this via the `localhostTest`/`testnetTest` helpers in the same file as `"COMMON_LOCALHOST"` or `"FILE_LOCALHOST"` network types.

Example usage:
```ts
localhostTest("should be used in the following way", async({ config: { bundlerUrl, chain, fundedClients }}) => {
    // chain, bundlerUrl spun up just in time for this test only...
    expect(await fundedClients.smartAccount.getAccountAddress()).toBeTruthy();
});
```

> **Note:** 
> Please avoid using multiple nested describe blocks in a single test file, as it is unnecessary and can lead to confusion regarding network scope.
> Using *many* test files is preferable, as describe blocks run in parallel. 

## Testing on Testnets or New Chains
- There is currently one area where SDK tests can be run against a remote testnet: the playground
- You can run the playground using the command: `bun run playground`. They playground is automatically ommitted from CICD.
- Additionally there are helpers for running tests on files on a public testnet:
    - `const NETWORK_TYPE: TestFileNetworkType = "TESTNET"` will pick up relevant configuration from environment variables, and can be used at the top of a test file to have tests run against the specified testnet instead of the localhost
    - If you want to run a single test on a public testnet *from inside a different describe block* you can use the: `testnetTest` helper:

Example usage:
```ts
testnetTest("should be used in the following way", async({ config: { bundlerUrl, chain, account }}) => {
    // chain, bundlerUrl etc taken from environment variables...
    expect(account).toBeTruthy(); // from private key, please ensure it is funded if sending txs
});
```

> **Note:** 
> As testnetTest runs against a public testnet the account related to the privatekey (in your env var) must be funded, and the testnet is not 'ephemeral', meaning state is obviously persisted on the testnet after the test finishes. 

- The playground does not run in CI/CD but can be triggered manually from the GitHub Actions UI or locally via bun run playground.
- The playground network is configured with environment variables:
    - PRIVATE_KEY
    - CHAIN_ID
    - RPC_URL (optional, inferred if unset)
    - BUNDLER_URL (optional, inferred if unset)
    - PAYMASTER_URL (tests skipped if unset)

## Debugging and Client Issues
It is recommended to use the playground for debugging issues with clients. Please refer to the following guidelines for escalation and handover: [Debugging Client Issues](https://www.notion.so/biconomy/Debugging-Client-Issues-cc01c1cab0224c87b37a4d283370165b)

## Testing Helpers
A [testClient](https://viem.sh/docs/clients/test#extending-with-public--wallet-actions) is available (funded and extended with walletActions and publicActions) during testing. Please use it as a master Client for all things network related. 

