Directory structure:
└── react-native/
│   ├── best-practices.mdx
│   ├── eip5792.mdx
│   ├── installation.mdx
│   ├── link-mode.mdx
│   ├── mobile-linking.mdx
│   ├── one-click-auth.mdx
│   ├── resources.mdx
│   ├── usage.mdx
│   ├── verify.mdx
│   ├── cloud/
│   │   ├── analytics.mdx
│   │   ├── explorer-submission.mdx
│   │   ├── relay.mdx
│   │   └── verify.mdx
│   ├── early-access/
│   │   └── chain-abstraction.mdx
│   └── notifications/
│       ├── push.mdx
│       └── notify/
│           ├── installation.mdx
│           ├── overview.mdx
│           ├── spam-protection.mdx
│           └── usage.mdx
│
└── best-practices.mdx


================================================
File: walletkit/react-native/best-practices.mdx
================================================
---
title: Best Practices
---

The purpose of this guide is to show the best practices in regards of the WalletKit client usage. The goal is to provide the best user experience that just works in every circumstances.

<Info>
In order to ensure the best user experience and flawless connection flow, please make sure that WalletKit is initialized immediately after your app launch, especially if launched via a WalletConnect Deep Link. It guarantees that websocket connection is opened immediately and all requests are received by your wallet
</Info>

## Pairing

A pairing is a connection between a wallet and a dapp that has fixed permissions to only allow a dapp to propose a session through it. Dapp can propose infinite number of sessions on one pairing. Wallet must use a pair method from the WalletKit client to pair with dapp.

```typescript
const uri = 'xxx'; // pairing uri
try {
    await walletKit.pair({ uri });
} catch (error) {
    // some error happens while pairing - check Expected errors section
}
``` 

### Pairing Expiry

A pairing expiry event is triggered whenever a pairing is expired. The expiry for inactive pairing is 5 mins, whereas for active pairing is 30 days. A pairing becomes active when a session proposal is received and user successfully approves it. This event helps to know when given pairing expires and update UI accordingly.

```typescript
core.pairing.events.on("pairing_expire", (event) => {
    // pairing expired before user approved/rejected a session proposal
    const { topic } = topic;
});
```
### Expected User flow

### Pairing Flow

<Frame caption="">
    <img src="/images/assets/pairing.gif" />
</Frame>

### Pairing Error

<Frame caption="">
    <img src="/images/assets/pairing_error.gif" />
</Frame>

### Expected Errors

While pairing the following errors might occur:

- No Internet connection error or pairing timeout when scanning QR with no Internet connection
  - User should pair again with Internet connection
- Pairing expired error when scanning a QR code with expired pairing
  - User should refresh a QR code and scan again
- Pairing with existing pairing is not allowed
  - User should refresh a QR code and scan again. I usually happens when user scans an already paired QR code.

## Session Proposal

A session proposal is a handshake sent by a dapp and it's purpose is to define a session rules. Whenever a user wants to establish a connection between a wallet and a dapp, one should approve a session proposal.

### User Action Feedback

Whenever user approves or rejects a session proposal, wallet should show loading indicators in a moment of the button press until Relay acknowledgement is received for any of this actions.

Approving session
```typescript
    try {
        await walletKit.approveSession(params);
        // update UI -> remove the loader
    } catch (error) {
        // present error to the user
    }
```
Rejecting session
```typescript
    try {
        await walletKit.rejectSession(params);
        // update UI -> remove the loader
    } catch (error) {
        // present error to the user
    }
```

### Session Proposal Expiry

A session proposal expiry is 5 mins. It means a given proposal is stored for 5 mins in the SDK storage and user has 5 mins for approval or rejection decision. After that time the below event is emitted and proposal modal should be removed from the app's UI.

```typescript
walletKit.on("proposal_expire", (event) => {
    // proposal expired and any modal displaying it should be removed
    const { id } = event;
});
```

### Expected User flow

### Approve or Reject Session Proposal

<Frame caption="">
    <img src="/images/assets/pairing.gif" />
</Frame>

### Error Handling

<Frame caption="">
    <img src="/images/assets/proposal_error.gif" />
</Frame>

### Expected Errors

While approving or rejecting a session proposal the following errors might occurs:

- No Internet connection
  - It happens when a user tries to approve or reject session proposal with no Internet connection
- Session proposal expired
  - It happens when users tries to approve or reject expired session proposal
- Invalid namespaces
  - It happens when a validation of session namespaces fails
- Timeout
  - It happens when Relay doesn't acknowledge session settle publish within 10s

## Session Request

A session request represents the request sent by a dapp to a wallet.

### User Action Feedback

Whenever user approves or rejects a session request, wallet should show loading indicators in a moment of the button press until Relay acknowledgement is received for any of this actions.

```typescript
    try {
        await walletKit.respondSessionRequest(params);
        // update UI -> remove the loader
    } catch (error) {
        // present error to the user
    }
```

### Session Request Expiry

A session request expiry is defined by a dapp. It's value must be between now() + 5mins and now() + 7 days. After the session request expires the below event is emitted and session request modal should be removed from the app's UI.

```typescript
walletKit.on("session_request_expire", (event) => {
    // request expired and any modal displaying it should be removed
    const { id } = event;
});
```

### Expected User flow

### Approve or Reject Session Proposal

<Frame caption="">
    <img src="/images/assets/session_request.gif" />
</Frame>

### Error Handling

<Frame caption="">
    <img src="/images/assets/session_request_error.gif" />
</Frame>

### Expected Errors

While approving or rejecting a session request the following error might occur:

- Invalid session
  - This error might happen when user approves or rejects a session request on expired session
- Session request expired
  - This error might happen when user approves or rejects a session request that already expires
- Timeout
  - It happens when Relay doesn't acknowledge session settle publish within 10s

## Web Socket Connection State

The Web Socket connection state tracks the connection with the relay server, event is emitted whenever a connection state changes.

```typescript
core.relayer.on("relayer_connect", () => {
    // connection to the relay server is established
})

core.relayer.on("relayer_disconnect", () => {
// connection to the relay server is lost
})

```

### Expected User flow

### Connection State

<Frame caption="">
    <img src="/images/assets/connection_state.gif" />
</Frame>



================================================
File: walletkit/react-native/eip5792.mdx
================================================
---
title: Wallet Call API
---

WalletConnect supports @EIP-5792, which defines new JSON-RPC methods that enable apps to ask a wallet to process a batch of onchain write calls and to check on the status of those calls.
Applications can specify that these onchain calls be executed taking advantage of specific capabilities previously expressed by the wallet; an additional, a novel wallet RPC is defined to enable apps to query the wallet for those capabilities.

- `wallet_sendCalls`: Requests that a wallet submits a batch of calls.
- `wallet_getCallsStatus`: Returns the status of a call batch that was sent via wallet_sendCalls.
- `wallet_showCallsStatus`: Requests that a wallet shows information about a given call bundle that was sent with wallet_sendCalls.
- `wallet_getCapabilities`: This RPC allows an application to request capabilities from a wallet (e.g. batch transactions, paymaster communication).



================================================
File: walletkit/react-native/installation.mdx
================================================
---
title: Installation
---

Install WalletKit package.

```sh
yarn add @reown/walletkit @walletconnect/react-native-compat
```

Additionally add these extra packages to help with async storage, polyfills and the instance of ethers.

```sh
yarn add @react-native-async-storage/async-storage @react-native-community/netinfo react-native-get-random-values fast-text-encoding
```

<Accordion title="Additional setup for Expo">
```sh
npx expo install expo-application
```
</Accordion>

For those using Typescript, we recommend adding these dev dependencies:

```sh
yarn add @walletconnect/jsonrpc-types --dev
```

## Next Steps

Now that you've installed WalletKit, you're ready to start integrating it. The next section will walk you through the process of setting up your project to use the SDK.



================================================
File: walletkit/react-native/link-mode.mdx
================================================
---
title: Link Mode
---

WalletKit Link Mode is a low latency mechanism for transporting One-Click Auth requests and session requests over Universal Links, reducing the need for a WebSocket connection with the Relay. This significantly enhances the user experience when connecting native dApps to native wallets by reducing the latency associated with network connections, especially when the user has an unstable internet connection.

### How to enable it:

To support Link Mode add a universal link for your wallet in Cloud project configuration @dashboard, configure your Metadata with a valid universal link and set the `linkMode` property to `true`:

```ts {10-11}
const walletKit = await WalletKit.init({
  core,
  metadata: {
    name: "Demo React Native Wallet",
    description: "Demo RN Wallet to interface with Dapps",
    url: "www.reown.com/walletkit",
    icons: ["https://your_wallet_icon.png"],
    redirect: {
      native: "yourwalletscheme://",
      universal: "https://example.com/example_wallet",
      linkMode: true,
    },
  },
});
```

### Platform specifics:

<Tabs>
<Tab title="iOS">

To enable universal links for your app, refer to @React Native Documentation.<br />

After following the steps provided in the official guide:

1. Ensure that you handle incoming Universal Links in the your `AppDelegate.mm` file.

```swift
#import <React/RCTLinkingManager.h>

// Enable deeplinks
- (BOOL)application:(UIApplication *)application
   openURL:(NSURL *)url
   options:(NSDictionary<UIApplicationOpenURLOptionsKey,id> *)options
{
  return [RCTLinkingManager application:application openURL:url options:options];
}

// Enable Universal Links
- (BOOL)application:(UIApplication *)application continueUserActivity:(nonnull NSUserActivity *)userActivity
 restorationHandler:(nonnull void (^)(NSArray<id<UIUserActivityRestoring>> * _Nullable))restorationHandler
{
 return [RCTLinkingManager application:application
                  continueUserActivity:userActivity
                    restorationHandler:restorationHandler];
}
```

2. Open your project in XCode and go to `Settings/Signing & Capabilities/Associated Domains` to add the new domain. After this, `your_project.entitlement` should look like this:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>com.apple.developer.associated-domains</key>
  <array>
    <string>applinks:example.com</string>
  </array>
</dict>
</plist>
```

3. Update/Create your domain's `.well-known/apple-app-site-association` file accordingly.

For more information about supporting universal links, visit the @Supporting associated domains page

For a debugging guide, visit the @Debugging Universal Links page.<br />

</Tab>
<Tab title="Android">

Android Studio provides a tool to configure Universal Links easily, you can read the guide in @Android Documentation

After following the steps provided in the guide:

1. Ensure that your Universal Link is properly configured in your app's `AndroidManifest.xml` file with the `autoVerify` set to `true`. It should look similar to this:

```xml
<intent-filter android:autoVerify="true">
  <action android:name="android.intent.action.VIEW" />

  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />

  <data android:scheme="https" />
  <data android:host="example.com" />
  <data android:pathPattern="/example_wallet" />
