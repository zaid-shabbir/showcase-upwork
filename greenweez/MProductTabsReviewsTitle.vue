<template>
  <div class="m-product-tabs-reviews-title">
    <ASvgIcon path="outline/star" class="-sm mr-2" />
    <span class="mr-2"> {{ $t('pages.products.tabs.reviews') }} </span>
    <MRatingStars
      :total-reviews="totalReviews"
      :review-rating="reviewRating"
      class="-secondary"
      with-opinion-text
      with-text
      spaced
    >
    </MRatingStars>
  </div>
</template>

<script>
import { defineComponent } from '@nuxtjs/composition-api'
import { integer, number } from 'vue-types'
export default defineComponent({
  props: {
    totalReviews: integer().def(0),
    reviewRating: number().def(0)
  }
})
</script>

<style lang="postcss">
.m-product-tabs-reviews-title {
  @apply flex items-center;
}
</style>
