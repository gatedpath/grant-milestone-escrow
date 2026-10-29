# Grant milestone escrow

A funder deposits stablecoin, a grantee proves eligibility, a reviewer releases tranches with an on-chain note.

<!-- assets/rehearsal.gif goes here: the twelve-step rehearsal on a local fork, 7 s -->

1 contract · 21 tests · 12-step rehearsal in 7 s · deployed on testnet in 1 block · 1,476,223 gas · 450 test RBNT · verified on Routescan

```sh
npm run dev -- --no-web --port 8549 --request-id 100   # forks testnet locally, seeds five wallets in the five credential states
scripts/rehearse-escrow.sh                              # the whole life of a grant, twelve steps, nothing signed, no key anywhere
cd contracts && forge script script/DeployGrantEscrow.s.sol --rpc-url redbelly_testnet --account <keystore-name> --sender <your-address> --broadcast
                                                        # the one place a key is used: yours, in your keystore, never read by anything here
```

**The gate is in the contract, not the wallet.** `accept` asks the network's identity verifier about
the grantee against request 100, the accredited-issuer recipe, and refuses a credential that was never
issued, has expired, was revoked or is from the wrong jurisdiction. A wallet cannot talk its way past
that, and neither can a front end.

**The check runs on every release.** A grantee whose credential is revoked after two tranches
does not get the third. The money waits in the escrow until the reviewer cancels, and the funder gets it
back with a note on chain saying why.

**Nothing deploys without the ship report.** Pre-flight, the gas report, the tests in all five credential
states and the Slither check write one file. The deploy script will not broadcast on mainnet without that
file dated today, and will not take a mock verifier there at all. On testnet it warns and lets you learn.

## Live on testnet

Chain 153, deployed and verified on 23 October 2026.