</intent-filter>
```

2. Update/Create your domains's `.well-known/assetlinks.json` file accordingly

For more information on how to configure universal links for your app, refer to @Android Documentation.<br />
For testing the configured universal link to app content check @this documentation page.<br />

</Tab>
</Tabs>

Once everything is properly configured, and the user interacts with a Link Mode-supporting dApp, your wallet will receive requests through it.



================================================
File: walletkit/react-native/mobile-linking.mdx
================================================
---
title: Mobile Linking
---

import HowToTest from "/snippets/walletkit/shared/mobile-linking.mdx";

<Note>

This feature is only relevant to native platforms.

</Note>

## Usage

Mobile Linking allows your wallet to automatically redirect back to the Dapp allowing for less user interactions and hence a better UX for your users.

### Establishing Communication Between Mobile Wallets and Apps

When integrating a wallet with a mobile application, it's essential to understand how they communicate. The process involves two main steps:

1. **QR Code Handshake:** The mobile app (Dapp) generates a unique URI (Uniform Resource Identifier) and displays it as a QR code. This URI acts like a secret handshake. When the user scans the QR code using their wallet app, they establish a connection. It's like saying, "Hey, let's chat!"
2. **Deep Links and Universal Links:** The URI from the QR code allows the wallet app to create a @deep link or @universal link. These links work on both Android and iOS. They enable seamless communication between the wallet and the app.

<Tip>

**Developers should prefer Deep Linking over Universal Linking.**

Universal Linking may redirect the user to a browser, which might not provide the intended user experience. Deep Linking ensures the user is taken directly to the app.

</Tip>

### Key Behavior to Address

In some scenarios, wallets use redirect metadata provided in session proposals to open applications. This can cause unintended behavior, such as:

Redirecting to the wrong app when multiple apps share the same redirect metadata (e.g., a desktop and mobile version of the same Dapp).
Opening an unrelated application if a QR code is scanned on a different device than where the wallet is installed.

#### Recommended Approach

To avoid this behavior, wallets should:

- **Restrict Redirect Metadata to Deep Link Use Cases**: Redirect metadata should only be used when the session proposal is initiated through a deep link. QR code scans should not trigger app redirects using session proposal metadata.

The connection and sign request flows are similar across platforms.

### Connection Flow

- **Dapp Prompts User:** The Dapp asks the user to connect.
- **User Chooses Wallet:** The user selects a wallet from a list of compatible wallets.
- **Redirect to Wallet:** The user is redirected to their chosen wallet.
- **Wallet Approval:** The wallet prompts the user to approve or reject the session (similar to granting permission).
- **Return to Dapp:**
  - **Manual Return:** The wallet asks the user to manually return to the Dapp.
  - **Automatic Return:** Alternatively, the wallet automatically takes the user back to the Dapp.
- **User Reunites with Dapp:** After all the interactions, the user ends up back in the Dapp.


<Frame>
  <img alt="Mobile Linking Connect Flow" className="block dark:hidden" src="/images/w3w/mobileLinking-light.png" />
  <img alt="Mobile Linking Connect Flow" className="hidden dark:block" src="/images/w3w/mobileLinking-dark.png" />
</Frame>

### Sign Request Flow

When the Dapp needs the user to sign something (like a transaction), a similar pattern occurs:

- **Automatic Redirect:** The Dapp automatically sends the user to their previously chosen wallet.
- **Approval Prompt:** The wallet asks the user to approve or reject the request.
- **Return to Dapp:**
  - **Manual Return:** The wallet asks the user to manually return to the Dapp.
  - **Automatic Return:** Alternatively, the wallet automatically takes the user back to the Dapp.
- **User Reconnects:** Eventually, the user returns to the Dapp.

<Frame>
  <img alt="Mobile Linking Connect Flow" className="block dark:hidden" src="/images/w3w/mobileLinking_sign-light.png" />
  <img alt="Mobile Linking Connect Flow" className="hidden dark:block" src="/images/w3w/mobileLinking_sign-dark.png" />
</Frame>

## Platform preparations

Since React Native leverages on native APIs, you must follow iOS and Android steps for each native platform

More information in official documentation: https://reactnative.dev/docs/linking?syntax=android#enabling-deep-links

<Tip>

Dapps developers must do the same for their own custom schemes if they want the wallet to be able to navigate back after a session approval or a sign request response

</Tip>

<HowToTest />

## Integration

In order to redirect to the Dapp, you'll need to use `Linking` from `react-native` and call `openURL()` method with the Dapp scheme that comes in the proposal metadata.

```js
import { Linking } from "react-native";

async function onApprove(proposal, namespaces) {
  const session = await walletKit.approveSession({
    id: proposal.id,
    namespaces,
  });

  const dappScheme = session.peer.metadata.redirect?.native;

  if (dappScheme) {
    Linking.openURL(dappScheme);
  } else {
    // Inform the user to manually return to the DApp
  }
}
```



================================================
File: walletkit/react-native/one-click-auth.mdx
================================================
---
title: One-click Auth
---

## Introduction

This section outlines an innovative protocol method that facilitates the initiation of a Sign session and the authentication of a wallet through a @Sign-In with Ethereum (SIWE) message, enhanced by @ReCaps (ReCap Capabilities).

This enhancement not only offers immediate authentication for dApps, paving the way for prompt user logins, but also integrates informed consent for authorization. Through this mechanism, dApps can request the delegation of specific capabilities to perform actions on behalf of the wallet user. These capabilities, encapsulated within SIWE messages as ReCap URIs, detail the scope of actions authorized by the user in an explicit and human-readable form.

By incorporating ReCaps, this method extends the utility of SIWE messages, allowing dApps to combine authentication with a nuanced authorization model. This model specifies the actions a dApp is authorized to execute on the user's behalf, enhancing security and user autonomy by providing clear consent for each delegated capability. As a result, dApps can utilize these consent-backed messages to perform predetermined actions, significantly enriching the interaction between dApps, wallets, and users within the Ethereum ecosystem.

<Frame>
  <img alt="Mobile Linking Connect Flow" className="block dark:hidden" src="/images/w3w/authenticatedSessions-light.png" />
  <img alt="Mobile Linking Connect Flow" className="hidden dark:block" src="/images/w3w/authenticatedSessions-dark.png" />
</Frame>

## Handling Authentication Requests

To handle incoming authentication requests, subscribe to the `session_authenticate` event. This will notify you of any authentication requests that need to be processed, allowing you to either approve or reject them based on your application logic.

```typescript
walletKit.on("session_authenticate", async (payload) => {
  // Process the authentication request here.
  // Steps include:
  // 1. Populate the authentication payload with the supported chains and methods
  // 2. Format the authentication message using the payload and the user's account
  // 3. Present the authentication message to the user
  // 4. Sign the authentication message(s) to create a verifiable authentication object(s)
  // 5. Approve the authentication request with the authentication object(s)
});
```

## Authentication Payload

```typescript
import { populateAuthPayload } from "@walletconnect/utils";

// EVM chains that your wallet supports
const supportedChains = ["eip155:1", "eip155:2", 'eip155:137'];
// EVM methods that your wallet supports
const supportedMethods = ["personal_sign", "eth_sendTransaction", "eth_signTypedData"];
// Populate the authentication payload with the supported chains and methods
const authPayload = populateAuthPayload({
  authPayload: payload.params.authPayload,
  chains: supportedChains,
  methods: supportedMethods,
});
// Prepare the user's address in CAIP10(https://github.com/ChainAgnostic/CAIPs/blob/main/CAIPs/caip-10.md) format
const iss = `eip155:1:0x0Df6d2a56F90e8592B4FfEd587dB3D5F5ED9d6ef`;
// Now you can use the authPayload to format the authentication message
const message = walletKit.formatAuthMessage({
  request: authPayload,
  iss
});

// Present the authentication message to the user
...
```

## Approving Authentication Requests

<Note>

1. The recommended approach for secure authentication across multiple chains involves signing a SIWE (Sign-In with Ethereum) message for each chain and account. However, at a minimum, one SIWE message must be signed to establish a session. It is possible to create a session for multiple chains with just one issued authentication object.
2. Sometimes a dapp may want to only authenticate the user without creating a session, not every approval will result with a new session.

</Note>

```typescript
// Approach 1
// Sign the authentication message(s) to create a verifiable authentication object(s)
const signature = await cryptoWallet.signMessage(message, privateKey);
// Build the authentication object(s)
const auth = buildAuthObject(
  authPayload,
  {
    t: "eip191",
    s: signature,
  },
  iss
);

// Approve
await walletKit.approveSessionAuthenticate({
  id: payload.id,
  auths: [auth],
});

// Approach 2
// Note that you can also sign multiple messages for every requested chain/address pair
const auths = [];
authPayload.chains.forEach(async (chain) => {
  const message = walletKit.formatAuthMessage({
    request: authPayload,
    iss: `${chain}:${cryptoWallet.address}`,
  });
  const signature = await cryptoWallet.signMessage(message);
  const auth = buildAuthObject(
    authPayload,
    {
      t: "eip191", // signature type
      s: signature,
    },
    `${chain}:${cryptoWallet.address}`
  );
  auths.push(auth);
});

// Approve
await walletKit.approveSessionAuthenticate({
  id: payload.id,
  auths,
});
```

## Rejecting Authentication Requests

If the authentication request cannot be approved or if the user chooses to reject it, use the rejectSession method.

```typescript
import { getSdkError } from "@walletconnect/utils";

