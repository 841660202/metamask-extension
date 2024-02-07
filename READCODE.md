## 为什么 window.ethereum.request 能够调用起 metamask 插件

<a href="https://docs.metamask.io/wallet/reference/provider-api/#:~:text=MetaMask%20injects%20the%20provider%20API%20into%20websites%20visited%20by%20its%20users%20using%20the%20window.ethereum%20provider%20object." target="_blank" >MetaMask injects the provider API into websites visited by its users using the window.ethereum provider object. You can use the provider properties, methods, and events in your dapp.</a>
<a href="https://docs.metamask.io/wallet/concepts/apis/#ethereum-provider-api" target="_blank" >见</a>

## Handling Multiple Injected Extensions

<a href="https://docs.cloud.coinbase.com/wallet-sdk/docs/injected-provider-guidance" target="_blank" >见</a>

在使用 MetaMask 或其他以太坊钱包扩展时，可能会遇到多个扩展注入`ethereum`对象到同一个页面的情况。这可能会导致冲突，因为你的 DApp（去中心化应用）不知道应该与哪个扩展进行交互。为了解决这个问题，你需要编写代码来正确地检测和选择你想要交互的提供者（provider）。

下面是一个示例代码，展示了如何检测并选择一个注入的提供者：

```javascript
let provider;

if (window.ethereum) {
  provider = window.ethereum;
} else if (window.web3) {
  provider = window.web3.currentProvider;
} else {
  console.log('No Ethereum browser extension detected.');
}

// 现在你可以使用provider来调用API方法，例如：
// provider.request({ method: 'eth_accounts' })...
```

在上面的代码中，我们首先检查`window.ethereum`对象是否存在，如果存在，我们就使用它作为我们的提供者。如果不存在，我们再检查`window.web3`对象，这是一些旧的以太坊浏览器扩展（例如早期版本的 MetaMask 或 Mist 浏览器）注入的对象。如果两者都不存在，我们就记录一条消息说没有检测到以太坊浏览器扩展。

如果你使用的是诸如`web3-react`或`wagmi`这样的第三方库来管理与区块链的连接，你通常会使用一个特殊的连接器（connector），它被设计为与注入的提供者一起工作。例如，在`web3-react`中，你可能会这样使用`InjectedConnector`：

```javascript
import { InjectedConnector } from '@web3-react/injected-connector';

const injected = new InjectedConnector({
  supportedChainIds: [1, 3, 4, 5, 42],
});

// 然后你可以在你的DApp中使用这个连接器
```

在`wagmi`中，你可能会这样使用`injected`配置：

```javascript
import {
  chain,
  configureChains,
  createClient,
  WagmiConfig,
  defaultChains,
} from 'wagmi';
import { InjectedConnector } from 'wagmi/connectors/injected';

// 配置链和RPC提供者
const { chains, provider } = configureChains(defaultChains, [
  /* ... */
]);

// 创建连接器
const connectors = () => {
  return [new InjectedConnector({ chains })];
};

const client = createClient({
  autoConnect: true,
  connectors,
  provider,
});

// 使用WagmiConfig在你的React组件中包装你的应用
```

在这些例子中，`InjectedConnector`或`Injected`是一个抽象，它代表任何注入到页面中的以太坊提供者。通过使用这些库提供的连接器，你可以更容易地在你的 DApp 中处理多个可能被注入的提供者。这些库还处理了许多常见的用例和边缘情况，使得与区块链交互更加简单和稳定。

## metamask 向 website 注入 window. ethereum, 是使用 chrome.scripting.registerContentScripts API 吗？

截至我知识更新的时间点（2023 年 4 月），MetaMask 使用的注入机制并不是直接通过 `chrome.scripting.registerContentScripts` API 实现的。`chrome.scripting.registerContentScripts` API 允许扩展动态地注册内容脚本，这是 Chrome 扩展新的 Manifest V3 (MV3) API 的一部分。

MetaMask，作为一个以太坊钱包和浏览器扩展，会将 `window.ethereum` 对象注入到网页中，使得网页可以与以太坊区块链进行交互。在 Manifest V2 (MV2) 的时代，这通常是通过在扩展的 `manifest.json` 文件中声明内容脚本来实现的，内容脚本会在指定的页面上运行，并注入全局变量。

例如，MetaMask 的 `manifest.json` 可能包含了类似以下的内容脚本声明：

```json
"content_scripts": [
  {
    "matches": ["<all_urls>"],
    "js": ["inpage.js"],
    "run_at": "document_start"
  }
]
```

在这个例子中，`inpage.js` 就是负责注入 `window.ethereum` 对象的脚本。

然而，随着 Chrome 的更新和 Manifest V3 的推出，扩展的开发方式有所变化。MV3 强调了更多的安全性和隐私保护，以及更高效的资源使用。在 MV3 中，背景页面被服务工作线程（Service Workers）取代，而 `chrome.scripting.registerContentScripts` API 允许动态注册和注销内容脚本。

