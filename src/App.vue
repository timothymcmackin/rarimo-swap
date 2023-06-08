<template>
  <div>
    <div>
      Source network: {{ sourceChainName }}<br/>
      Source token: {{ sourceTokenSymbol }}
    </div>
    <div style="margin-top:20px">
      Destination network: {{ destinationChainName }}<br/>
      Destination token: {{ destinationTokenSymbol }}
    </div>

    <button style="margin:46px 0" @click="sendSwapTransaction">Swap</button>

    <code>
    </code>
  </div>
</template>

<script setup lang="ts">
import { ChainNames } from '@rarimo/shared'
import { CheckoutOperationParams, createCheckoutOperation, EVMOperation, Price, BridgeChain } from '@rarimo/nft-checkout'
import { createProvider } from '@rarimo/provider'
import { MetamaskProvider } from '@rarimo/providers-evm'

const sourceChainName = ChainNames.Goerli
const destinationChainName = ChainNames.Goerli

const sourceTokenSymbol = "UNI"
const destinationTokenSymbol = "WETH";

const sendSwapTransaction = async () => {
  // Connect to the Metamask wallet in the browser, using the MetamaskProvider interface to limit bundle size.
  const provider = await createProvider(MetamaskProvider)

  // Initialize the object that represents the transaction operation, in this case on EVM.
  const op = createCheckoutOperation(EVMOperation, provider)

  // Get the chains that are supported from that chain type.
  const chains = await op.supportedChains()

  // Select the source and destination chains.
  // This example uses the Goerli chain, but your application can ask the user which chain to use.
  const sourceChain: BridgeChain = chains.find(i => i.name === sourceChainName)!
  const destinationChain: BridgeChain = chains.find(i => i.name === destinationChainName)!

  console.log('Swapping', sourceTokenSymbol, 'for', destinationTokenSymbol)

  // Configure the swap transaction
  const swapParams: CheckoutOperationParams = {
    // Source and destination chains
    chainIdFrom: sourceChain.id,
    chainIdTo: destinationChain.id,
    // Address to send the swapped tokens to
    recipient: provider.address!.toString(),
    // Amount of tokens to receive
    price: Price.fromRaw("0.1", 18, destinationTokenSymbol),
  }

  // Initialize the transaction
  await op.init(swapParams)

  // Identify the tokens to exchange, in this case UNI
  const tokens = await op.loadPaymentTokens(sourceChain!)
  const paymentToken = tokens.find(({ symbol }) => symbol === sourceTokenSymbol)
  const estimatedPrice = await op.estimatePrice(paymentToken!)

  // Run the transaction
  // The `checkout()` method takes the parameters from the operation instance, gets approval from the user's wallet, and calls the Rarimo contract to handle the transaction.
  const txHash = await op.checkout(estimatedPrice)
}
</script>