await walletKit.rejectSessionAuthenticate({
  id: payload.id,
  reason: getSdkError("USER_REJECTED"), // or choose a different reason if applicable
});
```

## Testing One-click Auth

You can use @AppKit Labs to test and verify that your wallet supports One-click Auth properly.

<Card
  title="Test One-click Auth"
  href="https://appkit-lab.reown.com/library/ethers-siwe/"
/>



================================================
File: walletkit/react-native/resources.mdx
================================================
---
title: Resources
---

Valuable assets for developers and users interested in integrating WalletKit into their applications.

- @Awesome WalletConnect - Community-curated collection of WalletConnect-enabled wallets, libraries, and tools.
- @AppKit Laboratory - A place to test your wallet integrations against various setups of AppKit.
- @WalletKit GitHub - WalletKit GitHub repository.

### Wallet Resources

@WalletKit simplifies the integration process for wallet developers by combining our Sign and Auth APIs. Please note that only V2 @WCURIs will work with this SDK, as V1 is being deprecated by June 28th, 2023.

#### Expo

Experimental: For Expo, we have an unofficial npx starter command. `newWallet` represents the name of your project.

```bash
npx create-wc-wallet-expo@latest newWallet
```

This downloads an Expo template with WalletKit installed. More information available in this @tutorial

### Dapp Resources

If you need to test your app's integration, you can use one of our following demo wallets and/or dapps.

**Sign**

- @React dApp (with standalone client) - v2 (@Demo)
- @React dApp (with EthereumProvider + Ethers.js) - v2 (@Demo)
- @React dApp (with EthereumProvider + web3.js) - v2 (@Demo)
- @React dApp (with CosmosProvider) - v2 (@Demo)


================================================
File: walletkit/react-native/usage.mdx
================================================
---
title: Usage
---

import CloudBanner from "/snippets/cloud-banner.mdx";

This section provides instructions on how to initialize the WalletKit client, approve sessions with supported namespaces, and respond to session requests, enabling easy integration of Web3 wallets with dapps through a simple and intuitive interface.

## Cloud Configuration

Create a new project on Reown Cloud at https://cloud.reown.com and obtain a new project ID.

<CloudBanner />

## Initialization

<Note>

`@walletconnect/react-native-compat` must be installed and imported before any `@reown/*` dependencies for proper React Native polyfills.

```ts
import "@walletconnect/react-native-compat";
// Other imports
```

</Note>

Create a new instance from `Core` and initialize it with your `projectId`. Next, create a WalletKit instance by calling `init` on `WalletKit`. Passing in the options object containing metadata about the app.

The `pair` function will help us pair between the dapp and wallet and will be used shortly.

```javascript
import { Core } from "@walletconnect/core";
import { WalletKit } from "@reown/walletkit";

const core = new Core({
  projectId: process.env.PROJECT_ID,
});

const walletKit = await WalletKit.init({
  core, // <- pass the shared `core` instance
  metadata: {
    name: "Demo React Native Wallet",
    description: "Demo RN Wallet to interface with Dapps",
    url: "www.walletconnect.com",
    icons: ["https://your_wallet_icon.png"],
    redirect: {
      native: "yourwalletscheme://",
    },
  },
});
```

## Session

A session is a connection between a dapp and a wallet. It is established when a user approves a session proposal from a dapp. A session is active until the user disconnects from the dapp or the session expires.

### Namespace Builder

With WalletKit (and @walletconnect/utils) we've published a helper utility that greatly reduces the complexity of parsing the `required` and `optional` namespaces. It accepts as parameters a `session proposal` along with your user's `chains/methods/events/accounts` and returns ready-to-use `namespaces` object.

```javascript
// util params
{
  proposal: ProposalTypes.Struct; // the proposal received by `.on("session_proposal")`
  supportedNamespaces: Record< // your Wallet's supported namespaces
    string, // the supported namespace key e.g. eip155
    {
      chains: string[]; // your supported chains in CAIP-2 format e.g. ["eip155:1", "eip155:2", ...]
      methods: string[]; // your supported methods e.g. ["personal_sign", "eth_sendTransaction"]
      events: string[]; // your supported events e.g. ["chainChanged", "accountsChanged"]
      accounts: string[] // your user's accounts in CAIP-10 format e.g. ["eip155:1:0x453d506b1543dcA64f57Ce6e7Bb048466e85e228"]
      }
  >;
};
```

Example usage

```javascript
// import the builder util
import { WalletKit, WalletKitTypes } from '@reown/walletkit'
import { buildApprovedNamespaces, getSdkError } from '@walletconnect/utils'

async function onSessionProposal({ id, params }: WalletKitTypes.SessionProposal){
  try{
    // ------- namespaces builder util ------------ //
    const approvedNamespaces = buildApprovedNamespaces({
      proposal: params,
      supportedNamespaces: {
        eip155: {
          chains: ['eip155:1', 'eip155:137'],
          methods: ['eth_sendTransaction', 'personal_sign'],
          events: ['accountsChanged', 'chainChanged'],
          accounts: [
            'eip155:1:0xab16a96d359ec26a11e2c2b3d8f8b8942d5bfcdb',
            'eip155:137:0xab16a96d359ec26a11e2c2b3d8f8b8942d5bfcdb'
          ]
        }
      }
    })
    // ------- end namespaces builder util ------------ //

    const session = await walletKit.approveSession({
      id,
      namespaces: approvedNamespaces
    })
  }catch(error){
    // use the error.message to show toast/info-box letting the user know that the connection attempt was unsuccessful
   ....

    await walletKit.rejectSession({
      id: proposal.id,
      reason: getSdkError("USER_REJECTED")
    })
  }
}


walletKit.on('session_proposal', onSessionProposal)
```

If your wallet supports multiple namespaces e.g. `eip155`,`cosmos` & `near`
Your `supportedNamespaces` should look like the following example.

```javascript
// ------- namespaces builder util ------------ //
const approvedNamespaces = buildApprovedNamespaces({
    proposal: params,
    supportedNamespaces: {
        eip155: {...},
        cosmos: {...},
        near: {...}
    },
});
// ------- end namespaces builder util ------------ //
```

### Get Active Sessions

You can get the wallet active sessions using the `getActiveSessions` function.

```js
const activeSessions = walletKit.getActiveSessions();
```

### EVM methods & events

In @walletconnect/ethereum-provider, (our abstracted EVM SDK for apps) we support by default the following Ethereum methods and events:

```ts
{
  //...
  methods: [
    "eth_accounts",
    "eth_requestAccounts",
    "eth_sendRawTransaction",
    "eth_sign",
    "eth_signTransaction",
    "eth_signTypedData",
    "eth_signTypedData_v3",
    "eth_signTypedData_v4",
    "eth_sendTransaction",
    "personal_sign",
    "wallet_switchEthereumChain",
    "wallet_addEthereumChain",
    "wallet_getPermissions",
    "wallet_requestPermissions",
    "wallet_registerOnboarding",
    "wallet_watchAsset",
    "wallet_scanQRCode",
    "wallet_sendCalls",
    "wallet_getCallsStatus",
    "wallet_showCallsStatus",
    "wallet_getCapabilities",
  ],
  events: [
    "chainChanged",
    "accountsChanged",
    "message",
    "disconnect",
    "connect",
  ]
}
```

### Session Approval

In order to connect with a dapp, you will need to receive a WalletConnect URI (WCURI) and this will talk to our protocol to facilitate a pairing session. Therefore, you will need a test dapp in order to communicate with the wallet. We recommend testing with our @React V2 Dapp as this is the most up-to-date development site.

In order to capture the WCURI, recommend having some sort of state management you will pass through a `TextInput` or QRcode instance.

The `session_proposal` event is emitted when a dapp initiates a new session with a user's wallet. The event will include a `proposal` object with information about the dapp and requested permissions. The wallet should display a prompt for the user to approve or reject the session. If approved, call `approveSession` and pass in the `proposal.id` and requested `namespaces`.

The `pair` method initiates a WalletConnect pairing process with a dapp using the given `uri` (QR code from the dapps). To learn more about pairing, checkout out the @docs.

```javascript
import { getSdkError } from "@walletconnect/utils";

// Approval: Using this listener for sessionProposal, you can accept the session
walletKit.on("session_proposal", async (proposal) => {
  const session = await walletKit.approveSession({
    id: proposal.id,
    namespaces,
  });
});

// Call this after WCURI is received
await walletKit.pair({ uri: wcuri });
```

### Session Rejection

You can use the `getSDKError` function, which is available in the `@walletconnect/utils` for the rejection function @library.

```javascript
import { getSdkError } from "@walletconnect/utils";

// Reject: Using this listener for sessionProposal, you can reject the session
walletKit.on("session_proposal", async (proposal) => {
  await walletKit.rejectSession({
    id: proposal.id,
    reason: getSdkError("USER_REJECTED_METHODS"),
  });
});
```

### Responding to Session requests

<Frame caption="session-request-example">
    <img src="/images/assets/SessionRequestExample.png" />
</Frame>

The `session_request` event is triggered by a dapp when it needs the wallet to perform a specific action, such as signing a transaction. The event contains a `topic` and a `request` object, which will vary depending on the action requested.

To respond to the request, the wallet can access the `topic` and `request` object by destructuring them from the event payload. To see a list of possible `request` and `response` objects, refer to the relevant JSON-RPC Methods for @Ethereum, @Solana, @Cosmos, or @Stellar.

As an example, if the dapp requests a `personal_sign` method, the wallet can extract the `params` array from the `request` object. The first item in the array is the hex version of the message to be signed, which can be converted to UTF-8 and assigned to a `message` variable. The second item in `params` is the user's wallet address.

To sign the message, the wallet can use the `wallet.signMessage` method and pass in the message. The signed message, along with the `id` from the event payload, can then be used to create a `response` object, which can be passed into `respondSessionRequest`.

The wallet then signs the message. `signedMessage`, along with the `id` from the event payload, can then be used to create a `response` object, which can be passed into `respondSessionRequest`.

```javascript
walletKit.on("session_request", async (event) => {
  const { topic, params, id } = event;
  const { request } = params;
  const requestParamsMessage = request.params[0];

  // convert `requestParamsMessage` by using a method like hexToUtf8
  const message = hexToUtf8(requestParamsMessage);

  // sign the message
  const signedMessage = await wallet.signMessage(message);

  const response = { id, result: signedMessage, jsonrpc: "2.0" };

  await walletKit.respondSessionRequest({ topic, response });
});
```

To reject a session request, the response should be similar to this.

```javascript
const response = {
  id,
  jsonrpc: "2.0",
  error: {
    code: 5000,
    message: "User rejected.",
  },
};
```

### Updating a Session

The `session_update` event is emitted from the wallet when the session is updated by calling `updateSession`. To update a session, pass in the @topic and the new namespace.

```javascript
await walletKit.updateSession({ topic, namespaces: newNs });
```

### Extending a Session

To extend the session, call the `extendSession` method and pass in the new `topic`. The `session_update` event will be emitted from the wallet.

```javascript
await walletKit.extendSession({ topic });
```

### Session Disconnect

When either the dapp or the wallet disconnects from a session, a `session_delete` event will be emitted. It's important to subscribe to this event so you could keep your state up-to-date.

To initiate a session disconnect, call the `disconnectSession` method and pass in the `topic` and `reason`. You can use the `getSDKError` utility function, which is available in the `@walletconnect/utils` @library.

```javascript
await walletKit.disconnectSession({
  topic,
  reason: getSdkError("USER_DISCONNECTED"),
});
```

### Emitting Session Events

To emit session events, call the `emitSessionEvent` and pass in the params. If you wish to switch to chain/account that is not approved (missing from `session.namespaces`) you will have to update the session first. In the following example, the wallet will emit `session_event` that will instruct the dapp to switch the active accounts.

```javascript
await walletKit.emitSessionEvent({
  topic,
  event: {
    name: "accountsChanged",
    data: ["0xab16a96D359eC26a11e2C2b3d8f8B8942d5Bfcdb"],
  },
  chainId: "eip155:1",
});
```

In the following example, the wallet will emit `session_event` when the wallet switches chains.

```javascript
await walletKit.emitSessionEvent({
  topic,
  event: {
    name: "chainChanged",
    data: 1,
  },
  chainId: "eip155:1",
});
```



================================================
File: walletkit/react-native/verify.mdx
================================================
---
title: Verify API
---

Verify API is a security-focused feature that allows wallets to notify end-users when they may be connecting to a suspicious or malicious domain, helping to prevent phishing attacks across the industry.
Once a wallet knows whether an end-user is on uniswap.com or eviluniswap.com, it can help them to detect potentially harmful connections through Verify's combined offering of Reown's domain registry.

When a user initiates a connection with an application, Verify API enables wallets to present their users with four key states that can help them determine whether the domain they’re about to connect to might be malicious.

These are:

<Frame caption="Verify Banner">
    <img src="/images/verify-banner.png" />
</Frame>

## Disclaimer

Verify API is not designed to be bulletproof but to make the impersonation attack harder and require a somewhat sophisticated attacker. We are working on a new standard with various partners to close those gaps and make it bulletproof.

## Domain risk detection

The Verify security system will discriminate session proposals & session requests with distinct validations that can be either `VALID`, `INVALID` or `UNKNOWN`.

- Domain match: The domain linked to this request has been verified as this application's domain.
  - This interface appears when the domain a user is attempting to connect to has been ‘verified’ in our domain registry as the registered domain of the application the user is trying to connect to, and the domain has not returned as suspicious from either of the security tools we work with. The `verifyContext` included in the request will have a validation of `VALID`.
- Unverified: The domain sending the request cannot be verified.
  - This interface appears when the domain a user is attempting to connect to has not been verified in our domain registry, but the domain has not returned as suspicious from either of the security tools we work with. The `verifyContext` included in the request will have a validation of `UNKNOWN`.
- Mismatch: The application's domain doesn't match the sender of this request.
  - This interface appears when the domain a user is attempting to connect to has been flagged as a different domain to the one this application has verified in our domain registry, but the domain has not returned as suspicious from either of the security tools we work with. The `verifyContext` included in the request will have a validation of `INVALID`
- Threat: This domain is flagged as malicious and potentially harmful.
  - This interface appears when the domain a user is attempting to connect to has been flagged as malicious on one or more of the security tools we work with. The `verifyContext` included in the request will contain parameter `isScam` with value `true`.

### Implementation

To check the Verify API validations and whether or not your user is interacting with potentially malicious app, you can do so by accessing the `verifyContext` included in the request payload.

```javascript
...
walletKit.on("auth_request", async (authRequest) => {
  const { verifyContext } = authRequest
  const validation = verifyContext.verified.validation // can be VALID, INVALID or UNKNOWN
  const origin = verifyContext.verified.origin // the actual verified origin of the request
  const isScam = verifyContext.verified.isScam // true if the domain is flagged as malicious

  // if the domain is flagged as malicious, you should warn the user as they may lose their funds - check the `Threat` case for more info
  if(isScam) {
    // show a warning screen to the user
    // and proceed only if the user accepts the risk
  }

  switch(validation) {
    case "VALID":
      // proceed with the request - check the `Domain match` case for more info
      break
    case "INVALID":
      // show a warning dialog to the user - check the `Mismatch` case for more info
      // and proceed only if the user accepts the risk
      break
    case "UNKNOWN":
      // show a warning dialog to the user - check the `Unverified` case for more info
      // and proceed only if the user accepts the risk
      break
  }
})
```

For live demo examples of the intended Verify API flows, check out our demo apps:

- @Demo Web Wallet
- @Demo React Native Wallet
- @Demo App - you can toggle between the verify states by clicking on the `gear` & selecting the decided Validation before connecting to the wallet
- @Demo Malicious App - this app is flagged as malicious and will have the `isScam` parameter set to `true` in the `verifyContext` of the request



================================================
File: walletkit/react-native/cloud/analytics.mdx
================================================
---
title: Analytics
---

import Analytics from "/snippets/cloud/analytics.mdx";

<Analytics />



================================================
File: walletkit/react-native/cloud/explorer-submission.mdx
================================================
---
title: Explorer Submission
---

import ExplorerSubmission from "/snippets/cloud/explorer-submission.mdx";

<ExplorerSubmission />



================================================
File: walletkit/react-native/cloud/relay.mdx
================================================
---
title: Relay
---

import Relay from "/snippets/cloud/relay.mdx";

<Relay />



================================================
File: walletkit/react-native/cloud/verify.mdx
================================================
---
title: Verify
---

import Verify from "/snippets/cloud/verify.mdx";

<Verify />



================================================
File: walletkit/react-native/early-access/chain-abstraction.mdx
================================================
---
title: Chain Abstraction
---

<Info>
💡 Chain Abstraction is currently in early access phase. 
</Info>

Chain Abstraction in WalletKit enables users with stablecoins on any network to spend them on-the-fly on a different network. Our Chain Abstraction solution provides a toolkit for wallet developers to integrate this complex functionality using WalletKit.

For example, when an app requests a 100 USDC payment on Base network but the user only has USDC on Arbitrum, WalletKit offers methods to detect this mismatch, generate necessary transactions, track the cross-chain transfer, and complete the original transaction after bridging finishes.


## How It Works

<Info>
Apps need to pass `gas` as null when sending a transaction to allow proper gas estimation by the wallet. Refer to this @guide for more details.
</Info>

When sending a transaction, you need to:
1. Check if the required chain has enough funds to complete the transaction
2. If not, use the `prepare` method to generate necessary bridging transactions
3. Check the status of the bridging transactions and wait for them to complete
4. Once the bridging transactions are completed, execute the initial transaction

The following sequence diagram illustrates the complete flow of a chain abstraction operation, from the initial dapp request to the final transaction confirmation

<Frame caption="Chain Abstraction Flow">
    <img src="/images/assets/chain-abstraction-flow.png" />
</Frame>

## Methods

<Info>
Make sure you are using canary version of `@reown/walletkit` and `@walletconnect/react-native-compat`
</Info>

Following are the methods from WalletKit that you will use in implementing chain abstraction.

### Prepare 

This method checks if a transaction requires additional bridging transactions beforehand.

```typescript
public prepare(params: {
  transaction: ChainAbstractionTypes.Transaction;
}): Promise<ChainAbstractionTypes.CanFulfilResponse>;
```

### Status 

Helper method used to check the fulfilment status of a bridging operation.

```typescript
public abstract status(params: {
  fulfilmentId: string;
}): Promise<ChainAbstractionTypes.FulfilmentStatusResponse>;
```

### Usage

When sending a transaction, first check if chain abstraction is needed using the `prepare` method. If it is needed, you need to sign all the fulfillment transactions and broadcast them in parallel. 
After that, you need to call the `status` method to check the status of the fulfillment operation. 

If the operation is successful, you need to broadcast the initial transaction and await the transaction hash and receipt. 
If the operation is not successful, you need to send the JsonRpcError to the dapp and display the error to the user. 

```typescript
walletkit.on("session_request", async (event) => {
  const {
    id,
    topic,
    params: { request, chainId },
  } = event;

  if (request.method === "eth_sendTransaction") {
    const originalTransaction = request.params[0];
    
    // Check if bridging transactions are required
    const prepareResponse = await wallet.prepare({
      transaction: {
        ...originalTransaction,
        chainId,
      },
    });

    if (prepareResponse.status === "error") {
      // Display error to user and respond to dapp
      await wallet.respondSessionRequest({
        topic,
        response: formatJsonRpcResult(id, prepareResponse.reason),
      });
      return;
    }

    if (prepareResponse.status === "available") {
      const { transactions, funding, fulfilmentId, checkIn } = prepareResponse.data;

      // Display bridging information to user
      // Once approved, execute bridging transactions
      for (const transaction of transactions) {
        const signedTransaction = await wallet.signTransaction(transaction);
        await wallet.sendTransaction(signedTransaction);
      }
      // await for the completed fulfilment status 
      await statusResult = await wallet.status({
        fulfilmentId
      });

      // Proceed with original transaction
      const signedTransaction = await wallet.signTransaction(originalTransaction);
      const result = await wallet.sendTransaction(signedTransaction);
      
      await wallet.respondSessionRequest({
        topic,
        response: formatJsonRpcResult(id, result)
      });
    } else {
      // No bridging required, process transaction normally
      const signedTransaction = await wallet.signTransaction(originalTransaction);
      const result = await wallet.sendTransaction(signedTransaction);
      
      await wallet.respondSessionRequest({
        topic,
        response: formatJsonRpcResult(id, result)
      });
    }
  }
});
```

## Types

Following are the types that are used in the chain abstraction methods.

```typescript
namespace ChainAbstractionTypes {
  type FundingFrom = {
    tokenContract: string;
    amount: string;
    chainId: string;
    symbol: string;
  };

  type Transaction = {
    from: string;
    to: string;
    value: string;
    chainId: string;
    gas?: string;
    gasPrice?: string;
    data?: string;
    nonce?: string;
    maxFeePerGas?: string;
    maxPriorityFeePerGas?: string;
  };

  type CanFulfilResponse =
    | {
        status: "not_required";
      }
    | {
        status: "available";
        data: {
          fulfilmentId: string;
          checkIn: number;
          transactions: Transaction[];
          funding: FundingFrom[];
        };
      }
    | {
        status: "error";
        reason: string; // reason can be insufficientFunds | insufficientGasFunds | noRoutesAvailable
      };

  type FulfilmentStatusResponse = {
    createdAt: number;
  } & (
    | {
        status: "completed";
      }
    | { status: "pending"; checkIn: number }
  );
}
```

For example, check out implementation of chain abstraction in @sample wallet with React Native CLI. 

## Testing 

To test Chain Abstraction, you can use the @AppKit laboratory and try sending @USDC/USDT with any chain abstraction supported wallet. 
You can also use this @sample wallet for testing. 

<video controls width="100%" height="100%" style={{ borderRadius: '10px' }}>
  <source src="/assets/chain-abstraction-demo.mp4" type="video/mp4" />
</video>



================================================
File: walletkit/react-native/notifications/push.mdx
================================================
---
title: Push Notifications
---

WalletKit provides the functionality for wallets to receive push notifications through Firebase Cloud Messaging (FCM) and Apple Push Notification Service (APNs) via the Push Server. This feature ensures that wallets are promptly notified of incoming signature requests. Each push notification contains the encrypted details of the signature request. Upon receiving the notification, it can be decrypted and presented to the developer, allowing for customization of the message according to their requirements.

## Server setup

For the push notifications to be forwarded to FCM or APNs, the @Push Server will need to be configured with your FCM or APNs server API credentials.

## App setup

### Register the device token

To enable a device for push notifications, it's essential to register the device token using `walletKit.registerDeviceToken`. This token can be obtained from either FCM or APNS, depending on the platform used.

To receive push notifications from WalletConnect's Push Server via Firebase Cloud Messaging, you will need to setup Firebase in your project. 
You can follow their documentation - @Firebase documentation. 
Once you have Firebase configured, you can obtain the device token by calling `messaging().getToken()`. This unique token is used to identify each device.

```ts
import messaging from '@react-native-firebase/messaging'

const token = await messaging().getToken()
```

The device token will be used to register for WalletConnect push notifications by calling `walletKit.registerDeviceToken` and passing the token as an argument.
The `registerDeviceToken` should be called every time the client is initialized.

```ts
walletKit.registerDeviceToken({
  token: await messaging().getToken(), // device token
  clientId: await walletKit.core.crypto.getClientId(), //your instance clientId
  notificationType: 'fcm', // notification type
  enableEncrypted: true // flag that enabled detailed notifications
})
```

With that the base setup is complete and you can start receiving push notifications from WalletConnect's Push Server for your sessions.

Note, that from time to time, the device token is refreshed, so you must make sure to register it again.

```ts
import messaging from '@react-native-firebase/messaging';

messaging().onTokenRefresh(async token => {
    await walletKit.registerDeviceToken({
        token: await messaging().getToken(), // device token
        clientId: await walletKit.core.crypto.getClientId(), //your instance clientId
        notificationType: 'fcm', // notification type
        enableEncrypted: true // flag that enabled detailed notifications
    });
});
```

### Receiving push notifications

After the device token is registered, the next step involves setting up the notification service specific to the platform being used. This service will decrypt the incoming requests and forward them to the developer for further processing and integration.

To receive the actual push notifications, you will need to subscribe to firebase messaging events.

```ts
import messaging from '@react-native-firebase/messaging';

// emitted when the app is open and a notification is received
messaging().onMessage(async notification => {
    ...
});

// emitted when the app is in the background or closed and a notification is received
messaging().setBackgroundMessageHandler(async notification => {
    ...
});

```

Now that we have the notifications listeners setup, we can start processing the incoming notifications.

```ts
import { WalletKit } from '@reown/walletkit';
import messaging from '@react-native-firebase/messaging';

messaging().onMessage(async notification => {
    // get the topic, encrypted message & tag from the notification payload
    const { topic, message, tag } = notification.data;

    // decrypt the message
    // note this is static method and can be called without initializing the walletKit
    const decryptedMessage = await WalletKit.notifications.decryptMessage({
    topic,
    encryptedMessage: message,
  });

    /*
    * `decryptedMessage` is JsonRpcRequest object, with the full payload of the incoming request such as method, params, id, etc.
    * You can use it to emit local push notification with the request to the user and ask for their approval.
    **/

   /*
   * the metadata contains name, description, icon and url of the dapp that initiated the request
   * note that only notifications with tag `1108`(session requests) will have metadata,
   **/
   let metadata

   if(tag == 1108) {
        metadata = await WalletKit.notifications.getMetadata({ topic });
   } else {
        // session proposals contain metadata in the request itself
        metadata = decryptedMessage.params.proposer.metadata
   }

    // with this information you can show a local push notification to the user
   ...
});
```



================================================
File: walletkit/react-native/notifications/notify/installation.mdx
================================================
---
title: Installation
---

Install the WalletConnect NotifyClient package.

```sh
yarn add @walletconnect/notify-client @walletconnect/react-native-compat
```

You will need to polyfill crypto depending on your environment. See instructions below.

<Tabs>
<Tab title="Expo">

```sh
yarn add expo-crypto
```

1. Create a file called `expo-crypto-shim.js` at the root of your project
2. Go to `expo-crypto-shim.js`and paste the following snippet into it.

```js
import { digest } from "expo-crypto";

