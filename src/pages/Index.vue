<template>
  <q-page class="flex flex-center column">
    <q-slide-transition>
      <div v-show="!checkSlideSVG" style="width: 280px">
        <div class="text-h4 q-mb-lg">Create check</div>
        <q-input v-model="seed" outlined label="Insert seed phrase"
          :rules="[
            val => !!val || 'Field is required',
            val => isValidMnemonic(val) || 'Is not valid seed phrase',
          ]"
        />
        <q-input class="q-mt-sm" v-model.number="amount" type="number" outlined label="Amount bip" />
        <q-btn class="q-mt-lg full-width" label="Make check" color="secondary" size="1.1em" @click="makeCheck()" />
      </div>
    </q-slide-transition>
    <q-slide-transition>
      <div v-show="checkSlideSVG" style="width: 310px">
        <div class="text-h4 q-mb-md q-mt-lg">Save check</div>
        <q-card>
          <q-card-section>
            <div id="checkSvg"></div>
          </q-card-section>
        </q-card>
        <div class="q-mb-md q-mt-md flex justify-between">
          <q-btn label="Save SVG" icon="save_alt" color="secondary" @click="saveAs('svg')" />
          <q-btn label="Save PNG" icon="save_alt" color="secondary" @click="saveAs('png')" />
          <q-btn @click="share(resultLink)" v-if="shareTest()" color="positive" class="q-mt-md" icon="share" label="Share" />
        </div>
      </div>
    </q-slide-transition>
  </q-page>
</template>

<script>
import { issueCheck, prepareLink, TX_TYPE } from 'minter-js-sdk'
import { walletFromMnemonic, isValidMnemonic } from 'minterjs-wallet'
import SVG from 'svg.js'
import { saveSvgAsPng, saveSvg } from 'save-svg-as-png/lib/saveSvgAsPng.js'
import QRCode from 'qrcode'
export default {
  name: 'Create',
  data () {
    return {
      checkSlideSVG: false,
      S: null,
      amount: null,
      seed: null,
      linkCheck: null,
      balance: 0,
      nonce: 1,
      wallet: null,
      check: null
    }
  },
  created () {},
  methods: {
    isValidMnemonic (seed) {
      if (isValidMnemonic(seed)) return true
      else return false
    },
    shareTest () {
      if (navigator.share) return true
      else return false
    },
    share () {
      navigator.share({
        title: 'Minter check',
        text: 'Open BIP wallet',
        url: this.linkCheck
      })
        .then(() => console.log('Successful share'))
        .catch(error => console.log('Error sharing', error))
    },
    makeCheck () {
      if (isValidMnemonic(this.seed)) {
        this.nonce = new Date().getTime() - 1582480000000
        this.check = issueCheck({
          privateKey: this.wallet.getPrivateKeyString(),
          passPhrase: 'hello',
          nonce: this.nonce,
          chainId: 1,
          coin: 'BIP',
          value: this.amount || 0,
          dueBlock: 99999999
        })
        this.checkSlideSVG = true
        this.makeSvg()
      }
    },
    makeSvg () {
      this.S = SVG('checkSvg').size(290, 480)
      this.S.viewbox(0, 0, 290, 480)
      this.S.text('Minter check: ').font({ size: 16 }).x(5).y(290)
      this.S.text(this.amount + ' BIP').font({ size: 16, weight: 'bold' }).x(105).y(290)
      this.S.text(add => {
        let arrText = this.check.match(/.{1,50}/g)
        arrText.forEach(item => {
          add.tspan(item).newLine()
        })
      }).font({ size: 10 }).x(5).y(320)

      this.S.text('Check password: ').font({ size: 16 }).x(5).y(413)
      this.S.text('hello').font({ size: 16, weight: 'bold' }).x(133).y(413)

      const txParams = {
        type: TX_TYPE.REDEEM_CHECK,
        data: {
          check: this.check,
          password: 'hello'
        },
        gasCoin: 'BIP'
      }
      this.linkCheck = prepareLink({ ...txParams, password: 'hello' })

      let svgLink = this.S.link(this.linkCheck)
      svgLink.text('Open deeplink wallet').font({ size: 16, fill: '#1a0dab', 'text-decoration': 'underline' }).x(5).y(440)
      svgLink.target('_blank')

      const opts = {
        errorCorrectionLevel: 'M',
        type: 'image/png',
        width: 280,
        margin: 0
      }
      let qrImage
      QRCode.toDataURL(this.linkCheck, opts).then(url => {
        qrImage = this.S.image(url)
        qrImage.size(280, 280).move(5, 5)
      }).catch(err => {
        console.error(err)
      })
    },
    saveAs (type) {
      switch (type) {
        case 'png':
          saveSvgAsPng(this.S.node, 'Check_' + this.nonce + '.png', {
            backgroundColor: '#fff'
          })
          break
        case 'svg':
          saveSvg(this.S.node, 'Check_' + this.nonce + '.svg')
          break
        default:
          return false
      }
    }
  },
  watch: {
    seed (val) {
      if (isValidMnemonic(val)) {
        this.wallet = walletFromMnemonic(this.seed)
      }
    }
  }
}
</script>
<style>
  .main-wrap {
    width: 280px
  }
</style>
