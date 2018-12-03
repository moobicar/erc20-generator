<template>
    <b-row>
        <b-col lg="10" offset-lg="1" class="mt-4 p-0">
            <b-card bg-variant="light" v-if="!loading" :title="$site.title">
                <p class="card-text">{{ $site.description }}</p>
                <small v-if="!metamask.installed">You need the <a href="https://metamask.io/" target="_blank">MetaMask</a> extension.</small>

                <b-alert variant="success" :show="makingTransaction" class="mt-3">
                    <div>Making transaction.</div>
                    <div v-if="!trxHash">Please wait...</div>
                    <div v-else>
                        <b>Well! Transaction done!</b><br>
                        Transaction id <a :href="trxLink" target="_blank"><span v-html="trxHash"></span></a><br>

                        Retrieving Token.
                        <div v-if="!token.address">Please wait...</div>
                        <div v-else>
                            <b>Your Token</b> <a :href="token.link" target="_blank"><span v-html="token.address"></span></a><br><br>
                            <div class="form-group">
                                <label>ABI</label>
                                <textarea class="form-control"
                                          readonly="readonly" rows="5"
                                          v-model="contracts.token.stringifiedAbi"></textarea>
                            </div>
                            <div>
                                <b>Note: To Verify your Token on Etherscan use:</b>
                                <ul>
                                    <li>Source Code: <a href="https://github.com/vittominacori/erc20-generator/blob/master/dist/BaseToken.dist.sol" target="_blank"><b>BaseToken.dist.sol</b></a></li>
                                    <li>Contract Name: <b>BaseToken</b></li>
                                    <li>Compiler: <b>v0.4.24+commit.e67f0147</b></li>
                                    <li>Optimization: <b>Yes</b></li>
                                    <li>Runs (Optimizer): <b>200</b></li>
                                    <li>Constructor Arguments: <b>your ABI-encoded arguments</b></li>
                                </ul>
                            </div>
                        </div>
                    </div>
                </b-alert>

                <b-form v-on:submit.prevent="generateToken" class="mt-3" v-if="!makingTransaction">
                    <fieldset :disabled="formDisabled">
                        <b-card>
                            <b-row>
                                <b-col lg="4">
                                    <b-form-group
                                            description="Choose a name for your token."
                                            label="Token name *"
                                            label-for="tokenName">
                                        <b-form-input
                                                id="tokenName"
                                                placeholder="Your token name"
                                                v-model.trim="token.name"
                                                :required="true"
                                                maxlength="20">
                                        </b-form-input>
                                    </b-form-group>
                                </b-col>
                                <b-col lg="4">
                                    <b-form-group
                                            description="Choose a symbol for your token (usually 3-4 chars)."
                                            label="Token symbol *"
                                            label-for="tokenSymbol">
                                        <b-form-input
                                                id="tokenSymbol"
                                                placeholder="Your token symbol"
                                                v-model.trim="token.symbol"
                                                :required="true"
                                                maxlength="5">
                                        </b-form-input>
                                    </b-form-group>
                                </b-col>
                                <b-col lg="4">
                                    <b-form-group
                                            description="Insert the decimal precision of your token. If you don't know what to insert, use 18."
                                            label="Token decimals *"
                                            label-for="tokenDecimals">
                                        <b-form-input
                                                id="tokenDecimals"
                                                placeholder="Your token decimals"
                                                v-model.trim="token.decimals"
                                                :required="true"
                                                type="number"
                                                min="0"
                                                max="36"
                                                step="1">
                                        </b-form-input>
                                    </b-form-group>
                                </b-col>
                            </b-row>
                        </b-card>
                        <b-card class="mt-3">
                            <b-row>
                                <b-col lg="4">
                                    <b-form-group
                                            description="Insert the maximum number of tokens available."
                                            label="Token cap *"
                                            label-for="tokenCap">
                                        <b-form-input
                                                id="tokenCap"
                                                placeholder="Your token cap"
                                                v-model.trim="token.cap"
                                                :required="true"
                                                type="number"
                                                min="1"
                                                max="1000000000000"
                                                step="1">
                                        </b-form-input>
                                    </b-form-group>
                                </b-col>
                                <b-col lg="4">
                                    <b-form-group
                                            description="Insert the initial number of tokens available. Will be put in your account."
                                            label="Token initial balance *"
                                            label-for="tokenInitialBalance">
                                        <b-form-input
                                                id="tokenInitialBalance"
                                                placeholder="Your token initial balance"
                                                v-model.trim="token.initialBalance"
                                                :required="true"
                                                type="number"
                                                min="0"
                                                :max="token.cap"
                                                step="1">
                                        </b-form-input>
                                    </b-form-group>
                                </b-col>
                                <b-col lg="4">
                                    <b-form-group
                                            description="Choose your Network."
                                            label="Network *"
                                            label-for="network">
                                        <b-form-select id="network" v-model="currentNetwork" @input="initDapp">
                                            <option v-for="(n, k) in network.list" :value="k">{{ n.name }}</option>
                                        </b-form-select>
                                    </b-form-group>
                                </b-col>
                            </b-row>
                        </b-card>
                        <b-row class="mt-3">
                            <b-col lg="6">
                                <b-button variant="success" size="lg" type="submit">Create Token</b-button>
                            </b-col>
                        </b-row>
                    </fieldset>
                </b-form>
            </b-card>
        </b-col>
    </b-row>
