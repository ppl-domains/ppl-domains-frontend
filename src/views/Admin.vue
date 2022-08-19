<template>
<div class="container">
  <h1 class="text-center">Admin</h1>
  
  <div class="row mt-3 text-center">
    <div class="col-md-8 offset-md-2">

      <!-- Minter: togglePaused -->
      <div v-if="isUserMinterAdmin">
        <h3 v-if="getMinterPaused">Unpause the minting contract</h3>
        <h3 v-if="!getMinterPaused">Pause the minting contract</h3>

        <button 
          v-if="isActivated" 
          class="btn btn-primary btn-lg mt-3" 
          @click="togglePaused" 
          :disabled="waiting"
        >
          <span v-if="waiting" class="spinner-border spinner-border-sm mx-1" role="status" aria-hidden="true"></span>
          <span v-if="getMinterPaused">Unpause</span>
          <span v-if="!getMinterPaused">Pause</span>
        </button>
      </div>
      <!-- END Minter: togglePaused -->

      <hr />

    </div>
  </div>
</div>
</template>

<script lang="ts">
import { ethers } from 'ethers';
import { useEthers } from 'vue-dapp';
import { mapActions, mapGetters } from 'vuex';
import { useToast, TYPE } from "vue-toastification";
import WaitingToast from "../components/toasts/WaitingToast.vue";
import MinterAbi from "../abi/Minter.json";

export default {
  name: "Admin",

  data() {
    return {
      waiting: false, // waiting for TX to complete
    }
  },

  computed: {
    ...mapGetters("network", ["getBlockExplorerBaseUrl"]),
    ...mapGetters("user", ["isUserMinterAdmin", "isUserTldAdmin", "isUserRoyaltyFeeUpdater"]),
    ...mapGetters("tld", ["getMinterAddress", "getTldContract", "getMinterPaused"]),

  },

  methods: {
    ...mapActions("tld", ["fetchMinterContractData"]),

    async togglePaused() {
      this.waiting = true;

      // minter contract (with signer)
      const minterIntfc = new ethers.utils.Interface(MinterAbi);
      const minterContractSigner = new ethers.Contract(this.getMinterAddress, minterIntfc, this.signer);

      try {
        const tx = await minterContractSigner.togglePaused();

        const toastWait = this.toast(
          {
            component: WaitingToast,
            props: {
              text: "Please wait for your transaction to confirm. Click on this notification to see transaction in the block explorer."
            }
          },
          {
            type: TYPE.INFO,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          }
        );

        const receipt = await tx.wait();

        if (receipt.status === 1) {
          this.toast.dismiss(toastWait);
          this.toast("You have un/paused the minter contract!", {
            type: TYPE.SUCCESS,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          });
          this.fetchMinterContractData();
          this.waiting = false;
        } else {
          this.toast.dismiss(toastWait);
          this.toast("Transaction has failed.", {
            type: TYPE.ERROR,
            onClick: () => window.open(this.getBlockExplorerBaseUrl+"/tx/"+tx.hash, '_blank').focus()
          });
          console.log(receipt);
          this.waiting = false;
        }

      } catch (e) {
        console.log(e)
        this.waiting = false;
        this.toast(e.message, {type: TYPE.ERROR});
      }

      this.waiting = false;
    },

  },

  setup() {
    const { address, isActivated, signer } = useEthers();
    const toast = useToast();

    return { address, isActivated, signer, toast }
  }
}
</script>

<style scoped>
p {
  font-size: 1.1em;
}

h3 {
  margin-top: 35px;
}
</style>