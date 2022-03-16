<template>
  <div>
    <v-row>
      <v-col cols="12" md="12" class="d-flex justify-center align-center">
        <v-card style="max-width:600px; min-width:600px;">
          <v-card-title>Кошелек MetaMask</v-card-title>
          <v-card-text>
            <v-row>
              <v-col cols="12" md="12">
                <span v-if="currentAccount">Подключен кошелек: <br>{{ currentAccount }}</span>
              </v-col>
            </v-row>
          </v-card-text>
          <v-card-actions class="justify-center">
            <v-btn v-if="isMetamaskSupported && !currentAccount" @click="connectWallet">
              Подключить
            </v-btn>
            <span v-else-if="!isMetamaskSupported">Кошелек MetaMask недоступен или не подключен</span>
            <span v-else-if="contract == undefined || contract == null">Проверьте правильность выбора сети. Должна быть Ropsten.</span>
          </v-card-actions>
        </v-card>
      </v-col>
    </v-row>
    <v-row v-if="currentAccount && contract">
      <v-col cols="12" md="12" class="d-flex justify-center align-center">
        <v-card style="max-width:600px; min-width:600px;">
          <v-card-title>Действия с контрактом</v-card-title>
          <v-card-text>
            <v-row>
              <v-col cols="12" md="8">
                <v-text-field v-model="contractAddress" label="Адрес контракта" />
              </v-col>
            </v-row>
            <v-row>
              <v-col cols="12" md="12">
                <h3 class="mb-5">
                  Операция Approve
                </h3>
                <v-row>
                  <v-col cols="12" md="4">
                    <v-text-field v-model="senderAccount" label="Адрес" />
                  </v-col>
                  <v-col cols="12" md="4">
                    <v-text-field v-model="ammount" label="Сумма" />
                  </v-col>
                  <v-col cols="12" md="4" class="mt-4">
                    <v-btn :disabled="(!senderAccount || senderAccount == '') || (!ammount || ammount == '') || approveWaiting" @click="sendApprove">
                      Approve
                    </v-btn>
                    <v-progress-circular
                      v-if="approveWaiting"
                      indeterminate
                      color="primary"
                    />
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" md="12">
                    <span v-if="approveTransactionHash !== ''">Tx: <a :href="`https://ropsten.etherscan.io/tx/${approveTransactionHash}`" target="_blank">{{ approveTransactionHash }}</a></span>
                  </v-col>
                </v-row>
              </v-col>
            </v-row>
            <v-row>
              <v-col cols="12" md="12">
                <h3 class="mb-5">
                  Проверка Allowance
                </h3>
                <v-row>
                  <v-col cols="12" md="8">
                    <v-text-field v-model="senderAccount" label="Отправитель" />
                  </v-col>
                  <v-col cols="12" md="4" class="mt-4">
                    <v-btn :disabled="(!senderAccount || senderAccount == '') || (!currentAccount || currentAccount == '') || allowanceWaiting" @click="checkAllowance">
                      Allowance
                    </v-btn>
                    <v-progress-circular
                      v-if="allowanceWaiting"
                      indeterminate
                      color="primary"
                    />
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" md="12">
                    <span v-if="allowanceAmmount">{{ allowanceAmmount }}</span>
                  </v-col>
                </v-row>
              </v-col>
            </v-row>
            <v-row>
              <v-col cols="12" md="12">
                <h3 class="mb-5">
                  Отправка токенов TransferFrom
                </h3>
                <v-row>
                  <v-col cols="12" md="3">
                    <v-text-field v-model="senderAccount" label="Адрес отправителя" />
                  </v-col>
                  <v-col cols="12" md="3">
                    <v-text-field v-model="receiverAccount" label="Адрес получателя" />
                  </v-col>
                  <v-col cols="12" md="2">
                    <v-text-field v-model="transferAmmount" label="Сумма" />
                  </v-col>
                  <v-col cols="12" md="4" class="mt-4">
                    <v-btn :disabled="(!senderAccount || senderAccount == '') || (!receiverAccount || receiverAccount == '') || (!transferAmmount || transferAmmount == '') || transferWaiting" @click="sendTransferFrom">
                      Transfer
                    </v-btn>
                    <v-progress-circular
                      v-if="transferWaiting"
                      indeterminate
                      color="primary"
                    />
                  </v-col>
                </v-row>
                <v-row>
                  <v-col cols="12" md="12">
                    <span v-if="transferTransactionHash !== ''">Tx: <a :href="`https://ropsten.etherscan.io/tx/${transferTransactionHash}`" target="_blank">{{ transferTransactionHash }}</a></span>
                  </v-col>
                </v-row>
              </v-col>
            </v-row>
          </v-card-text>
        </v-card>
      </v-col>
    </v-row>
  </div>