| | |
|---|---|
| `GrantEscrow` | [`0x57B3848Dabd2488495Af156e7b198f19eF32D395`](https://redbelly.testnet.routescan.io/address/0x57B3848Dabd2488495Af156e7b198f19eF32D395/contract/153/code) |
| deploy transaction | `0x162dcbfe7ae40c250ca6f307cabaee341b2faa13cf07807bc6f9971cc2e91519`, block 3,055,920 |
| gas | 1,476,223 at a base fee of 304,859 gwei, 450.04 test RBNT, about 70 US cents at that day's feed price |
| verifier | [`0x2E652B946DC303d8f53bC92D0787e13B109003e5`](https://redbelly.testnet.routescan.io/address/0x2E652B946DC303d8f53bC92D0787e13B109003e5/contract/153/code), a `ReceptorMock` I share with the gated token project |
| eligibility request | 100, the accredited-issuer recipe in `recipes/accredited-issuer/` |
| record and report | `contracts/deployments/153-GrantEscrow.json`, `contracts/deployments/ship-153-2026-10-25.md` (the report re-run read-only with 0.2.3 on 25 October; the 0.2.1 report that gated the deploy on the 23rd is in my private record) |

The verifier is a mock because testnet has no real verifier for this recipe yet. Every wallet starts
ineligible and the mock's admin decides. Nothing here is on mainnet, and I will not put a mock there;
the deploy script refuses it for me.

## Install

| You need | Why |
|---|---|
| Node 22 or later | The scripts and the web app; `npm run doctor` checks it |
| [Foundry](https://getfoundry.sh) | `forge`, `anvil` and `cast`: build, test, fork, deploy |
| A wallet verified on the network, for step 4 only | The network refuses any other sender, and so does the script's own pre-flight |

`npm run doctor` checks the first two and tells you what is missing, one line each with the fix.

1. **Install**, once per clone:

   ```sh
   npm install
   npm run contracts:install
   ```

2. **Test**:

   ```sh
   npm run test        # 89 tests: the scaffold's 68 and 21 for the escrow, one a fuzz over tranches
   ```

3. **Rehearse** on a local fork, two shells, nothing signed:

   ```sh
   npm run dev -- --no-web --port 8549 --request-id 100
   ```

   ```sh
   scripts/rehearse-escrow.sh
   ```

   A test stablecoin, the escrow against the seeded verifier, a 30,000 grant in three tranches, the
   grantee's acceptance, two releases with notes, three refusals (release before acceptance, release by
   the funder, acceptance by a revoked wallet), a cancelled second grant with the money back, and the
   third tranche held when the grantee's credential is revoked. Save the output outside the project:
   transaction hashes look like keys to the secret scanner.

4. **Deploy** to testnet with your own keystore. Make it once with `cast wallet import <keystore-name>
   --interactive`, verify that wallet on the network, fund it from the faucet, then:

   ```sh
   npm run preflight
   npm run ship -- --chain 153 --account <keystore-name>
   cd contracts && forge script script/DeployGrantEscrow.s.sol --rpc-url redbelly_testnet --account <keystore-name> --sender <your-address> --broadcast
   ```

   Set `VERIFIER` to a verifier contract you already have and the script uses it, which is what I did.
   Leave it unset and the script deploys a mock and says so in capitals. `REVIEWER` defaults to the
   deployer.

## The flow

| When | Command | What you get |
|---|---|---|
| once per machine | `npm run doctor` | What is missing, with the fix |
| once per clone | `npm install && npm run contracts:install` | The contracts and the web app ready to build |
| every change | `npm run test` | 89 tests, every gated path in all five credential states |
| before a deploy | `npm run dev` and `scripts/rehearse-escrow.sh` | The whole life of a grant on a fork, in seven seconds |
| before a deploy | `npm run ship -- --chain 153 --account <keystore-name>` | One report: pre-flight, gas, tests, analysers. On mainnet, the gate |
| the deploy | `forge script ... --broadcast` | Here: one block, 1,476,223 gas, 450 test RBNT |
| after the deploy | `forge verify-contract ...` | Source on Routescan; the record carries the arguments already encoded |
| in use | `fund`, `accept`, `release`, `cancel` | A grant whose every step is an event on chain |

<details>
<summary>What the contract does, action by action</summary>

- **`fund(grantee, token, amount, tranches)`**, anyone. Pulls `amount` of an ERC-20 from the funder and
  records the grant: grantee, token, tranches, which must add up to the amount. A zero tranche is refused.
- **`accept(grantId)`**, the grantee only, and gated. Asks the verifier about the grantee's wallet against
  request 100 before any money moves.
- **`release(grantId, note)`**, the reviewer only. Sends the next tranche and emits the note. Refused
  before acceptance, refused from anyone else, refused with an empty note, and refused again if the
  credential has been revoked since.
- **`cancel(grantId, note)`**, the reviewer only. Returns what is left to the funder, with a note.

Every grant, milestone, note and remainder can be read back, and every action emits an event. The escrow
records the amount it was asked to move, not the amount that arrived. A stablecoin that takes a fee on
transfer or rebases would break that; the contract assumes a plain one.

</details>

<details>
<summary>Why this network, and what the escrow would be anywhere else</summary>

The escrow asks the chain's own identity layer whether the grantee is who the funder required, at
acceptance and again at every release. No front end can do that for it.

| The same escrow | Who checks the grantee | What a revoked credential does |
|---|---|---|
| on a chain with no identity layer | nobody, or a front end anyone can bypass | nothing |
| with an allow-list the funder keeps | the funder, by hand, off chain | whatever the funder remembers to do |
| here | the contract, against the network's verifier, on every release | the next tranche is refused, on chain |

</details>

## Docs

| | |
|---|---|
| 01. The run record | `docs/01-run-record.md`: what was built, what the fork run found, what was not done |
| 02. The deploy | `contracts/deployments/ship-153-2026-10-25.md`: the ship report, every check and figure, re-run with 0.2.3 against the deployed address |
| 03. The rehearsal | `docs/03-rehearsal.txt`: the twelve steps as they ran, gas and status per transaction |
| 04. The threat model | `THREAT-MODEL.md` |
| 05. The recipes | `recipes/README.md`: the eligibility recipes the kit ships, and why this escrow uses `accredited-issuer` |

## Development

```sh
git clone https://github.com/gatedpath/grant-milestone-escrow && cd grant-milestone-escrow
npm install && npm run contracts:install
npm run test
```

Issues and pull requests are welcome; `CONTRIBUTING.md` is short. If you have found a security problem,
please read `SECURITY.md` first. This is built on my kit at `npm create redbelly-dapp` and it is an
independent project: Redbelly did not make it and is not associated with it. Their developer portal is
[Vine](https://vine.redbelly.network), and that is where the network's own facts live. MIT.

<details>
<summary>The scaffold's own README, as generated</summary>

<!-- the template README goes here, unchanged except the title -->

</details>