// eslint-disable-next-line no-undef
const webCrypto = typeof crypto !== "undefined" ? crypto : new Crypto();
webCrypto.subtle = {
  digest: (algo, data) => {
    const buf = Buffer.from(data);
    return digest(algo, buf);
  },
};
(() => {
  if (typeof crypto === "undefined") {
    Object.defineProperty(window, "crypto", {
      configurable: true,
      enumerable: true,
      get: () => webCrypto,
    });
  }
})();
```

3. Then head over your `index.js` file at the root of your project and add the following imports.

```js
import "@walletconnect/react-native-compat";
import "./expo-crypto-shim.js";
```

</Tab>
<Tab title="React Native CLI">

```sh
yarn add react-native-quick-crypto react-native-quick-base64 stream-browserify @craftzdog/react-native-buffer babel-plugin-module-resolver
```

For iOS only

```bash
cd ios && pod install
```

1. Go to your `index.js` file at the root of your project and add the following polyfill

```js
import { AppRegistry } from "react-native";
import App from "./App";
import { name as appName } from "./app.json";
import crypto from "react-native-quick-crypto";

const polyfillDigest = async (algorithm, data) => {
  const algo = algorithm.replace("-", "").toLowerCase();
  const hash = crypto.createHash(algo);
  hash.update(data);
  return hash.digest();
};

