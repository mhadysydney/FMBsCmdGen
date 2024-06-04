<template>
  <q-layout>
    <q-page-container>
      <q-page>
        <q-splitter v-model="splitterModel" class="full-height" style="height: 400px;">

          <template v-slot:before>
            <q-tabs v-model="tab" vertical class="text-black bg-grey-3 text-body2 full-height" align="left"
              indicator-color="primary" style="height: 400px;">
              <q-tab name="commande" icon="keyboard_command_key" label="Commande" />
              <q-separator />
              <q-tab name="histo" icon="history" label="Historique" />
              <q-separator />
            </q-tabs>
          </template>

          <template v-slot:after>
            <q-tab-panels v-model="tab" class="full-height" animated swipeable vertical transition-prev="jump-up"
              transition-next="jump-up">
              <q-tab-panel name="commande" class="full-height">
                <div class="text-h4 q-mb-md">Commande</div>

                <q-input rounded outlined v-model="smsCmd" label="Message à envoyer" class="full-width" />
                <q-btn label="Générer Commande" size="lg" :loading="show" :disable="show" text-color="white"
                  color="primary" @click="genCommande" class="full-width button-text-shadow q-my-md" no-caps />

                <q-card class="bg-white full-width full-height" style="height: 400px;">
                  <q-card-section>
                    <div class="float-right q-px-sm">
                      <q-badge align="middle" class="bg-white text-black text-caption q-pa-sm"
                        v-show="copyMsg.length > 0">{{ copyMsg }}
                        <q-icon color="positive" name="check" />
                      </q-badge>
                      <q-btn color="primary" class="float-right" v-show="showCopy" round flat icon="content_paste"
                        size="md" @click="copyContent(genedCommande)" />
                    </div>
                  </q-card-section>

                  <q-card-section>
                    <div class="text-h6 text-dark">Message Text: {{ smsCmd }}</div>
                    <div class="text-subtitle2 text-dark">Commande: {{ genedCommande }}</div>
                  </q-card-section>
                </q-card>

              </q-tab-panel>

              <q-tab-panel name="histo" class="full-height">
                <q-table flat bordered class="q-ma-md" title="Historique des commandes" :rows="histoData"
                  :columns="cols" row-key="name" :filter="filter" :loading="tLoad"
                  no-data-label="Aucune donnée trouvée">
                  <template v-slot:body-cell-cmd="props">
                    <q-td :props="props">
                      <div class="row q-ma-md">
                        <div class="col-8 text-caption">{{ props.value }}</div>
                        <div class="col-4">
                          <q-badge align="middle" class="bg-white text-black text-caption q-pa-sm"
                            v-show="copyMsg.length > 0">{{ copyMsg }}
                            <q-icon color="positive" name="check" />
                          </q-badge>
                          <q-btn color="primary" class="float-right" round flat icon="content_paste" size="md"
                            @click="copyContent(props.value)" />
                        </div>
                      </div>
                    </q-td>
                  </template>
                  <template v-slot:top-right>
                    <q-input rounded outlined dense v-model="filter" style="margin-left:2px;" placeholder="Search">
                      <template v-slot:append>
                        <q-icon name="search" />
                      </template>
                    </q-input>
                  </template>
                </q-table>
                <!--- <q-card v-for="(hist, index) in histoData" :key="index">
                  <q-card-section>
                    <div class="text-h6 text-black">Message Text: {{ hist.msg }}</div>
                  </q-card-section>
                  <q-card-section>
                    <div class="text-subtitle1 text-black">Commande: {{ hist.cmd }}</div>
                  </q-card-section>
                </q-card>-->
              </q-tab-panel>

            </q-tab-panels>
          </template>

        </q-splitter>
      </q-page>
    </q-page-container>
  </q-layout>
</template>

<script>
import { defineComponent, ref } from 'vue'
import { copyToClipboard } from 'quasar'


