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
import { ChainNames, BridgeChain, Amount, RARIMO_BRIDGE_FEE, NATIVE_TOKEN_WRAP_SLIPPAGE_MULTIPLIER } from '@rarimo/shared'
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
    await swapper.init()
    const op = createCheckoutOperation(EVMOperation, provider)
    const chains = await op.supportedChains()

    const selectedChain = chains.find((i: BridgeChain) => i.name === ChainNames.Goerli)!

    // Initialize the swap operation and set the amount of the source token to swap
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

    const chainTo = swapper.chains.find(
      i => Number(i.id) === Number(opInitParams.chainIdTo),
    )

    const paymentTokens = await op.loadPaymentTokens(selectedChain)
    const selectedToken = paymentTokens?.find(i => i.symbol === sourceToken.symbol)!
    const estimatedPrice:EstimatedPrice = await op.estimatePrice(selectedToken)

    // Get the amount you'll get out of the swap
    const { amountIn, amountOut } = getAmounts(opInitParams, estimatedPrice)

    const swapParams:ExecuteArgs = {
      amountIn,
      from: sourceToken,
      to: destinationToken,
      amountOut,
      chainTo,
      isWrapped: true,
      path: estimatedPrice.path, // What is this path?
    }

    // Run the swap
    await swapper.execute(swapParams)

  } catch (e) {
    console.error(e)
  }
}

// Copied these functions from packages/nft-checkout/src/operations/evm/evm-operation.ts and helpers.js
const getNativeAmountIn = (
  params: CheckoutOperationParams,
  rawPrice: Price | Amount,
): Amount => {
  if (isSameChainOperation(params)) return rawPrice

  const amountWithSlippage = BN.fromBigInt(
    rawPrice.value,
    rawPrice.decimals,
  ).mul(BN.fromRaw(NATIVE_TOKEN_WRAP_SLIPPAGE_MULTIPLIER, rawPrice.decimals))

  return Amount.fromBigInt(amountWithSlippage.value, rawPrice.decimals)
}

const getAmounts = (
  params: CheckoutOperationParams,
  e: EstimatedPrice,
): { amountIn: Amount; amountOut: Amount } => {
  const amountIn = e.from.isNative
    ? getNativeAmountIn(params, e.price)
    : e.price

  const amountOut = getSwapAmount(params)

  return { amountIn, amountOut }
}

import { BN } from '@distributedlab/tools'

const getSwapAmount = (params: CheckoutOperationParams): Amount => {
  const rawPrice = params.price
  if (isSameChainOperation(params)) return rawPrice

  const decimals = rawPrice.decimals
  const numerator = BN.fromBigInt(rawPrice.value, decimals)

  const percentBN = BN.fromRaw(RARIMO_BRIDGE_FEE, decimals).div(
    BN.fromRaw(100, decimals),
  )

  const denominator = BN.fromRaw(1, decimals).sub(percentBN)

  return Amount.fromBigInt(numerator.div(denominator).value, rawPrice.decimals)
}
const isSameChainOperation = (params: CheckoutOperationParams) => {
  return Number(params.chainIdFrom) === Number(params.chainIdTo)
}

</script>
