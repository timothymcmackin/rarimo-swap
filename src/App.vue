<template>
  <div>
    <div>
      Source network: {{ selectedChainName }}
    </div>
    <div style="margin-top:20px">
      Destination network: {{ destinationChainName }}
    </div>

    <button style="margin:46px 0" @click="test">Swap</button>

    <code>
    </code>
  </div>
</template>

<script setup lang="ts">
import { ChainNames, BridgeChain, Amount } from '@rarimo/shared'
import {createProvider} from '@rarimo/provider'
import {MetamaskProvider} from '@rarimo/providers-evm'
import {
  Price,
  createCheckoutOperation,
  EVMOperation,
  CheckoutOperationParams,
  EstimatedPrice,
} from '@rarimo/nft-checkout'
import { createEVMSwapper, createSwapper, ExecuteArgs } from '@rarimo/swap'
import { Token, newToken, NewTokenOpts } from '@rarimo/bridge'

const selectedChainName = ChainNames.Goerli
const destinationChainName = ChainNames.Goerli

const test = async () => {

  try {

    // Initialize the wallet provider, swapper, and operation
    const provider = await createProvider(MetamaskProvider)
    const swapper = createSwapper(createEVMSwapper, provider)
    const op = createCheckoutOperation(EVMOperation, provider)
    const chains = await op.supportedChains()

    const selectedChain = chains.find((i: BridgeChain) => i.name === ChainNames.Goerli)!

    // Initialize the swap operation
    const opInitParams:CheckoutOperationParams = {
      chainIdFrom: selectedChain.id,
      chainIdTo: selectedChain.id,
      price: Price.fromRaw("0.01", 18, "ETH"),
    }
    await op.init(opInitParams)

    // Source token type, in this case UNI
    const sourceTokenOpts: NewTokenOpts = {
      chain: selectedChain,
      name: "Uniswap",
      symbol: "UNI",
      address: provider.address!,
      decimals: 18,
    }
    const sourceToken = newToken(sourceTokenOpts)

    // Destination token type
    const destinationTokenOpts: NewTokenOpts = {
      chain: selectedChain,
      name: "Wrapped Ether",
      symbol: "WETH",
      address: provider.address!,
      decimals: 18,
    }
    const destinationToken:Token = newToken(destinationTokenOpts)

    const paymentTokens = await op.loadPaymentTokens(selectedChain)
    const selectedToken = paymentTokens?.find(i => i.symbol === sourceToken.symbol)!
    const estimatedPrice:EstimatedPrice = await op.estimatePrice(selectedToken)

    // The amount of the source token to swap
    const sourceAmount:Amount = Amount.fromRaw("0.01", 18)

    const swapParams:ExecuteArgs = {
      amountIn: sourceAmount,
      from: sourceToken,
      to: destinationToken,
      amountOut: estimatedPrice, // Do I have to calculate this myself?
      path: estimatedPrice.path, // Don't know what this path is
    }

    // Run the swap
    await swapper.execute(swapParams)

  } catch (e) {
    console.error(e)
  }
}
</script>