globalThis.crypto = crypto;
globalThis.crypto.subtle = {
  digest: polyfillDigest,
};

AppRegistry.registerComponent(appName, () => App);
```

2. Update your `babel.config.js` with the following configuration

```js
module.exports = {
  presets: ['module:metro-react-native-babel-preset'],
  plugins: [
   [
     'module-resolver',
     {
       alias: {
         'crypto': 'react-native-quick-crypto',
         'stream': 'stream-browserify',
         'buffer': '@craftzdog/react-native-buffer',
       },
     },
   ],
    ...
  ],
};
```

</Tab>
</Tabs>

## Next Steps

Now that you've installed WalletConnect Notify, you're ready to start integrating it. The next section will walk you through the process of setting up your project to use the Notify API.



================================================
File: walletkit/react-native/notifications/notify/overview.mdx
================================================
---
title: Overview
---

<Note>
For those integrating notifications related to wallet pairing and sign requests, please check @here.
</Note>

The WalletKit Notify API is designed to enhance the interaction between wallet users and dapps by offering a robust notification system. This API empowers wallet developers to implement a dynamic notification experience directly within their wallets. It provides the functionality for users to opt-in to notifications, ensuring they stay informed about critical events and interactions.

The Notify API is versatile, with support for both iOS and Android platforms, making it an ideal choice for cross-platform wallet applications.

Coupled with the @AppKit Notifications, the Notify API forms part of a comprehensive toolkit that enables seamless integration of web3 communication and messaging features into dapps. This ensures a more connected and interactive experience for users in the decentralized ecosystem.

## Features

Some of the key features of the Notify API include:

- **Push Notifications for Desktop and Native Platforms**: This feature enables dapps to directly send vital notifications to user wallets, ensuring timely and relevant communication.
- **Robust Spam Protection**: Users have complete authority over which dapps can send them notifications, effectively eliminating any unsolicited messages from unknown sources. Furthermore, users can fine-tune their preferences to only receive notifications types they are interested in, like new features or some important events occurrence.
- **Chain Agnostic Architecture**: The Notify API is built to be compatible with any blockchain, allowing seamless multi-chain support without the need for writing additional integration code. **As of November 2023, the Notify Server and Clients are equipped to support EVM chains. Plans to extend support to non-EVM chains are in progress and are a significant part of our upcoming development roadmap.**

_Example integration_
<Frame caption="Web3Inbox">
    <img src="/images/assets/web3inbox/w3i-hero.png" />
</Frame>



================================================
File: walletkit/react-native/notifications/notify/spam-protection.mdx
================================================
---
title: Spam Protection
---

Users play a critical role in web3. That's why, with Web3Inbox, we’re committed to ensuring users can enjoy a safe, seamless, and reliable experience that puts them in the driver’s seat. As part of that pledge, Web3Inbox provides a number of user-first, anti-spam features and elements that ensure users are always in control of their web3 communications.

## How are users protected from spam with Web3Inbox?

### Becoming a WalletKit Notification customer

When a wallet offers app notifications to their users via Web3Inbox, the feature will always be optional. If users decide they want to receive notifications from selected apps via their wallet, they’ll be able to ‘opt-in’ and subscribe to an app’s notifications by signing a message request. Similarly, when accessing notifications through the @Web3Inbox.com app, users will be met with the same request for each application they choose to subscribe to. This feature not only enables users to experience a customized, ‘app-by-app’ approach to staying connected in web3, but also ensures they only ever hear from the apps they choose to — no unsolicited notifications or spam from unknown senders. Its their curated inbox, connected with only those they choose.

### Setting customized notification preferences

Once users have subscribed to their chosen apps, they have the option to define and set which types of notifications they receive from those apps. For example, a user may wish to receive only information regarding changes to their portfolio from a DEX, or, they might want to receive notifications from an NFT marketplace — but only notifications regarding their own NFT collections. In these scenarios, they’ll have the ability to disable other notification types, like marketing updates, and ensure their feed is curated to show only information that’s meaningful to them. As apps set their own notification types, they have unlimited optionality to really build out a notification structure they know can support their users’ needs — no ‘one size fits all’ approach, but a personable, community-oriented structure that puts both app and user needs’ at the forefront of communication.

### Rate limiting

Apps are limited to a maximum number of notifications they’re able to send to their community. Specifically, apps may send accounts notifications twice an hour on average, but may exceed that average in bursts of up to 50 at a time.

## Our continued pledge on spam protection

We're constantly working on improving and growing our products, and we have a number of impactful anti-spam features and functions in the works set to increase the overall protection and user experience of WalletKit Notification users:

### User reporting

Users will have the ability to report applications that appear to be acting or engaging with their community in a malicious or suspicious manner. Projects that are flagged as malicious may be removed from the WalletKit Notification discover page and have notification functionality disabled.



================================================
File: walletkit/react-native/notifications/notify/usage.mdx
================================================
---
title: Usage
---

import CloudBanner from "/snippets/cloud-banner.mdx";

In this section, we showcase the aspects of using the Notify API. We'll guide you through the initial steps of initializing the Notify client and logging in a blockchain account. You'll also learn how to manage your subscriptions and messages. Additionally, we cover the process of setting up and displaying push notifications on your preferred platform. To ensure a good user experience, we include best practices for spam protection, helping you to enable the users to maintain control over the notifications wallet receives.

## Content

Links to sections on this page. Some sections are platform specific and are only visible when the platform is selected. To view a summary of useful platform specific topics, check out Extra (Platform Specific) under this section.

- @Initialization:
  Creating a new Notify Client instance and initializing it with a projectId from @Cloud.
- @Account login:
  A SIWE message must be signed by the user in order to authorize the client to use Notify API
- @Subscribing to a new dapp:
  Opt-in to receive notifications from dapp
- @Fetching active subscriptions:
  Get active subscriptions
- @Fetching subscription’s notification:
  Get notifications of a subscription
- @Fetching available notification types:
  Get latest notification types
- @Updating subscriptions notification settings:
  Change allowed notification types sent by dapp
- @Unsubscribe from a dapp:
  Opt-out from receiving notifications from a dapp
- @Account logout:
  To stop receiving notifications to this client, accounts can logout of using Notify API
- @Push Notification Setup:
  Configuring app in order to decrypt notifications

## Initialization

<CloudBanner />

#### Initialize the SDK clients

```javascript
import { NotifyClient } from "@walletconnect/notify-client";

const notifyClient = await NotifyClient.init({
  projectId: "<YOUR PROJECT ID>",
});
```

## Add listeners for relevant events

```javascript
// Handle response to a `notifyClient.subscribe(...)` call
notifyClient.on("notify_subscription", async ({ params }) => {
  const { error } = params;

  if (error) {
    // Setting up the subscription failed.
    // Inform the user of the error and/or clean up app state.
    console.error("Setting up subscription failed: ", error);
  } else {
    // New subscription was successfully created.
    // Inform the user and/or update app state to reflect the new subscription.
    console.log(`Subscribed successfully.`);
  }
});

// Handle an incoming notification
notifyClient.on("notify_message", ({ params }) => {
  const { message } = params;
  // e.g. build a notification using the metadata from `message` and show to the user.
});

// Handle response to a `notifyClient.update(...)` call
notifyClient.on("notify_update", ({ params }) => {
  const { error } = params;

  if (error) {
    // Updating the subscription's scope failed.
    // Inform the user of the error and/or clean up app state.
    console.error("Setting up subscription failed: ", error);
  } else {
    // Subscription's scope was updated successfully.
    // Inform the user and/or update app state to reflect the updated subscription.
    console.log(`Successfully updated subscription scope.`);
  }
});

// Handle a change in the existing subscriptions (e.g after a subscribe or update)
notifyClient.on("notify_subscriptions_changed", ({ params }) => {
  const { subscriptions } = params;
  // `subscriptions` will contain any *changed* subscriptions since the last time this event was emitted.
  // To get a full list of subscriptions for a given account you can use `notifyClient.getActiveSubscriptions({ account: 'eip155:1:0x63Be...' })`
});
```

## Account login

In order to register account in Notify API to be able to subscribe to any dapp to start receiving notifications, account needs to sign SIWE message to prove ownership. Developers can check if an account is registered by calling **`isRegistered()`** function. If the account is not registered, developers should call **`prepareRegistration()`** and then **`register()`** function to register the account.

<Info>
This is a one-time action per account. It does not need to be repeated after initial registration of the new account.
</Info>

### Registering as a wallet

```javascript
const account = `eip155:1:0x63Be2c680685d2A9620c11b0068291261aa62d76`
const domain =  'app.mydomain.com', // pass the domain (i.e. the hostname) where your dapp is hosted.
const allApps =  true // The user will be prompted to authorize this wallet to send and receive messages on their behalf for ALL domains using their WalletConnect identity.

