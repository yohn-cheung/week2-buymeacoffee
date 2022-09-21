<script setup>
import abi from './utils/BuyMeACoffee.json'
import { ethers } from "ethers";
import { ref, watch, onMounted } from 'vue'

// Contract Address & ABI
const contractAddress = "0x76B9278899c4823bF8812Cd8e3CFABFb4aB85696";
const contractABI = abi.abi;

const currentAccount = ref("")
const name = ref("")
const message = ref("")
const memos = ref([])

const notification = ref(false)
const errorMessage = ref('')
const loading = ref(false)

async function isWalletConnected() {
   try {
      const { ethereum } = window;

      const accounts = await ethereum.request({method: 'eth_accounts'})
      console.log("accounts: ", accounts);

      if (accounts.length > 0) {
        const account = accounts[0];
        currentAccount.value = accounts[0];
        console.log("wallet is connected! " + account);
      } else {
        notification.value = true
        errorMessage.value = 'Make sure MetaMask is connected'
      }
    } catch (error) {
      console.log("error: ", error);
    }
}

async function connectWallet() {
  deleteNotification()
  try {
      const {ethereum} = window;

      if (!ethereum) {
        console.log("please install MetaMask");
        notification.value = true
        errorMessage.value = 'please install MetaMask'
      }

      const accounts = await ethereum.request({
        method: 'eth_requestAccounts'
      });

      currentAccount.value = accounts[0]
    } catch (error) {
      console.log(error);
      notification.value = true
      errorMessage.value = 'Something went wrong'
    }
}

async function buyCoffee(value) {
  const ether = value
  deleteNotification()
  try {
    const {ethereum} = window;

    if(name.value === "" || message === "" ) {
      notification.value = true
      errorMessage.value = 'Enter name and/or message'
      return
    }

    if (ethereum) {
      const provider = new ethers.providers.Web3Provider(ethereum, "any");
      const signer = provider.getSigner();
      const buyMeACoffee = new ethers.Contract(
          contractAddress,
          contractABI,
          signer
        );

      console.log("buying coffee..")
      loading.value = true
      const coffeeTxn = await buyMeACoffee.buyCoffee(
        name.value,
        message.value,
        {value: ethers.utils.parseEther(ether)}
      );

      await coffeeTxn.wait();
      console.log("mined ", coffeeTxn.hash);
      console.log("coffee purchased!");

      // Clear the form fields.
      name.value = ""
      message.value = ""
      loading.value = false
      getMemos()
    }
  } catch (error) {
    console.log(error);
    notification.value = true
    errorMessage.value = 'Could NOT buy coffee'

    name.value = ""
    message.value = ""
    loading.value = false
  }
}

async function getMemos() {
  try {
      const { ethereum } = window;
      if (ethereum) {
        const provider = new ethers.providers.Web3Provider(ethereum);
        const signer = provider.getSigner();
        const buyMeACoffee = new ethers.Contract(
          contractAddress,
          contractABI,
          signer
        );
        
        console.log("fetching memos from the blockchain..");
        const fetchMemos = await buyMeACoffee.getMemos();
        console.log("fetched!");
        memos.value = fetchMemos
      } else {
        console.log("Metamask is not connected");
      }
    } catch (error) {
      console.log(error);
    }
}

function deleteNotification() {
  notification.value = false
  errorMessage.value = ''
}

async function getBalance(provider, address) {
  const balanceBigInt = await provider.getBalance(address);
  return ethers.utils.formatEther(balanceBigInt);
}

async function withdraw() {
  try {
    const {ethereum} = window;

    if (ethereum) {
      loading.value = true
      notification.value = true
      const provider = new ethers.providers.Web3Provider(ethereum, "any");
      const signer = provider.getSigner();
      const buyMeACoffee = new ethers.Contract(
          contractAddress,
          contractABI,
          signer
        );

      

      // Check starting balances.
      // console.log("current balance of owner: ", await getBalance(provider, signer.address), "ETH");
      const contractBalance = await getBalance(provider, buyMeACoffee.address);
      console.log("current balance of contract: ", await getBalance(provider, buyMeACoffee.address), "ETH");

      // Withdraw funds if there are funds to withdraw.
      if (contractBalance !== "0.0") {
        console.log("withdrawing funds..")
        errorMessage.value = 'withdrawing funds..'
        const withdrawTxn = await buyMeACoffee.withdrawTips();
        await withdrawTxn.wait();
      } else {
        console.log("no funds to withdraw!");
        errorMessage.value = 'no funds to withdraw!'
      }

      loading.value = false

      // Check ending balance.
      // console.log("current balance of owner: ", await getBalance(provider, signer.address), "ETH");
    }
  } catch (error) {
    console.log(error);
  }
}

onMounted(() => {
  isWalletConnected();
  getMemos();
})
</script>

<template>
  <div class="container">
    <div v-if="notification" class="notification is-danger">
      <button class="delete" @click="deleteNotification"></button>
      {{errorMessage}}
    </div>

    <section class="section">
      <h1 class="title coffeeTitle">Buy me a coffee</h1>
      <div v-if='currentAccount'>
        <div>
            
              <div class="formgroup">
                <label>
                  Name
                </label>
                <br/>
                
                <input
                  class="input"
                  type="text"
                  placeholder="Yohn"
                  required
                  v-model="name"
                  :disabled="loading"
                  />
              </div>
              <br/>
              <div class="formgroup">
                <label>
                  Send Yohn a message
                </label>
                <br/>

                <textarea
                  rows={3}
                  class="textarea" 
                  placeholder="Enjoy your coffee!"
                  required
                  v-model="message"
                  :disabled="loading"
                >
                </textarea>
              </div>
              <div class="mt-4 mb-4">
                <button
                  class="button is-info is-fullwidth"
                  @click="buyCoffee('0.001')"
                  :disabled="loading"
                >
                  Send 1 Coffee for 0.001ETH
                </button>
              </div>

              <div class="mt-4 mb-4">
                <button
                  class="button is-info is-fullwidth"
                  @click="buyCoffee('0.003')"
                  :disabled="loading"
                >
                  Buy Large Coffee for 0.003ETH
                </button>
              </div>

              <div class="mt-4 mb-4">
                <button
                  class="button is-info is-fullwidth"
                  @click="withdraw"
                  :disabled="loading"
                >
                  Withdraw tips
                </button>
              </div>
            
            <progress v-if="loading" class="progress is-medium is-dark mgt-medium" max="100"></progress>
          </div>

       </div>
       <div v-else>
        <button @click="connectWallet"  class="button is-info is-fullwidth"> Connect your wallet </button>
       </div>
    </section>
    <section v-if="currentAccount" class="section">

      <div class="card" v-for="memo, index in memos" :key="index">
        <div class="card-content">
          <div class="media">
            <div class="media-content">
              <p class="title is-4">{{memo.name}} </p>

            </div>
          </div>

          <div class="content">
            {{memo.message}}
            
            <br>
            <time >{{ new Date(memo.timestamp * 1000).toLocaleString('en-GB')}}</time>
          </div>
        </div>
      </div>
    </section>
  </div>
</template>

<style scoped>
@import "https://cdn.jsdelivr.net/npm/bulma@0.9.4/css/bulma.min.css";

.coffeeTitle {
  text-align: center;
}

</style>