</template>

<script>
import detectEthereumProvider from '@metamask/detect-provider'
import Web3 from 'web3'
import { abi, networks } from '~/contracts/contractAbi.json'

export default {
  name: 'IndexPage',
  data: () => ({
    isMetamaskSupported: false,
    provider: null,
    chainId: null,
    currentAccount: null,
    contractAddress: null,
    contract: null,
    senderAccount: null,
    receiverAccount: null,
    ammount: null,
    approveTransactionHash: '',
    allowanceAmmount: null,
    transferTransactionHash: '',
    transferAmmount: null,
    approveWaiting: false,
    allowanceWaiting: false,
    transferWaiting: false
  }),
  async mounted () {
    this.provider = await detectEthereumProvider()
    if (this.provider) {
      this.isMetamaskSupported = Boolean(window.ethereum && window.ethereum.isMetaMask)
      if (this.isMetamaskSupported) {
        this.chainId = await window.ethereum.request({ method: 'eth_chainId' })
        await window.ethereum.request({ method: 'eth_accounts' })
          .then(this.handleAccountsChanged)

        const web3 = new Web3(this.provider, null, { transactionConfirmationBlocks: 1 })
        const networkId = await web3.eth.net.getId()
        if (networks[networkId] !== undefined && networks[networkId].address !== undefined) {
          this.contractAddress = networks[networkId].address
          this.contract = new web3.eth.Contract(
            abi,
            networks[networkId].address
          )
        }
      }
    }
  },
  methods: {
    async connectWallet () {
      await this.checkConnectedAccounts()
    },
    async checkConnectedAccounts () {
      await window.ethereum.request({ method: 'eth_requestAccounts' })
        .then(this.handleAccountsChanged)
        .catch(() => null)
    },
    handleAccountsChanged (accounts) {
      if (accounts.length === 0) {
        this.currentAccount = null
      } else {
        this.currentAccount = accounts[0]
      }
    },
    async sendApprove () {
      this.approveWaiting = true
      await this.contract.methods.approve(this.senderAccount, this.ammount).send({ from: this.currentAccount })
        .then((res) => {
          if (res) {
            this.approveTransactionHash = res.transactionHash
          }
          this.approveWaiting = false
        })
        .catch((error) => {
          console.log(error)
          this.approveWaiting = false
        })
    },
    async checkAllowance () {
      this.allowanceWaiting = true
      await this.contract.methods.allowance(this.currentAccount, this.senderAccount).call()
        .then((res) => {
          if (res) {
            this.allowanceAmmount = res
          }
          this.allowanceWaiting = false
        })
        .catch((error) => {
          console.log(error)
          this.allowanceWaiting = false
        })
    },
    async sendTransferFrom () {
      this.transferWaiting = true
      await this.contract.methods.transferFrom(this.senderAccount, this.receiverAccount, this.transferAmmount).send({ from: this.currentAccount })
        .then((res) => {
          if (res) {
            this.transferTransactionHash = res.transactionHash
          }
          this.transferWaiting = false
        })
        .catch((error) => {
          console.log(error)
          this.transferWaiting = false
        })
    }
  }
}
</script>
