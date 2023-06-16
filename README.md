# The Pay4Best Wallet's User Guide

The Pay4Best wallet is web-based wallet deployed at [pay4.best](https://pay4.best). Instead of storing the private key in [localStorage](https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage), it always derives the private key from a signature signed by a Web3 wallet, such as MetaMask, TrustWallet, etc. So, you do not need to remember another set of bip39 mnemonics, or worry about the on-disk localStorage.

However, you must NEVER sign a message containing **"I hereby grant this website the permission to access my 💰pay4.best wallet💰"** on other websites. If you signed, they can get the private key used by the Pay4Best wallet.

A DApp interacts with the Pay4Best wallet through [cross-window messaging](https://developer.mozilla.org/en-US/docs/Web/API/Window/postMessage). The DApp's website should contain a iframe of https://pay4.best, like this:

```html
<!-- When using the mainnet: -->

<iframe src="https://pay4.best" id="pay4bestFrame"></iframe>

<!-- When using the testnet: -->

<iframe src="https://pay4.best/?testnet=true" id="pay4bestFrame"></iframe>

```

The Pay4Best frame talks to its parent window through [MessageChannel](https://developer.mozilla.org/en-US/docs/Web/API/MessageChannel). The communication protocol is described as below.

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

To make the Pay4Best frame sign a transaction, you should post a message containing `Pay4BestWalletReq: {UnsignedTx: ...}` field:


```javascript
  const channel = new MessageChannel();
  pay4bestFrame.contentWindow.postMessage({
      Pay4BestWalletReq: {
          UnsignedTx: {
              transaction: <the first argument of signUnsignedTransaction>
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

The `UnsignedTx` field provides the arguments used to call [`signUnsignedTransaction`](https://github.com/pay4best/pay4best.github.io/blob/main/utils/index.ts#L79).

The Pay4Best frame will post back two responses in sequence.

The first response has a `ok` field and a `reqID` field. The `ok` field shows whether the window for signing confirmation has successfully popped out. If it's not, you should use the `reqID` to assemble a URL and use it to call `window.open` again, after asking the user to unblock all the pop-out windows from this DApp.

The second response has a `refused` field and a `signedTx` field. The `refused` field shows whether the user refused to sign the transaction. If the user did not refuse, the `signedTx` is the return value from `signUnsignedTransaction`.