export default defineComponent({
  name: 'home-page',

  data() {


    return {
      smsCmd: null,
      tab: ref('commande'),
      filter: '',
      tLoad: false,
      splitterModel: ref(20),
      histoData: [],
      show: false,
      showCopy: true,
      copyMsg: "",
      genedCommande: '',
      cols: [
        { name: 'msg', label: "Message", field: 'message', width: 'auto' },
        { name: 'cmd', label: "Commande", field: 'command', width: 'auto' }
      ]
    }
  },
  methods: {

    getHistoData() {
      console.log("Getting histo data...");
      this.tLoad = true
      const histoD = JSON.parse(localStorage.getItem('histo'))
      this.histoData = typeof histoD !== 'undefined' && histoD ? histoD : []
      this.tLoad = false
      console.log("Histo Data Done!");
    },

    genCommande() {
      if (this.smsCmd) {
        this.show = true
        console.log("Generating command")
        const codec = "0C", type = "05", qtity = "01"
        const cmdEnd = "0D0A"
        var cmdLen = this.smsCmd.length + 2, asciihex = "", ascii = "", packSize = ""
        cmdLen = cmdLen.toString(16).toUpperCase()

        const msgLen = ("00000000" + cmdLen).slice(-8)
        for (let i = 0; i < this.smsCmd.length; i++) {

          let code = this.smsCmd.charCodeAt(i);
          ascii += code
          asciihex += code.toString(16);
        }

        asciihex += cmdEnd
        packSize = (asciihex.length + 16) / 2

        packSize = ("00000000" + packSize.toString(16).toUpperCase()).slice(-8)

        const dataPackage = codec + qtity + type + msgLen + asciihex + qtity

        let crc = this.crc16ibm(dataPackage);
        crc = ("00000000" + crc.toString(16).toUpperCase()).slice(-8)
        this.genedCommande = "00000000" + packSize + dataPackage + crc

        this.genedCommande = this.genedCommande.replace(/(.{2})/g, "$1 ");

        this.genedCommande = this.genedCommande.substring(0, this.genedCommande.length - 1)

        this.histoData.push({ message: this.smsCmd, command: this.genedCommande })

        localStorage.setItem('histo', JSON.stringify(this.histoData))

        this.getHistoData()

      } else console.log("Taper le message svp!");
      this.show = false


    },

    debugCmd(data) {
      console.log("Debugging command")

      console.log("cmdLen: ", data.cmdLen)
      console.log("msgLen: ", data.msgLen)
      console.log("ascii: ", data.ascii)
      console.log("asciihex: ", data.asciihex)
      console.log("packSize: ", data.packSize)
      console.log("dataPackage: ", data.dataPackage)
      console.log("crc: ", crc)
      console.log("genedCommande: ", data.gCmd)
    },

    crc16ibm(message) {

      const table = [
        0x0000, 0xc0c1, 0xc181, 0x0140, 0xc301, 0x03c0, 0x0280, 0xc241,
        0xc601, 0x06c0, 0x0780, 0xc741, 0x0500, 0xc5c1, 0xc481, 0x0440,
        0xcc01, 0x0cc0, 0x0d80, 0xcd41, 0x0f00, 0xcfc1, 0xce81, 0x0e40,
        0x0a00, 0xcac1, 0xcb81, 0x0b40, 0xc901, 0x09c0, 0x0880, 0xc841,
        0xd801, 0x18c0, 0x1980, 0xd941, 0x1b00, 0xdbc1, 0xda81, 0x1a40,
        0x1e00, 0xdec1, 0xdf81, 0x1f40, 0xdd01, 0x1dc0, 0x1c80, 0xdc41,
        0x1400, 0xd4c1, 0xd581, 0x1540, 0xd701, 0x17c0, 0x1680, 0xd641,
        0xd201, 0x12c0, 0x1380, 0xd341, 0x1100, 0xd1c1, 0xd081, 0x1040,
        0xf001, 0x30c0, 0x3180, 0xf141, 0x3300, 0xf3c1, 0xf281, 0x3240,
        0x3600, 0xf6c1, 0xf781, 0x3740, 0xf501, 0x35c0, 0x3480, 0xf441,
        0x3c00, 0xfcc1, 0xfd81, 0x3d40, 0xff01, 0x3fc0, 0x3e80, 0xfe41,
        0xfa01, 0x3ac0, 0x3b80, 0xfb41, 0x3900, 0xf9c1, 0xf881, 0x3840,
        0x2800, 0xe8c1, 0xe981, 0x2940, 0xeb01, 0x2bc0, 0x2a80, 0xea41,
        0xee01, 0x2ec0, 0x2f80, 0xef41, 0x2d00, 0xedc1, 0xec81, 0x2c40,
        0xe401, 0x24c0, 0x2580, 0xe541, 0x2700, 0xe7c1, 0xe681, 0x2640,
        0x2200, 0xe2c1, 0xe381, 0x2340, 0xe101, 0x21c0, 0x2080, 0xe041,
        0xa001, 0x60c0, 0x6180, 0xa141, 0x6300, 0xa3c1, 0xa281, 0x6240,
        0x6600, 0xa6c1, 0xa781, 0x6740, 0xa501, 0x65c0, 0x6480, 0xa441,
        0x6c00, 0xacc1, 0xad81, 0x6d40, 0xaf01, 0x6fc0, 0x6e80, 0xae41,
        0xaa01, 0x6ac0, 0x6b80, 0xab41, 0x6900, 0xa9c1, 0xa881, 0x6840,
        0x7800, 0xb8c1, 0xb981, 0x7940, 0xbb01, 0x7bc0, 0x7a80, 0xba41,
        0xbe01, 0x7ec0, 0x7f80, 0xbf41, 0x7d00, 0xbdc1, 0xbc81, 0x7c40,
        0xb401, 0x74c0, 0x7580, 0xb541, 0x7700, 0xb7c1, 0xb681, 0x7640,
        0x7200, 0xb2c1, 0xb381, 0x7340, 0xb101, 0x71c0, 0x7080, 0xb041,
        0x5000, 0x90c1, 0x9181, 0x5140, 0x9301, 0x53c0, 0x5280, 0x9241,
        0x9601, 0x56c0, 0x5780, 0x9741, 0x5500, 0x95c1, 0x9481, 0x5440,
        0x9c01, 0x5cc0, 0x5d80, 0x9d41, 0x5f00, 0x9fc1, 0x9e81, 0x5e40,
        0x5a00, 0x9ac1, 0x9b81, 0x5b40, 0x9901, 0x59c0, 0x5880, 0x9841,
        0x8801, 0x48c0, 0x4980, 0x8941, 0x4b00, 0x8bc1, 0x8a81, 0x4a40,
        0x4e00, 0x8ec1, 0x8f81, 0x4f40, 0x8d01, 0x4dc0, 0x4c80, 0x8c41,
        0x4400, 0x84c1, 0x8581, 0x4540, 0x8701, 0x47c0, 0x4680, 0x8641,
        0x8201, 0x42c0, 0x4380, 0x8341, 0x4100, 0x81c1, 0x8081, 0x4040
      ];

      var msgBufferArray = new Uint8Array(message.match(/(.{2})/g).map(function (h) {
        return parseInt(h, 16);
      }))

      //console.log("msgBufferArray: ", msgBufferArray)
      //console.log("typeof buffer: ", msgBufferArray.buffer)

      let buffer = msgBufferArray
      let crc = 0x0000;
      for (const byte of buffer) {
        crc = crc >>> 8 ^ table[(crc ^ byte) & 0xff];
      }

      return crc;

    },


    copyContent(textCpy) {
      copyToClipboard(textCpy)
        .then(() => {
          console.log("content coppied");
          this.copyMsg = "Copied!"

          setTimeout(() => {
            this.copyMsg = ""
          }, 3000);
        })
        .catch(() => {
          console.log("copy failed");
        })
    }
  },

  mounted() {
    console.log("Home mounted");
    this.getHistoData()
  }
})
</script>