如果 MetaMask 移植到了 MV3，那么它可能会使用 `chrome.scripting.registerContentScripts` 或者其他 MV3 提供的 API 来实现类似的功能。但请注意，由于我的知识截止日期是 2023 年 4 月，我无法提供更新的信息。要了解最新的实现细节，建议查看 MetaMask 的最新官方文档或其在 GitHub 上的源代码。

## 不确定是不是

```js
const registerInPageContentScript = async () => {
  try {
    await chrome.scripting.registerContentScripts([
      {
        id: 'inpage',
        matches: ['file://*/*', 'http://*/*', 'https://*/*'],
        js: ['inpage.js'],
        runAt: 'document_start',
        world: 'MAIN',
      },
    ]);
  } catch (err) {
    /**
     * An error occurs when app-init.js is reloaded. Attempts to avoid the duplicate script error:
     * 1. registeringContentScripts inside runtime.onInstalled - This caused a race condition
     *    in which the provider might not be loaded in time.
     * 2. await chrome.scripting.getRegisteredContentScripts() to check for an existing
     *    inpage script before registering - The provider is not loaded on time.
     */
    console.warn(`Dropped attempt to register inpage content script. ${err}`);
  }
};
```

<img src="http://t-blog-images.aijs.top/img/202402061802508.webp" />

<a href="https://github.com/MetaMask/providers#:~:text=is%20used%20to%20instantiate%20the%20object%20injected%20by%20MetaMask%20into%20web%20pages%20as%20window.ethereum." target="_blank" > used to instantiate the object injected by MetaMask into web pages as window.ethereum.</a>

## metamask/providers

<a href="https://github.com/MetaMask/providers/blob/126a8c8/src/initializeInpageProvider.ts" target="_blank" >见</a>

<a href="https://github.com/MetaMask/providers/blob/126a8c868785eb088511769cd532a72072662369/src/initializeInpageProvider.ts#L32-L102" target="_blank" >见</a>

<a href="https://github.com/MetaMask/providers/blob/126a8c868785eb088511769cd532a72072662369/src/initializeInpageProvider.ts#L100" target="_blank" >最终注入代码</a>

## eip-1193

<a href="https://eips.ethereum.org/EIPS/eip-1193" target="_blank" >https://eips.ethereum.org/EIPS/eip-1193</a>

## dapp 交易时候需要钱包签名，私钥是不离开钱包的，默认钱包是安全的？

在去中心化应用（DApp）中进行交易时，确实需要钱包进行签名操作。签名过程是区块链交互的关键安全特性，用来证明交易是由资产的合法拥有者发起的。私钥用于生成这些签名，但它们从不应该离开用户的钱包。以下是关于这一过程的一些要点：

1. **私钥的安全性**：私钥是用户控制其区块链地址和资产的密钥。钱包软件会安全地存储私钥，并确保它不会在签名过程中泄露。通常，私钥存储在用户的设备上，而不是在钱包的服务器或任何远程位置。

2. **签名过程**：当用户发起交易时，DApp 会创建交易的详细信息，并请求钱包对其进行签名。钱包应用会使用存储在本地的私钥对交易信息进行签名，然后只将签名的交易（不包括私钥）返回给 DApp。之后，DApp 可以将这个已签名的交易发送到区块链网络。

3. **钱包安全性**：尽管默认情况下，流行的钱包如 MetaMask 被设计为安全地管理私钥，但这并不意味着它们是绝对安全无懈可击的。钱包的安全性取决于多个因素，包括但不限于：

   - 钱包软件的代码质量和安全性。
   - 用户设备的安全性，包括操作系统和其他软件。
   - 用户的安全习惯，比如定期更新软件，不点击可疑链接，不泄露个人密钥等。

4. **硬件钱包**：为了提高安全性，许多用户选择使用硬件钱包（如 Ledger 或 Trezor）。硬件钱包将私钥存储在一个安全的硬件设备中，该设备设计成即使连接到已经感染恶意软件的计算机上，私钥也不会被泄露。

5. **智能合约和权限**：一些 DApp 使用智能合约来处理交易，用户可能需要授权这些合约以代表他们执行特定操作。这种授权本质上是给予合约某种程度的访问权，用户应该只授权给那些他们信任的合约。

6. **用户警惕性**：用户需要对任何请求签名的行为保持警惕。恶意的网站或 DApp 可能会试图诱导用户签署授权交易或其他可能损害其资产的签名。

总之，尽管默认情况下许多钱包被设计为安全的，用户仍然需要采取谨慎的安全措施来保护自己的资产，包括使用可信的钱包软件、保持软件更新、使用硬件钱包来提高安全性，以及对任何签名请求保持警惕。
