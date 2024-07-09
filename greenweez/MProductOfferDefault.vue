<template>
  <div class="m-product-offer-default">
    <div class="__availability-wrapper">
      <div class="bg-gray-400 rounded-full inline-flex p-1 mr-2">
        <ASvgIcon path="gwz/delivery" class="-md" />
      </div>
      <div class="leading-tight">
        <div class="__status">En stock</div>
        <div class="__delivery">
          {{ $t('pages.products.delivery') }} <b>{{ currentOffer.shipping }}</b>
        </div>
      </div>
    </div>
    <div class="__shipping-price">
      <ASvgIcon path="outline/check" class="-sm mr-1 text-primary-600" />
      <b>
        {{ $t('pages.products.shippingPrice') }} :
        {{ displayPrice(currentOffer.shippingPrice) }}
      </b>
      <span class="__link-wrapper">
        &nbsp;(<NuxtLink to="/" class="__link">{{
          $t('pages.products.showTerms')
        }}</NuxtLink
        >)</span
      >
    </div>
    <div class="__estimated-delivery">
      <ASvgIcon path="outline/check" class="-sm mr-1 text-primary-600" />
      {{ $t('pages.products.estimatedDelivery') }} &nbsp;<b>
        {{ currentOffer.delivery }}</b
      >
    </div>
  </div>
</template>

<script>
import { defineComponent } from '@nuxtjs/composition-api'
import { shape, string, integer, number } from 'vue-types'
import displayPrice from '@/composition/useDisplayPrice'
export default defineComponent({
  props: {
    currentOffer: shape({
      id: string(),
      name: string(),
      availability: integer(),
      shipping: string(),
      shippingPrice: number(),
      price: shape({
        currentPrice: number(),
        oldPrice: number(),
        discount_percent: number(),
        ppu: string()
      }),
      delivery: string()
    }).isRequired
  },

  setup() {
    return {
      displayPrice: displayPrice.displayPrice
    }
  }
})
</script>
<style lang="postcss" scoped>
.m-product-offer-default {
  .__availability-wrapper {
    @apply flex items-center bg-gray-300 px-4 py-1 rounded-lg w-[fit-content] mb-4;

    .__status {
      @apply text-primary-700 font-semibold;
    }

    .__delivery {
      @apply text-gray-700;
    }
  }

  .__shipping-price,
  .__estimated-delivery {
    @apply flex items-center text-gray-700;

    .__link-wrapper {
      @apply text-primary-700;

      .__link {
        @apply underline;
      }
    }
  }
}
</style>
