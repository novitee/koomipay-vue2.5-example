<script>
import { CheckoutContainer, createClient } from "@koomipay/vue2";
import _ from "lodash";

export default {
  name: "KoomipayVue2Sample",
  data() {
    return {
      koomipayClient: null,
      loading: false,
      amount: {
        currency: "SGD",
        price: 1
      },
      countryCode: "SG",
      selectedPaymentMethod: null,
      paymentData: null,
      isValid: true,
      availablePaymentMethods: [],
      submitting: false
    };
  },
  components: {
    CheckoutContainer
  },
  async mounted() {
    this.loading = true;
    const apiKey =
      localStorage.getItem("api-key") ||
      "live.iMOgnA-v+Tg6lcINEXq0UAUOXkVosfS0RYwxCqw5oQXFHUp/jdFRTF5PZEI.ov-AJ+.QGjCXSF6nLes_v1H2yELAsm2vZ/RfXjFWe9O7WpjRt9923f-_jcPmNA4yADDnSw-v5P604f-s3JTIlZ+VX5BWY1KFscRgQ";
    this.koomipayClient = createClient({ apiKey });
    const paymentMethods = await this.koomipayClient.getPaymentMethods({
      amount: this.amount,
      countryCode: this.countryCode
    });
    this.loading = false;
    this.availablePaymentMethods = paymentMethods.filter(
      (method) => method.type !== "applepay" || window.ApplePaySession
    );
  },
  methods: {
    async handleSubmit() {
      if (!this.isValid) {
        return;
      }
      this.submitting = true;
      try {
        const { browserInfo, ...params } = this.paymentData;
        const response = await this.koomipayClient.instantCheckout({
          orderID: `#test01`,
          paymentMethod: params,
          browserInfo,
          amount: this.amount,
          countryCode: this.countryCode
        });

        this.submitting = false;
        if (
          _.get(response, "data.success") &&
          _.get(response, "data.data.resultCode") === "Authorised"
        ) {
          const redirectUrl =
            _.get(response, "data.data.redirect3ds") ||
            _.get(response, "data.data.redirectPOS");
          if (redirectUrl) {
            window.open(redirectUrl, "_blank");
          } else {
            alert("Successfully");
            return;
          }
        } else {
          alert(
            _.get(response, "data.message") ||
              _.get(response, "data.data.resultCode")
          );
        }
      } catch (error) {
        console.log(error);
      }
    },
    async handleOnAuthorise(state) {
      if (this.submitting) {
        return false;
      }
      const data = {
        browserInfo: this.paymentData.browserInfo || {},
        type: this.selectedPaymentMethod.type
      };

      if (this.selectedPaymentMethod.type === "applepay") {
        data.applePayToken = btoa(
          JSON.stringify(state.payment.token.paymentData || null)
        );
      } else if (this.selectedPaymentMethod.type === "googlepay") {
        data.googlePayToken = state.paymentMethodData.tokenizationData.token;
      }

      this.paymentData = data;
      this.handleSubmit();
      return true;
    },
    handleValid(isValid) {
      this.isValid = isValid;
    },
    handleChange(paymentData) {
      this.paymentData = paymentData;
    },
    handleSelectPayment(e) {
      const foundSelectedMethod = this.availablePaymentMethods.find(
        (method) => method.type === e.target.value
      );
      if (foundSelectedMethod) {
        this.selectedPaymentMethod = foundSelectedMethod;
      }
    }
  }
};
</script>

<template>
  <div id="app">
    <div v-if="loading">Loading methods...</div>
    <div v-for="method in availablePaymentMethods" :key="method.scheme">
      <input
        type="radio"
        @change="handleSelectPayment"
        :checked="
          selectedPaymentMethod && method.type === selectedPaymentMethod.type
        "
        :value="method.type"
      />
      {{ method.name }}
    </div>
    <main>
      <div>
        <CheckoutContainer
          :client="koomipayClient"
          :amount="amount"
          :country-code="countryCode"
          :payment-method="selectedPaymentMethod"
          @onAuthorise="handleOnAuthorise"
          @onChange="handleChange"
          @onValid="handleValid"
        />
        <button
          :style="`border: 1px solid black; padding: 10px 30px; opacity: ${
            isValid ? '1' : '0.4'
          };`"
          v-if="
            selectedPaymentMethod && selectedPaymentMethod.type === 'scheme'
          "
          v-on:click="handleSubmit"
        >
          <span v-if="submitting">Paying...</span>
          <span v-else>Pay</span>
        </button>
      </div>
    </main>
  </div>
</template>
