<template>
  <div>
    <h1 v-if="loading">{{loading_text}}...</h1>
    <div class="max-w-sm rounded overflow-hidden shadow-lg bg-gray-500 text-white" v-else>
      <!--<img class="w-full" src="/img/card-top.jpg" alt="Sunset in the mountains">-->
      <div class="px-6 py-4">
        <div class="font-bold text-xl mb-2">My account</div>
        <p class="text-white-700 text-base">
          <strong>Address:</strong><br><span class="text-xs">{{address}}</span>
        </p>
        <p class="text-white-700 text-base">
          <strong>Address balance:</strong><br>
          {{balance}} AE
        </p>

        <p class="text-white-700 text-base">
          <strong>Contract balance:</strong><br>
          {{contract_balance}} Aettos
        </p>
        <div>
          <button class="bg-purple-500 hover:bg-purple-700 text-white font-bold py-2 px-4 rounded" @click="pay_back">
            Pay back
          </button>

          <button class="bg-purple-500 hover:bg-purple-700 text-white font-bold py-2 px-4 rounded" @click="pay_out">
            Pay out
          </button>
        </div>
      </div>
      <!-- <div class="px-6 py-4">
        <span class="inline-block bg-gray-200 rounded-full px-3 py-1 text-sm font-semibold text-gray-700 mr-2">#photography</span>
        <span class="inline-block bg-gray-200 rounded-full px-3 py-1 text-sm font-semibold text-gray-700 mr-2">#travel</span>
        <span class="inline-block bg-gray-200 rounded-full px-3 py-1 text-sm font-semibold text-gray-700">#winter</span>
      </div> -->
    </div>
  </div>
</template>

<script>
  import aeternity from "../utils/aeternity";
  import axios from 'axios'

  const contractSource = `
    /* 3.f - an escrow that is controlled by a third party for a fee. */

    contract PaidEscrow =

      record state = {
        payer : address,
        payee : address,
        arbiter : address,
        fee : int }

      stateful entrypoint init(_payee : address, _arbiter : address, _fee : int) =
        { payer = Call.caller,
          // Chain.spend(Contract.balance,  'amount' ),
          payee = _payee,
          arbiter = _arbiter,
          fee = _fee }

      stateful entrypoint pay_out() =
        if(Call.caller == state.arbiter)
            Chain.spend(state.arbiter, state.fee)
            Chain.spend(state.payee, Contract.balance)

      stateful entrypoint pay_back()  =
        if(Call.caller == state.arbiter)
            Chain.spend(state.arbiter, state.fee)
            Chain.spend(state.payer, Contract.balance)
            
      entrypoint balance() : int =
        Contract.balance
        
      entrypoint am_i_arbiter() : bool =
        Call.caller == state.arbiter`;
  
  let contractAddress = "ct_7GRs3rwG19CWDhcDbwPGKxiFsQzALBqDbSwoM4GWCo81EuJpa";
  
  async function callStatic(func, args) {
    //Create a new contract instance that we can interact with
    const contract = await aeternity.client.getContractInstance(contractSource, {contractAddress});
    
    //Make a call to get data of smart contract func, with specefied arguments
    const calledGet = await contract.call(func, args, {callStatic: true}).catch(e => console.error("Static call error: " + e));
    
    //Make another call to decode the data received in first call
    const decodedGet = await calledGet.decode().catch(e => console.error("Decoding error: " + e));
    console.log("decodedGet: " + decodedGet);

    return decodedGet;
  }

  async function contractCall(func, args, value) {
    const contract = await aeternity.client.getContractInstance(contractSource, {contractAddress});
    
    //Make a call to write smart contract func, with aeon value input
    const calledSet = await contract.call(func, args, {amount: value}).catch(e => console.error("Contract call error: " + e));

    return calledSet;
  }

  export default {
    name: 'Home',
    components: {},
    data() {
      return {
        arbiter: false,
        loading: true,
        loading_text: "Loading",
        address: null,
        balance: null,
        contract_balance: null
      };
    },

    async mounted() {
      this.loading_text = "Initializing client";
      await aeternity.initClient();

      // if (aeternity.isTestnet() && aeternity.balance <= 5) {
      //   this.loading_text = "Topping up"
      //   await axios.post(`https://testnet.faucet.aepps.com/account/${aeternity.address}`, {}, {headers: {'content-type': 'application/x-www-form-urlencoded'}}).catch(console.error);
      // }
      
      this.address = aeternity.address;
      this.balance = aeternity.balance;

      this.loading_text = "Getting contract balance"
      let result = await callStatic('balance', []);
      this.contract_balance = result;

      this.loading_text = "Getting arbiter status"
      result = await callStatic('am_i_arbiter', []);
      console.log("arbiter: ", result);
      //this.arbiter = result;

      this.loading = false;
      this.loading_text = "Loading";
    },

    methods: {
      async pay_back () {
        console.log("pay_back clicked")
        let result = await contractCall('pay_back', [], 0);
        console.log(result);
        await this.get_balance();
      },

      async pay_out () {
        console.log("pay_out clicked")
        let result = await contractCall('pay_out', [], 0);
        console.log(result);
        await this.get_balance();
      },

      async get_balance() {
        this.contract_balance = await callStatic('balance', []);
      }
    }
  };
</script>

<style scoped>

</style>