</template>

<script>
  import dapp from '../mixins/dapp';

  export default {
    name: "Generator",
    mixins: [
      dapp
    ],
    data() {
      return {
        loading: true,
        currentNetwork: null,
        trxHash: '',
        makingTransaction: false,
        formDisabled: false,
        token: {}
      };
    },
    mounted() {
      this.currentNetwork = this.getParam('network') || this.network.default;
      this.initDapp();
    },
    methods: {
      async initDapp () {
        this.network.current = this.network.list[this.currentNetwork];
        try {
          await this.initWeb3(this.currentNetwork, true);
          this.initToken();
          this.loading = false;
        } catch (e) {
          alert(e);
          document.location.href = this.$withBase('/');
        }
      },
      async generateToken () {
        if (!this.metamask.installed) {
          alert("To create a Token please install MetaMask!");
          return;
        } else {
          if (this.metamask.netId !== this.network.current.id) {
            alert("Your MetaMask in on the wrong network. Please switch on " + this.network.current.name + " and try again!");
            return;
          }
        }

        const name = this.token.name;
        const symbol = this.token.symbol.toUpperCase();
        const decimals = new this.web3.BigNumber(this.token.decimals);
        const cap = new this.web3.BigNumber(this.token.cap).mul(Math.pow(10, this.token.decimals));
        const initialBalance = new this.web3.BigNumber(this.token.initialBalance).mul(Math.pow(10, this.token.decimals));

        try {
          this.trxHash = '';
          this.formDisabled = true;
          this.makingTransaction = true;

          if (!this.legacy) {
            await this.web3Provider.enable();
          }

          setTimeout(() => {
            this.contracts.token.new(
              name,
              symbol,
              decimals,
              cap,
              initialBalance,
              {
                from: this.web3.eth.coinbase,
                data: this.contracts.token.bytecode,
              }, (e, tokenContract) => {
                if (e) {
                  this.makingTransaction = false;
                  this.formDisabled = false;
                } else {
                  // NOTE: The callback will fire twice!
                  // Once the contract has the transactionHash property
                  // set and once its deployed on an address.
                  if (!tokenContract.address) {
                    this.trxHash = tokenContract.transactionHash;
                    this.trxLink = this.network.current.etherscanLink + "/tx/" + this.trxHash;
                  } else {
                    this.token.address = tokenContract.address;
                    this.token.link = this.network.current.etherscanLink + "/token/" + this.token.address;
                    this.$forceUpdate();
                  }
                }
              }
            );
          }, 500);
        } catch (e) {
          this.makingTransaction = false;
          this.formDisabled = false;
          alert("Some error occurred. Maybe you rejected the transaction or you have MetaMask locked!");
        }
      },
      getParam(param) {
        const vars = {};
        window.location.href.replace(location.hash, '').replace(
          /[?&]+([^=&]+)=?([^&]*)?/gi, // regexp
          function( m, key, value ) { // callback
            vars[key] = value !== undefined ? value : '';
          }
        );

        if (param) {
          return vars[param] ? vars[param] : null;
        }
        return vars;
      },
    },
  }
</script>