// No need to register and sign message if already registered.
if (notifyClient.isRegistered({ account, domain, allApps })) return;

const {registerParams, message}  = notifyClient.prepareRegistration({
  account,
  domain,
  allApps
});

const signature = await ethersWallet.signMessage(message);

await notifyClient.register({
  registerParams,
  signature,
})
```

## Subscribing to a new dapp

To begin receiving notifications from a dapp, users must opt-in by subscribing. This subscription process grants permission for the dapp to send notifications to the user. These notifications can serve a variety of purposes, such as providing updates on the user's blockchain account activities or informing them about ongoing campaigns within the dapp. Upon initial subscription, clients will be automatically enrolled to receive all types of notifications as defined by the dapp at that moment. Users have the flexibility to modify their notification settings later, allowing them to tailor the types of alerts they receive according to their preferences.

<Note>
To identify dapps that can be subscribed to via Notify, we can query the following Explorer API endpoint:

https://explorer-api.walletconnect.com/v3/dapps?projectId=YOUR_PROJECT_ID&is_notify_enabled=true
</Note>

```javascript
// Get the domain of the target dapp from the Explorer API response
const appDomain = new URL(fetchedExplorerDapp.platform_browser).hostname;

// Subscribe to `fetchedExplorerDapp` by passing the account to be subscribed and the domain of the target dapp.
await notifyClient.subscribe({
  account,
  appDomain,
});

// -> Success/Failure will be received via the `notify_update` event registered previously.
// -> New subscription will be emitted via the `notify_subscriptions_changed` watcher event.
```

## Fetching active subscriptions

To fetch the current list of subscriptions an account has, call **`getActiveSubscriptions()`**.

```javascript
// Will return all active subscriptions for the provided account, keyed by subscription topic.
const accountSubscriptions = notifyClient.getActiveSubscriptions({
  account: `eip155:1:0x63Be...`,
});
```

## Fetching subscription's notifications

To fetch subscription's notifications by calling **`getNotificationHistory()`**.

```javascript
const notifications = notifyClient.getNotificationHistory(account);
```

## Fetching available notification types

Developers can fetch latest notification types specified by dapp by calling **`getNotificationTypes()`** function.

You can use the `scope` object of the subscription to get the available notification types.

```typescript
// get notification types by accessing `scope` member of a dapp's subscription
const notificationTypes = notifyClient
  .getActiveSubscriptions({ account })
  .filter((subscription) => subscription.topic === topic).scope;
```

## Updating subscriptions notification settings

Users can alter their notification settings to filter out unwanted alerts from a dapp. During this process, they review and select the types of notifications they wish to receive, based on the latest options provided by the dapp. Available notification types fetching is shown in the @next section.

```javascript
// `topic` - subscription topic of the subscription that should be updated.
// `scope` - an array of notification types that should be enabled going forward. The current scopes can be found under `subscription.scope`.
await notifyClient.update({
  topic,
  scope: ["alerts"],
});
```

## Unsubscribe from a dapp

To opt-out of receiving notifications from a dap, a user can decide to unsubscribe from dapp.

```javascript
notifyClient.deleteSubscription({
  topic: "subscription_topic_to_unsubscribe_from",
});
```

## Account logout

If an account is removed from the client or a user no longer wants to receive notifications for this account, you can logout the account from Notify API by calling **`unregister()`**. This will remove all subscriptions and messages for this account from the client’s storage.

```javascript
const account = `eip155:1:0x63Be2c680685d2A9620c11b0068291261aa62d76`;

await notifyClient.unregister({
  account,
});
```

## Fetch notification history (Pagination)

There might be different approaches to implement pagination in your app depending on your needs. You can see the following example implemented with `FlatList` which introduces infinite scroll functionality with a basic example:

Please make sure you have better handling of the notify client instance which handles worst cases by checking initialization status, account status, for production ready apps.

```javascript
export default function SubscriptionDetailsScreen() {
  const {topic} = useRoute().params as {topic: string};
  const [notifications, setNotifications] = React.useState([]);
  const [hasMore, setHasMore] = React.useState(false);
  const [isLoading, setIsLoading] = React.useState(false);

  const lastItem = notifications?.[notifications.length - 1]?.id;

  async function getNotificationHistory(startingAfter?: string) {
    setIsLoading(true);

    const notificationHistory = await notifyClient.getNotificationHistory({
      topic,
      limit: 15,
      startingAfter,
    });

    setNotifications(
      prevNotifications => prevNotifications.concat(notificationHistory.notifications),
    );
    setHasMore(notificationHistory.hasMore);
    setIsLoading(false);

    return notificationHistory;
  }

  React.useEffect(() => {
    getNotificationHistory();
  }, [topic]);

  return (
    <FlatList
      data={sortedByDate}
      keyExtractor={item => item.sentAt.toString()}
      onEndReached={() => {
        if (hasMore && lastItem) {
          getNotificationHistory(lastItem)
        }
      }}
      ListFooterComponent={() => {
        if (!isLoading) return null
        return <NotifcationItemSkeleton />
      }}
      renderItem={({item}) => (
        <NotificationItem key={item.id} item={item} />
      )}
    />
  );
}
```

### Push Notification Setup

Install @`@react-native-firebase/app`, @`@react-native-firebase/messaging` and @`@notifee/react-native` to handle Push Notifications.
Please refer to the respective package documentation to configure them properly.

```
yarn add @notifee/react-native @react-native-firebase/app @react-native-firebase/messaging
```

Update your `index.js` file to include the following logic.

```js
import { AppRegistry } from "react-native";
import { name as appName } from "./app.json";
import crypto from "react-native-quick-crypto";

import messaging from "@react-native-firebase/messaging";
import notifee, {
  AndroidImportance,
  AndroidVisibility,
  EventType,
} from "@notifee/react-native";
import { NotifyClient } from "@walletconnect/notify-client";
import { decryptMessage } from "@walletconnect/notify-message-decrypter";

import App from "./src/App";

const polyfillDigest = async (algorithm, data) => {
  const algo = algorithm.replace("-", "").toLowerCase();
  const hash = crypto.createHash(algo);
  hash.update(data);
  return hash.digest();
};

globalThis.crypto = crypto;
globalThis.crypto.subtle = {
  digest: polyfillDigest,
};

// Create notification channel (Android only feature)
notifee.createChannel({
  id: "default",
  name: "Default Channel",
  lights: false,
  vibration: true,
  importance: AndroidImportance.HIGH,
  visibility: AndroidVisibility.PUBLIC,
});

let notifyClient;

const projectId = process.env.ENV_PROJECT_ID;

async function registerAppWithFCM() {
  // This is expected to be automatically handled on iOS. See https://rnfirebase.io/reference/messaging#registerDeviceForRemoteMessages
  if (Platform.OS === "android") {
    await messaging().registerDeviceForRemoteMessages();
  }
}

async function registerClient(deviceToken, clientId) {
  const body = JSON.stringify({
    client_id: clientId,
    token: deviceToken,
    type: "fcm",
    always_raw: true,
  });

  const requestOptions = {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body,
  };

  return fetch(
    `https://echo.walletconnect.com/${projectId}/clients`,
    requestOptions
  )
    .then((response) => response.json())
    .then((result) => console.log(">>> registered client", result))
    .catch((error) => console.log(">>> error while registering client", error));
}

async function handleGetToken(token) {
  const status = await messaging().requestPermission();
  const enabled =
    status === messaging.AuthorizationStatus.AUTHORIZED ||
    status === messaging.AuthorizationStatus.PROVISIONAL;

  if (enabled) {
    notifyClient = await NotifyClient.init({ projectId });
    const clientId = await notifyClient.core.crypto.getClientId();
    return registerClient(token, clientId);
  }
}

messaging().getToken().then(handleGetToken);
messaging().onTokenRefresh(handleGetToken);

async function onMessageReceived(remoteMessage) {
  if (!remoteMessage.data?.blob || !remoteMessage.data?.topic) {
    console.log("Missing blob or topic on notification message.");
    return;
  }

  const decryptedMessage = await decryptMessage({
    topic: remoteMessage.data?.topic,
    encryptedMessage: remoteMessage.data?.blob,
  });

  return notifee.displayNotification({
    title: decryptedMessage.title,
    body: decryptedMessage.body,
    id: "default",
    android: {
      channelId: "default",
      importance: AndroidImportance.HIGH,
      visibility: AndroidVisibility.PUBLIC,
      smallIcon: "ic_launcher", // optional, defaults to 'ic_launcher'.
      // pressAction is needed if you want the notification to open the app when pressed. See https://notifee.app/react-native/docs/ios/interaction#press-action
      pressAction: {
        id: "default",
      },
    },
  });
}

messaging().onMessage(onMessageReceived);
messaging().setBackgroundMessageHandler(onMessageReceived);

notifee.onBackgroundEvent(async ({ type, detail }) => {
  const { notification, pressAction } = detail;

  // Check if the user pressed the "Mark as read" action
  if (type === EventType.ACTION_PRESS && pressAction.id === "mark-as-read") {
    // Remove the notification
    await notifee.cancelNotification(notification.id);
  }
});

function HeadlessCheck({ isHeadless }) {
  if (isHeadless) {
    // App has been launched in the background by iOS, ignore
    return null;
  }

  // Render the app component on foreground launch
  return <App />;
}

