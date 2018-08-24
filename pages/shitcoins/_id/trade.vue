<template lang="pug">
  .content
    .wrapper
      shitcoin-header(:shitcoin="base_shitcoin" :price="book.price" active="trade")
      p(v-if="$store.state.quote_id == base_shitcoin.id") Please select another shitcoin to trade.
      .container(v-else)
        .item.order_book
          .side.bids
            h2 Bids
            table
              tbody
                tr
                  th Rate
                  th Quantity
                tr.bid(v-for="bid in book.bids") 
                  td {{ bid.rate }}
                  td {{ bid.quantity }}
          .side.bids
            h2 Asks
            table.side
              tbody
                tr
                  th Rate
                  th Quantity
                tr.bid(v-for="ask in book.asks") 
                  td {{ ask.rate }}
                  td {{ ask.quantity }}
        .item.new_order
          h2 New Order
          form(@submit.prevent="submitOrder")
            table.form
              tbody
                tr
                  td 
                  td You have {{ base_available }} {{ base_shitcoin.symbol }} available
                tr
                  td
                  td
                    select(v-model="newOrder.kind")
                      option(value="limit") Limit
                      option(value="market") Market
                tr
                  td
                    label Quantity
                  td 
                    input(v-model="newOrder.quantity")
                tr
                  td
                    label Price
                  td
                    input(v-model="newOrder.rate")
                tr
                  td
                  td
                    button(@click="newOrder.side = 'buy'" :disabled="!canBuy") BUY
                    button(@click="newOrder.side = 'sell'" :disabled="!canSell") SELL
                    p(v-if="!$store.state.user") Log in to trade
        .item.my_oders
          h2 Open Orders
          table.orders
            thead
              tr
                th Created at
                th Type
                th Quantity
                th Rate
                th Filled
                th 
            tbody
              tr.order(v-for="order in orders")
                td {{ order.created_at }}
                td {{ order.side }} at {{ order.kind }}
                td {{ order.quantity }}
                td {{ order.rate }}
                td {{ order.quantity_filled }}
                td
                  button(@click="cancelOrder(order.id)") Cancel
</template>

<script lang="coffee">
module.exports =
  computed:
    base_available: ->
      if @$store.state.balances[@base_shitcoin.id]
        @$store.state.balances[@base_shitcoin.id].available
      else
        0
    quote_available: ->
      if @$store.state.balances[@$store.state.quote_id]
        @$store.state.balances[@$store.state.quote_id].available
      else
        0
    canBuy: ->
      @$store.state.user && @newOrder.kind == 'limit'
    canSell: ->
      @$store.state.user
  subscriptions:
    OrderbookChannel:
      params: ->
        base_id: @base_shitcoin.id
        quote_id: @$store.state.quote_id
      received: (book) ->
        @book = book
    # TODO: somehow subscribe to order notification globally, even if component is not shown.
    OrdersChannel:
      params: ->
        base_shitcoin_id: @base_shitcoin.id
      received: (order) ->
        if order.cancelled_at
          @orders.delete(order)
        else if order.filled_at
          @orders.delete(order)
        else
          @orders.upsert(order)
  watch:
    '$store.state.user': ->
       @orders = await this.$axios.$get('/orders', params: {base_shitcoin_id: this.base_shitcoin.id})
  asyncData: ({app: {$axios}, params, error, store}) ->
    base_id = params.id
    [base_shitcoin, orders, book] = await Promise.all [
      $axios.$get("/shitcoins/#{base_id}"),
      $axios.$get('/orders', params: {base_shitcoin_id: base_id})
      $axios.$get('/order_book/depth', params: {base: base_id, quote: store.state.quote_id})
    ]

    return
      newOrder:
        base_shitcoin_id: base_shitcoin.id
        rate: null
        quantity: null
        side: 'sell'
        kind: 'limit'
      book: book
      orders: orders
      base_shitcoin: base_shitcoin
   methods:
     cancelOrder: (id) ->
       this.$axios.$delete('/orders/'+id)
     submitOrder: ->
       this.newOrder.quote_shitcoin_id = this.$store.state.quote_id
       this.$axios.$post('/orders', order: this.newOrder)
       

</script>

<style lang="scss">

.order_book .side {
  width: 50%;
  display: inline-block;
  vertical-align: top;
}
</style>