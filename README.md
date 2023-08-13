# The Pay4Best Wallet's User Guide

The Pay4Best wallet is web-based wallet deployed at [pay4.best](https://pay4.best). Instead of storing the private key in [localStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage), it always derives the private key from a signature signed by a Web3 wallet, such as MetaMask, TrustWallet, Rabby, imToken, TokenPocket, etc. So, you do not need to remember another set of bip39 mnemonics, or worry about leaking the on-disk localStorage.

However, you must NEVER sign a message containing **"I hereby grant this website the permission to access my 💰pay4.best wallet💰"** on other websites. If you signed, they can get the private key used by the Pay4Best wallet.

Pay4Best Wallet uses different methods to interact with DApp on PC browsers and on mobile devices, because the mobile wallets restrict the permissions of iframes while PC browsers do not.

## Working on PC browsers

A DApp interacts with the Pay4Best wallet through [cross-window messaging](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage). The DApp's website should contain a iframe of https://pay4.best, like this:

```html
<!-- When using the mainnet: -->

<iframe src="https://pay4.best" id="pay4bestFrame"></iframe>

<!-- When using the testnet: -->

<iframe src="https://pay4.best/?testnet=true" id="pay4bestFrame"></iframe>

```

The Pay4Best frame talks to its parent window through [MessageChannel](https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel). The communication protocol is described as below.

### Get the Address and the Public Key

To get the cash address of the account managed by the Pay4Best frame, you should post a message containing `Pay4BestWalletReq: {GetAddr: true}` field. And the result will be posted back in the event's `data` field.

```javascript
   const channel = new MessageChannel();
   pay4bestFrame.contentWindow.postMessage({
       Pay4BestWalletReq: {
           GetAddr: true,
       }
   }, '*', [channel.port1]);
   channel.port2.onmessage = function(event) {
       if (event.data === 'wallet-not-init') {
           // the pay4best wallet is not loaded yet
       } else {
           // event.data is the Cash Address
       }
   }
```

To get the public key of the account managed by the Pay4Best frame, you should post a message containing `Pay4BestWalletReq: {GetPublicKey: true}` field. And the result will be posted back in the event's `data` field.

Just replace `GetAddr` with `GetPublicKey` in the above example and `event.data` will be the public key.

### Get a signed transaction

To make the Pay4Best frame sign a transaction, you should post a message containing `Pay4BestWalletReq: {UnsignedTx: ...}` field:


```javascript
  const channel = new MessageChannel();
  pay4bestFrame.contentWindow.postMessage({
      Pay4BestWalletReq: {
          UnsignedTx: {
              transaction: <the first argument of signUnsignedTransaction>,
              sourceOutputs: <the second argument of signUnsignedTransaction>
          }
      }
  }, '*', [channel.port1]);
  channel.port2.onmessage = function(event) { // to parse the first response
      if (!event.data.ok) {
          // the browser blocked 'window.open', ask the user to unblock pop-out windows
          // and retry window.open:
          var url = "https://pay4.best/?origin="+encodeURI(location.origin)+"&req="+event.data.reqID;
          window.open(url);
      }
      channel.port2.onmessage = function (event) { // to parse the second response
          if (event.data.refused) {
              // the user refused to sign
          } else {
              //event.data.signedTx is what you need
          }
      }
  }
```

The `UnsignedTx` field provides the arguments used to call [`signUnsignedTransaction`](https://github.com/pay4best/pay4best.github.io/blob/main/utils/index.ts).

The Pay4Best frame will post back two responses in sequence.

The first response has a `ok` field and a `reqID` field. The `ok` field shows whether the window for signing confirmation has successfully popped out. If it's not, you should use the `reqID` to assemble a URL and use it to call `window.open` again, after asking the user to unblock all the pop-out windows from this DApp.

The second response has a `refused` field and a `signedTx` field. The `refused` field shows whether the user refused to sign the transaction. If the user did not refuse, the `signedTx` is the return value from `signUnsignedTransaction`.

### Get a signature for CheckSig

Cashscript supports a data type named `sig`, which is a signature used by the `CheckSig` function. Generating such a signature is different from signing a transaction. So, to make the Pay4Best frame generate such a signature, you should post a message containing `Pay4BestWalletReq: {UnsignedTx: ...}` field like this:

```
  const channel = new MessageChannel();
  pay4bestFrame.contentWindow.postMessage({
      Pay4BestWalletReq: {
          UnsignedTx: {
              transaction: <the first argument of signTransactionForArg>,
              sourceOutputs: <the second argument of signTransactionForArg>,
              inputIndex: <the third argument of signTransactionForArg>,
              bytecode: <the fourth argument of signTransactionForArg>
          }
      }
  }, '*', [channel.port1]);
```

The `UnsignedTx` field provides the arguments used to call [`signTransactionForArg`](https://github.com/pay4best/pay4best.github.io/blob/main/utils/index.ts). Please note the whole transaction is needed to generate the signature that is used in the unlocking bytecode of just one input.

The Pay4Best frame will post back two responses in sequence.

The first response has a `ok` field and a `reqID` field. The `ok` field shows whether the window for generating signature has successfully popped out. If it's not, you should use the `reqID` to assemble a URL and use it to call `window.open` again, after asking the user to unblock all the pop-out windows from this DApp.

The second response has a `refused` field and a `signedTx` field. The `refused` field shows whether the user refused to generate the signature. If the user did not refuse, the `signedTx` is the return value from `signTransactionForArg`.

## Working on mobile devices

To work on mobile devices, the Pay4Best's page runs in a mobile wallet (MetaMask, TrustWallet, imToken, TokenPocket, etc) and the DApp's page runs in a normal browser (Chrome, Safari, Firefox, etc). The DApp calls Pay4Best through [DeepLinks](https://www.adjust.com/glossary/deep-linking/) to get addresses and to broadcast transactions.

To broadcast a transaction, you call the URL `https://pay4.best/?broadcasttx=<base64_presentation_of_tx>`. The `broadcasttx` parameter is calculated using the following code:

```javascript
import { decode, encode } from "algo-msgpack-with-bigint";

function pack(transaction, sourceOutputs) {
    var unsignedTx = {transaction:transaction, sourceOutputs: sourceOutputs};
    return base64EncodeURL(encode(tx))
}

function base64EncodeURL(byteArray: Uint8Array) {
  return btoa(Array.from(new Uint8Array(byteArray)).map(val => {
    return String.fromCharCode(val);
  }).join('')).replace(/\+/g, '-').replace(/\//g, '_').replace(/\=/g, '');
}
```

The `transaction` and `sourceOutputs` are the first and second argumets of [`signUnsignedTransaction`](https://github.com/pay4best/pay4best.github.io/blob/main/utils/index.ts#L106).

To get the EVM address and the derived cash address of Pay4Best, you call the URL `https://pay4.best/?generateUrl=<origin>`, where `origin` is your original DApp page which needs to know the addresses.

Pay4Best will generate a URL string which points to the original DApp page with two additional parameters: `wallet`, which is the cash address derived by Pay4Best, and `evmAddress`, which is the EVM address used by Pay4Best.

Currently, Pay4Best does not support generate signatures for CheckSig.