AppRegistry.registerComponent(appName, () => HeadlessCheck);
```


================================================
File: walletkit/best-practices.mdx
================================================
---
sidebarTitle: Best Practices
title: Best Practices for Wallets
---

To ensure the smoothest and most seamless experience for our users, Reown is committed to working closely with wallet providers to encourage the adoption of our recommended best practices.

By implementing these guidelines, we aim to optimize performance and minimize potential challenges, even in suboptimal network conditions.

We are actively partnering with wallet developers to optimize performance in scenarios such as:

1. **Success and Error Messages** - Users need to know what’s going on, at all times. Too much communication is better than too little. The less users need to figure out themselves or assume what’s going on, the better.
2. **(Perceived) Latency** - A lot of factors can influence latency (or perceived latency), e.g. network conditions, position in the boot chain, waiting on the wallet to connect or complete a transaction and not knowing if or when it has done it.
3. **Old SDK Versions** - Older versions can have known and already fixed bugs, leading to unnecessary issues to users, which can be simply and quickly solved by updating to the latest SDK.

To take all of the above into account and to make experience better for users, we've put together some key guidelines for wallet providers. These best practices focus on the most important areas for improving user experience.

Please follow these best practices and make the experience for your users and yourself a delightful and quick one.

## Checklist Before Going Live

To make sure your wallet adheres to the best practices, we recommend implementing the following checklist before going live. You can find more detailed information on each point below.

1. **Success and Error Messages**
   - ✅ Display clear and concise messages for all user interactions
   - ✅ Provide feedback for all user actions
     - ✅ Connection success
     - ✅ Connection error
     - ✅ Loading indicators for waiting on connection, transaction, etc.
   - ✅ Ensure that users are informed of the status of their connection and transactions
   - ✅ Implement status indicators internet availability
   - ✅ Make sure to provide feedback not only to users but also back to the dapp (e.g., if there's an error or a user has not enough funds to pay for gas, don't just display the info message to the user, but also send the error back to the dapp so that it can change the state accordingly)
2. **Mobile Linking**
   - ✅ Implement mobile linking to allow for automatic redirection between the wallet and the dapp
   - ✅ Use deep linking over universal linking for a better user experience
   - ✅ Ensure that the user is redirected back to the dapp after completing a transaction
3. **Latency**
   - ✅ Optimize performance to minimize latency
   - ✅ Latency for connection in normal conditions: under 5 seconds
   - ✅ Latency for connection in poor network (3G) conditions: under 15 seconds
   - ✅ Latency for signing in normal conditions: under 5 seconds
   - ✅ Latency for signing in poor network (3G) conditions: under 10 seconds
4. **Verify API**
   - ✅ Present users with four key states that can help them determine whether the domain they’re about to connect to might be malicious (Domain match, Unverified, Mismatch, Threat)
5. **Latest SDK Version**
   - ✅ Ensure that you are using the latest SDK version
   - ✅ Update your SDK regularly to benefit from the latest features and bug fixes
   - ✅ Subscribe to SDK updates to stay informed about new releases

## 1. Success and Error Messages

Users often face ambiguity in determining whether their connection or transactions were successful. They are also not guided to switching back into the dapp or automatically switched back when possible, causing unnecessary user anxiety. Additionally, wallets typically lack status indicators for connection and internet availability, leaving users in the dark.
<Frame caption="An example of a successful connection message in a Rainbow wallet">
![](/images/assets/connection-successful.png)
</Frame>
### Pairing

A pairing is a connection between a wallet and a dapp that has fixed permissions to only allow a dapp to propose a session through it. Dapp can propose infinite number of sessions on one pairing. Wallet must use a pair method from WalletKit client to pair with dapp.

<Tabs>
  <Tab title="Web" >
```jsx
const uri = 'xxx'; // pairing uri
try {
    await walletKit.pair({ uri });
} catch (error) {
    // some error happens while pairing - check Expected errors section
}
```
</Tab>
<Tab title="React Native">
```jsx
const uri = 'xxx'; // pairing uri
try {
    await walletKit.pair({ uri });
} catch (error) {
    // some error happens while pairing - check Expected errors section
}
```
</Tab>
<Tab title="iOS">
```swift
let uri = WalletConnectURI(string: urlString)

if let uri {
Task {
try await WalletKit.instance.pair(uri: uri)
}
}

````
</Tab>
<Tab title="Android">
```kotlin
val pairingParams = Wallet.Params.Pair(pairingUri)
WalletKit.pair(pairingParams,
    onSuccess = {
        //Subscribed on the pairing topic successfully. Wallet should await for a session proposal
    },
    onError = { error ->
        //Some error happens while pairing - check Expected errors section
    }
}
````

</Tab>
</Tabs>

#### Pairing Expiry

A pairing expiry event is triggered whenever a pairing is expired. The expiry for inactive pairing is 5 mins, whereas for active pairing is 30 days. A pairing becomes active when a session proposal is received and user successfully approves it. This event helps to know when given pairing expires and update UI accordingly.

<Tabs>
  <Tab title="Web" >
```typescript
core.pairing.events.on("pairing_expire", (event) => {
    // pairing expired before user approved/rejected a session proposal
    const { topic } = topic;
});
```
  </Tab>
  <Tab title="React Native">
```typescript
core.pairing.events.on("pairing_expire", (event) => {
    // pairing expired before user approved/rejected a session proposal
    const { topic } = topic;
});
```
  </Tab>
  <Tab title="iOS">
```Swift
WalletKit.instance.pairingExpirationPublisher
    .receive(on: DispatchQueue.main)
    .sink { pairing in
    guard !pairing.active else { return }
    // let user know that pairing has expired
}.store(in: &publishers)
```
  </Tab>
  <Tab title="Android" label="Android">
```kotlin
val coreDelegate = object : CoreClient.CoreDelegate {
    override fun onPairingExpired(expiredPairing: Core.Model.ExpiredPairing) {
        // Here a pairing expiry is triggered
    }
    // ...other callbacks
}

CoreClient.setDelegate(coreDelegate)

````
</Tab>
</Tabs>

#### Pairing messages

1. Consider displaying a successful pairing message when pairing is successful. Before that happens, wallet should show a loading indicator.
2. Display an error message when a pairing fails.

#### Expected Errors

While pairing, the following errors might occur:

- **No Internet connection error or pairing timeout when scanning QR with no Internet connection**
  - User should pair again with Internet connection
- **Pairing expired error when scanning a QR code with expired pairing**
  - User should refresh a QR code and scan again
- **Pairing with existing pairing is not allowed**
  - User should refresh a QR code and scan again. It usually happens when user scans an already paired QR code.

### Session Proposal

A session proposal is a handshake sent by a dapp and its purpose is to define session rules. Whenever a user wants to establish a connection between a wallet and a dapp, one should approve a session proposal.

Whenever user approves or rejects a session proposal, a wallet should show a loading indicator the moment the button is pressed, until Relay acknowledgement is received for any of these actions.

#### Approving session

<Tabs>
  <Tab title="Web" >
```typescript
try {
    await walletKit.approveSession(params);
    // update UI -> remove the loader
} catch (error) {
    // present error to the user
}
````

  </Tab>
  <Tab title="React Native">
```typescript
try {
    await walletKit.approveSession(params);
    // update UI -> remove the loader
} catch (error) {
    // present error to the user
}
```
  </Tab>
  <Tab title="iOS">
```swift
do {
    try await WalletKit.instance.approve(proposalId: proposal.id, namespaces: sessionNamespaces, sessionProperties: proposal.sessionProperties)
    // Update UI, remove loader
} catch {
    // present error
}
```
  </Tab>
  <Tab title="Android" label="Android">
```kotlin
WalletKit.approveSession(approveProposal,
  onSuccess = {
    //Session approval response was sent successfully - update your UI
  }
    onError = { error ->
      //Error while sending session approval - update your UI
  })
```
</Tab>
</Tabs>

#### Rejecting session

<Tabs>
  <Tab title="Web" >
```typescript
try {
    await walletKit.rejectSession(params);
    // update UI -> remove the loader
} catch (error) {
    // present error to the user
}
```
  </Tab>
  <Tab title="React Native">
```typescript
try {
    await walletKit.rejectSession(params);
    // update UI -> remove the loader
} catch (error) {
    // present error to the user
}
```
  </Tab>
  <Tab title="iOS">
```swift
do {
    try await WalletKit.instance.reject(proposalId: proposal.id, reason: .userRejected)
    // Update UI, remove loader
} catch {
    // present error
}
```
  </Tab>
  <Tab title="Android" label="Android">
```kotlin
WalletKit.rejectSession(reject,
  onSuccess = {
      //Session rejection response was sent successfully - update your UI
  },
  onError = { error ->
        //Error while sending session rejection - update your UI
  })
```
</Tab>
</Tabs>

#### Session proposal expiry

A session proposal expiry is 5 minutes. It means a given proposal is stored for 5 minutes in the SDK storage and user has 5 minutes for the approval or rejection decision. After that time, the below event is emitted and proposal modal should be removed from the app's UI.

<Tabs>
  <Tab title="Web" >
```typescript
walletKit.on("proposal_expire", (event) => {
    // proposal expired and any modal displaying it should be removed
    const { id } = event;
});
```
  </Tab>
  <Tab title="React Native">
```typescript
walletKit.on("proposal_expire", (event) => {
    // proposal expired and any modal displaying it should be removed
    const { id } = event;
});
```
  </Tab>
  <Tab title="iOS">
```swift
WalletKit.instance.sessionProposalExpirationPublisher.sink { _ in
    // let user know that session proposal has expired, update UI
}.store(in: &publishers)
```
  </Tab>
  <Tab title="Android" label="Android">
```kotlin
val walletDelegate = object : WalletKit.WalletDelegate {
  override fun onProposalExpired(proposal: Wallet.Model.ExpiredProposal) {
          // Here this event is triggered when a proposal expires - update your UI
  }
  // ...other callbacks
}
WalletKit.setWalletDelegate(walletDelegate)
```
</Tab>
</Tabs>

#### Session Proposal messages

1. Consider displaying a successful session proposal message before redirecting back to the dapp. Before the success message is displayed, wallet should show a loading indicator.
2. Display an error message when session proposal fails.

#### Expected errors

While approving or rejecting a session proposal, the following errors might occur:

- **No Internet connection**
  - It happens when a user tries to approve or reject a session proposal with no Internet connection
- **Session proposal expired**
  - It happens when a user tries to approve or reject an expired session proposal
- **Invalid @namespaces**
  - It happens when a validation of session namespaces fails
- **Timeout**
  - It happens when Relay doesn't acknowledge session settle publish within 10s

### Session Request

A session request represents the request sent by a dapp to a wallet.

Whenever user approves or rejects a session request, a wallet should show a loading indicator the moment the button is pressed, until Relay acknowledgement is received for any of these actions.

<Tabs>
  <Tab title="Web" >
```typescript
try {
    await walletKit.respondSessionRequest(params);
    // update UI -> remove the loader
} catch (error) {
    // present error to the user
}
```
  </Tab>
  <Tab title="React Native">
```typescript
try {
    await walletKit.respondSessionRequest(params);
    // update UI -> remove the loader
} catch (error) {
    // present error to the user
}
```
  </Tab>
  <Tab title="iOS">
```swift
do {
    try await WalletKit.instance.respond(requestId: request.id, signature: signature, from: account)
    // update UI -> remove the loader
} catch {
    // present error to the user
}
```
  </Tab>
  <Tab title="Android" label="Android">
```kotlin
WalletKit.respondSessionRequest(Wallet.Params.SessionRequestResponse,
  onSuccess = {
      //Session request response was sent successfully - update your UI
  },
  onError = { error ->
      //Error while sending session response - update your UI
  })
```
</Tab>
</Tabs>

#### Session request expiry

A session request expiry is defined by a dapp. Its value must be between `now() + 5mins` and `now() + 7 days`. After the session request expires, the below event is emitted and session request modal should be removed from the app's UI.

<Tabs>
  <Tab title="Web" >
```typescript
walletKit.on("session_request_expire", (event) => {
  // request expired and any modal displaying it should be removed
  const { id } = event;
});
```
  </Tab>
  <Tab title="React Native">
```typescript
walletKit.on("session_request_expire", (event) => {
  // request expired and any modal displaying it should be removed
  const { id } = event;
});
```
  </Tab>
  <Tab title="iOS">
```swift
WalletKit.instance.requestExpirationPublisher.sink { _ in
    // let user know that request has expired
}.store(in: &publishers)
```
  </Tab>
  <Tab title="Android" label="Android">
```kotlin
val walletDelegate = object : WalletKit.WalletDelegate {
  override fun onRequestExpired(request: Wallet.Model.ExpiredRequest) {
      // Here this event is triggered when a session request expires - update your UI
  }
  // ...other callbacks
}
WalletKit.setWalletDelegate(walletDelegate)
```
</Tab>
</Tabs>

#### Expected errors

While approving or rejecting a session request, the following errors might occur:

- **Invalid session**
  - This error might happen when a user approves or rejects a session request on an expired session
- **Session request expired**
  - This error might happen when a user approves or rejects a session request that already expired
- **Timeout**
  - It happens when Relay doesn't acknowledge session settle publish within 10 seconds

### Connection state

The Web Socket connection state tracks the connection with the Relay server. An event is emitted whenever a connection state changes.

<Tabs>
  <Tab title="Web" >
```typescript
core.relayer.on("relayer_connect", () => {
  // connection to the relay server is established
})

