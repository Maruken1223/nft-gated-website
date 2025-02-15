# NFT Gated Website

[webサイトを見る](https://nft-gated-website-five.vercel.app/login)
<img width="1440" alt="スクリーンショット 2023-05-05 11 37 17" src="https://user-images.githubusercontent.com/113430420/236367094-7cfd7424-7a2f-4ce1-975a-790b7ac8cadd.png">
>

This project demonstrates how you can restrict content on your website to only those users who own an NFT from your collection.

We use an [Edition Drop](https://portal.thirdweb.com/pre-built-contracts/edition-drop) contract to enable users to claim one of the NFTs, and serve users the restricted content if they have at least one of the NFTs claimed.

## Tools:

- [React SDK](https://docs.thirdweb.com/react): To access the connected wallet, switch the user's network, and claim an NFT from our Edition Drop collection.
- [Auth](https://portal.thirdweb.com/auth): To ask users to sign a message and verify they own the wallet they claim to be, while on the server-side.

## Using This Template

Create a project using this example:

```bash
npx thirdweb create --template nft-gated-website
```

- Create an [Edition Drop](https://thirdweb.com/contracts/new/pre-built/drop/edition-drop) contract using the dashboard.
- Update the information in the [yourDetails.js](./const/yourDetails.js) file to use your contract address and auth domain name.
- Add your wallet's private key as an environment variable in a `.env.local` file called `PRIVATE_KEY`:

```text title=".env.local"
THIRDWEB_AUTH_PRIVATE_KEY=your-wallet-private-key
```

## How It Works

Using [Auth](https://portal.thirdweb.com/auth), we can verify a user's identity on the server-side, by asking them to sign a message and verify they own the wallet they claim to be, and validating the signature.

When we verified the user's identity on the server-side, we check their wallet to see if they have an NFT from our collection. We can then serve different content and restrict what pages they can access based on their NFT balance.

```jsx
function MyApp({ Component, pageProps }) {
  return (
    <ThirdwebProvider
      desiredChainId={activeChainId}
      authConfig={{
        domain: domainName, // This can be any value. e.g. "example.org"
        authUrl: "/api/auth",
      }}
    >
      <Component {...pageProps} />
    </ThirdwebProvider>
  );
}

export default MyApp;
```

Next, we need to create a configuration file that contains our wallet's private key (used to generate messages for users to sign) and our site's domain name:

This file is called `auth.config.js` and is at the root of the project.

```jsx
import { ThirdwebAuth } from "@thirdweb-dev/auth/next";
import { PrivateKeyWallet } from "@thirdweb-dev/auth/evm";
import { domainName } from "./const/yourDetails";

export const { ThirdwebAuthHandler, getUser } = ThirdwebAuth({
  domain: domainName,
  wallet: new PrivateKeyWallet(process.env.THIRDWEB_AUTH_PRIVATE_KEY || ""),
});
```

Finally, we have a [catch-all API route](https://nextjs.org/docs/api-routes/dynamic-api-routes#catch-all-api-routes) called `pages/api/auth/[...thirdweb].js`, which exports the `ThirdwebAuthHandler` to manage all of the required auth endpoints like `login` and `logout`.

```jsx
import { ThirdwebAuthHandler } from "../../../auth.config";

export default ThirdwebAuthHandler();
```

## Restricting Access

To begin with, the user will reach the website with no authentication.

When they try to access the restricted page (the `/` route), we use [getServerSideProps](https://nextjs.org/docs/basic-features/data-fetching/get-server-side-props) to check two things:

1. If the user is currently authenticated (using `getUser`).
2. If the user's wallet balance is greater than `0` of the NFTs in our NFT collection.

If either of these checks is `false`, we redirect the user to the `/login` page before they are allowed to access the restricted page.

Let's break that down into steps:

### Setting Up the Auth SDK

Inside the [\_app.jsx](./pages/_app.jsx) file, we configure the Auth SDK in the `ThirdwebProvider` component that wraps our application, allowing us to use the hooks of the SDK throughout our application:

### Checking For Authentication Token

First, we check if this user has already been authenticated.

If this is the first time the user has visited the website, they will not have an `access_token` cookie.

```js
// This gets called on every request
export async function getServerSideProps(context) {
  const user = await getUser(context.req);

  if (!user) {
    return {
      redirect: {
        destination: "/login",
        permanent: false,
      },
    };
  }

  // ...
}
```

If the user is not authenticated, then we don't check the user's wallet balance; we just immediately redirect them to the `/login` page.

If there _is_ a detected user from `getUser`, we can then check their balance.

### Checking Wallet Balance

Now we're ready to check the user's wallet balance.

To do this, we have created a utility function called [checkBalance](./util/checkBalance.js) that we can use to check the user's balance for a given NFT.

```js
import { contractAddress } from "../const/yourDetails";

export default async function checkBalance(sdk, address) {
  const editionDrop = await sdk.getContract(
    contractAddress, // replace this with your contract address
    "edition-drop"
  );

  const balance = await editionDrop.balanceOf(address, 0);

  // gt = greater than
  return balance.gt(0);
}
```

This function returns true or false that we can store in a variable:

```js
const hasNft = await checkBalance(sdk, address);
```

Here's our final check, if the user has a `balance` of `0`, then we redirect them to the `/login` page.

```js
// If they don't have an NFT, redirect them to the login page
if (!hasNft) {
  return {
    redirect: {
      destination: "/login",
      permanent: false,
    },
  };
}
```

If the user gets past these checks, then we allow them to view the restricted page.

```js
// Finally, return the props
return {
  props: {},
};
```

## Signing In

We've now successfully restricted access to our home page, now let's explore the `/login` page.

We use the `ConnectWallet` component to handle the connection to the user's wallet, signing in, and signing out.

```js
<ConnectWallet />
```

## Join our Discord!

For any questions, suggestions, join our discord at [https://discord.gg/thirdweb](https://discord.gg/thirdweb).