core.relayer.on("relayer_disconnect", () => {
// connection to the relay server is lost
})

````
  </Tab>
  <Tab title="React Native">
```typescript
core.relayer.on("relayer_connect", () => {
  // connection to the relay server is established
})

core.relayer.on("relayer_disconnect", () => {
  // connection to the relay server is lost
})
````

  </Tab>
  <Tab title="iOS">
```swift
WalletKit.instance.socketConnectionStatusPublisher
  .receive(on: DispatchQueue.main)
  .sink { status in
  switch status {
  case .connected:
    // ...
  case .disconnected:
    // ...
  }
}.store(in: &publishers)
```
  </Tab>
  <Tab title="Android" label="Android">
```kotlin
val walletDelegate = object : WalletKit.WalletDelegate {
  override fun onConnectionStateChange(state: Wallet.Model.ConnectionState) {
    // Here this event is triggered when a connection state has changed
  }
  // ...other callbacks
}
WalletKit.setWalletDelegate(walletDelegate)
```
</Tab>
</Tabs>

#### Connection state messages

When the connection state changes, show a message in the UI. For example, display a message when the connection is lost or re-established.

## 2. Mobile Linking

### Why use Mobile Linking?

Mobile Linking uses the mobile device’s native OS to automatically redirect between the native wallet app and a native app. This results in few user actions a better UX.

#### Establishing Communication Between Mobile Wallets and Apps

When integrating a wallet with a mobile application, it's essential to understand how they communicate. The process involves two main steps:

1. **QR Code Handshake:** The mobile app (Dapp) generates a unique URI (Uniform Resource Identifier) and displays it as a QR code. This URI acts like a secret handshake. When the user scans the QR code or copy/pastes the URI using their wallet app, they establish a connection. It's like saying, "Hey, let's chat!"
2. **Deep Links and Universal Links:** The URI from the QR code allows the wallet app to create a @deep link or @universal link. These links work on both Android and iOS. They enable seamless communication between the wallet and the app.

<Tip>

**Developers should prefer Deep Linking over Universal Linking.**

Universal Linking may redirect the user to a browser, which might not provide the intended user experience. Deep Linking ensures the user is taken directly to the app.

</Tip>

### Key Behavior to Address

In some scenarios, wallets use redirect metadata provided in session proposals to open applications. This can cause unintended behavior, such as:

Redirecting to the wrong app when multiple apps share the same redirect metadata (e.g., a desktop and mobile version of the same Dapp).
Opening an unrelated application if a QR code is scanned on a different device than where the wallet is installed.

#### Recommended Approach

To avoid this behavior, wallets should:

- **Restrict Redirect Metadata to Deep Link Use Cases**: Redirect metadata should only be used when the session proposal is initiated through a deep link. QR code scans should not trigger app redirects using session proposal metadata.

### Connection Flow

1. **Dapp prompts user:** The Dapp asks the user to connect.
2. **User chooses wallet:** The user selects a wallet from a list of compatible wallets.
3. **Redirect to wallet:** The user is redirected to their chosen wallet.
4. **Wallet approval:** The wallet prompts the user to approve or reject the session (similar to granting permission).
5. **Return to dapp:**
   - **Manual return:** The wallet asks the user to manually return to the Dapp.
   - **Automatic return:** Alternatively, the wallet automatically takes the user back to the Dapp.
6. **User reunites with dapp:** After all the interactions, the user ends up back in the Dapp.
<Frame>
![](/images/w3w/mobileLinking-light.png)
</Frame>

### Sign Request Flow

When the Dapp needs the user to sign something (like a transaction), a similar pattern occurs:

1. **Automatic redirect:** The Dapp automatically sends the user to their previously chosen wallet.
2. **Approval prompt:** The wallet asks the user to approve or reject the request.
3. **Return to dapp:**
   - **Manual return:** The wallet asks the user to manually return to the Dapp.
   - **Automatic return:** Alternatively, the wallet automatically takes the user back to the Dapp.
4. **User reconnects:** Eventually, the user returns to the Dapp.

<Frame>
![](/images/w3w/mobileLinking_sign-light.png)
</Frame>


### Platform Specific Preparation

<Tabs>
  <Tab title="iOS">
  Read the specific steps for iOS here: @Platform preparations
  </Tab>

<Tab title="Android" label="Android">
  Read the specific steps for Android here: [Platform
  preparations](./android/mobile-linking#platform-preparations)
</Tab>

<Tab title="Flutter" label="Flutter">
  Read the specific steps for Flutter here: [Platform
  preparations](./flutter/mobile-linking#platform-preparations)
</Tab>

  <Tab title="React Native">
  Read the specific steps for React Native here: @Platform preparations
  </Tab>
</Tabs>

### How to Test

To experience the desired behavior, try our Sample Wallet and Dapps which use our Mobile linking best practices. These are available on all platforms.

Once you have completed your integration, you can test it against our sample apps to see if it is working as expected. Download the app and and try your mobile linking integration on your device.

<Tabs>
  <Tab title="iOS">
  - @Sample Wallet - on TestFlight
  - @Sample DApp - on TestFlight
  </Tab>

<Tab title="Android" label="Android">
  - @Sample Wallet -
  on Firebase - [Sample
  DApp](https://appdistribution.firebase.dev/i/5e4fe4b30c8a208d) - on Firebase
</Tab>

<Tab title="Flutter" label="Flutter">
  - Sample Wallet: - [Sample Wallet for
  iOS](https://testflight.apple.com/join/Uv0XoBuD) - [Sample Wallet for
  Android](https://appdistribution.firebase.dev/i/2b8b3dce9e2831cd) - AppKit
  DApp: - @AppKit Dapp for iOS -
  [AppKit Dapp for
  Android](https://appdistribution.firebase.dev/i/2c6573f6956fa7b5)
</Tab>

  <Tab title="React Native">
  - Sample Wallet:
    - @Sample Wallet for Android
  - Sample DApp:
    - @Sample App for iOS
    - @Sample App for Android
  </Tab>
</Tabs>

## 3. Latency

Our SDK’s position in the boot chain can lead to up to 15 seconds in throttled network conditions. Lack of loading indicators exacerbates the perceived latency issues, impacting user experience negatively. Additionally, users often do not receive error messages or codes when issues occur or timeouts happen.

### Target latency

For **connecting**, the target latency is:

- **Under 5 seconds** in normal conditions
- **Under 15 seconds** when throttled (3G network speed)

For **signing**, the target latency is:

- **Under 5 seconds** in normal conditions
- **Under 10 seconds** when throttled (3G network speed)

### How to test

To test latency under suboptimal network conditions, you can enable throttling on your mobile phone. You can simulate different network conditions to see how your app behaves in various scenarios.

For example, on iOS you need to enable Developer Mode and then go to **Settings > Developer > Network Link Conditioner**. You can then select the network condition you want to simulate. For 3G, you can select **3G** from the list, for no network or timeout simulations, choose **100% Loss**.

Check this article for how to simulate slow internet connection on iOS & Android, with multiple options for both platforms: @How to simulate slow internet connection on iOS & Android.

## 4. Verify API

Verify API is a security-focused feature that allows wallets to notify end-users when they may be connecting to a suspicious or malicious domain, helping to prevent phishing attacks across the industry. Once a wallet knows whether an end-user is on uniswap.com or eviluniswap.com, it can help them to detect potentially harmful connections through Verify's combined offering of Reown's domain registry.

When a user initiates a connection with an application, Verify API enables wallets to present their users with four key states that can help them determine whether the domain they’re about to connect to might be malicious.

Possible states:

- Domain match
- Unverified
- Mismatch
- Threat
<Frame>
!@Verify States
</Frame>
<Frame>
!@Verify States
</Frame>

<Note>

Verify API is not designed to be bulletproof but to make the impersonation attack harder and require a somewhat sophisticated attacker. We are working on a new standard with various partners to close those gaps and make it bulletproof.

</Note>

### Domain risk detection[](https://docs.reown.com/walletkit/web/verify#domain-risk-detection)

The Verify security system will discriminate session proposals & session requests with distinct validations that can be either `VALID`, `INVALID` or `UNKNOWN`.

- **Domain match:** The domain linked to this request has been verified as this application's domain.
  - This interface appears when the domain a user is attempting to connect to has been ‘verified’ in our domain registry as the registered domain of the application the user is trying to connect to, and the domain has not returned as suspicious from either of the security tools we work with. The `verifyContext` included in the request will have a validation of `VALID`.
- **Unverified:** The domain sending the request cannot be verified.
  - This interface appears when the domain a user is attempting to connect to has not been verified in our domain registry, but the domain has not returned as suspicious from either of the security tools we work with. The `verifyContext` included in the request will have a validation of `UNKNOWN`.
- **Mismatch:** The application's domain doesn't match the sender of this request.
  - This interface appears when the domain a user is attempting to connect to has been flagged as a different domain to the one this application has verified in our domain registry, but the domain has not returned as suspicious from either of the security tools we work with. The `verifyContext` included in the request will have a validation of `INVALID`
- **Threat:** This domain is flagged as malicious and potentially harmful.
  - This interface appears when the domain a user is attempting to connect to has been flagged as malicious on one or more of the security tools we work with. The `verifyContext` included in the request will contain parameter `isScam` with value `true`.

### Verify API Implementation

To see how to implement Verify API for your framework, see @Verify API page and select your platform to see code examples.

### How to test

To test Verify API with a malicious domain, you can check out the @Malicious React dapp, created specifically for testing. This app is flagged as malicious and will have the `isScam` parameter set to `true` in the `verifyContext` of the request. You can use this app to test how your wallet behaves when connecting to a malicious domain.

### Error messages
<Frame>
!@Verify API flagged domain
</Frame>
_A sample error warning when trying to connect to a malicious domain_

## 5. Latest SDK

Numerous features have been introduced, bugs have been identified and fixed over time, stability has improved, but many dapps and wallets continue to use older SDK versions with known issues, affecting overall reliability.

Make sure you are using the latest version of the SDK for your platform

<Tabs>
  <Tab title="iOS">
  - **WalletConnectSwiftV2**: @Latest release
  </Tab>

<Tab title="Android" label="Android">
  - **WalletConnectKotlinV2**: [Latest
  release](https://github.com/WalletConnect/WalletConnectKotlinV2/releases/latest)
</Tab>

<Tab title="Flutter" label="Flutter">
  - **WalletConnectFlutterV2**: [Latest
  release](https://github.com/WalletConnect/WalletConnectFlutterV2/releases/latest)
</Tab>

  <Tab title="React Native">
  - **AppKit for React Native**: @Latest release
  </Tab>
</Tabs>

### Subscribe to updates

To stay up to date with the latest SDK releases, you can use GitHub's native feature to subscribe to releases. This way, you will be notified whenever a new release is published. You can find the "Watch" button on the top right of the repository page. Click on it, then select "Custom" and "Releases only". You'll get a helpful ping whenever a new release is out.

!@Subscribe to releases

## Resources

- @React Wallet - for testing dapps, features, Verify API messages, etc.
- @React dapp - for testing wallets
- @Malicious React dapp - for testing Verify API with malicious domain




