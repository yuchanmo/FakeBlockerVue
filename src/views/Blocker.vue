<template>
  <v-container>
    <v-row>
      <v-col cols="12" sm="6">
        <v-file-input
          v-model="loadedImg"
          :rules="rules"
          accept="image/png, image/jpeg, image/bmp"
          placeholder="Load a Image"
          prepend-icon="mdi-camera"
          label="Pic"
          @change="selectFile"
          enctype="multipart/form-data"
        >
        </v-file-input>
      </v-col>
    </v-row>
    <v-row>
      <v-col cols="12">
        <v-card class="justify-center" rounded>
          <v-container>
            <v-row justify="center" align="center">
              <v-col align="center" justify="center">
                <h3>Confidence : {{ selectedConfidence }} %</h3>
              </v-col>
            </v-row>
            <v-row align="center" justify="center" class="pa-ma-5">
              <v-col cols="12" align="center" justify="center">
                <v-overlay :absolute="absolute" :value="overlay">
                  <v-container>
                    <v-row>
                      <v-col>
                        <v-progress-circular
                          :size="70"
                          :width="7"
                          color="purple"
                          indeterminate
                        ></v-progress-circular>
                      </v-col>
                    </v-row>
                    <v-row>
                      <v-col>
                        <h3>{{ overlayMsg }}</h3>
                      </v-col>
                    </v-row>
                  </v-container>
                </v-overlay>

                <svg
                  v-if="selectedImg !== null"
                  class="ma-pa-5"
                  :width="svgWidth"
                  :height="svgHeight"
                >
                  <image
                    :href="imgUrl"
                    :width="svgWidth"
                    :height="svgHeight"
                  ></image>

                  <rect
                    class="face"
                    v-for="(box, i) in faceBoxes"
                    @mouseover="displayConfidence(box)"
                    @mouseleave="clearConfidence()"
                    :key="i"
                    :x="box[0]"
                    :y="box[1]"
                    :width="box[2]"
                    :height="box[3]"
                    stroke="white"
                    fill="transparent"
                    stroke-width="2"
                  ></rect>
                </svg>
              </v-col>
            </v-row>

            <v-row>
              <v-col>
                <v-divider class="pa-ma-1"></v-divider>
              </v-col>
            </v-row>
            <v-row style="height: 90px" align="center" justify="center">
              <v-col>
                <v-card-text>
                  <v-slider
                    v-model="selectedTick"
                    :tick-labels="ticksLabels"
                    step="10"
                    thumb-label
                    ticks
                  ></v-slider>
                </v-card-text>
              </v-col>
            </v-row>
          </v-container>
        </v-card>
      </v-col>
    </v-row>
    <v-row style="height: 30px">
      <v-col align="center" justify="center">
        <v-divider></v-divider>
      </v-col>
    </v-row>

    <v-row>
      <v-col cols="12">
        <v-card min-height="500px" rounded>
          <v-container>
            <v-row>
              <v-col
                v-for="(img, i) in imgs"
                :key="i"
                class="d-flex child-flex ma-pa-5"
                cols="4"
              >
                <v-img
                  @click="selectImage(img)"
                  :src="img.ori.url"
                  aspect-ratio="1"
                  class="grey lighten-2 archiveimg ma-pa-3"
                >
                </v-img>
              </v-col>
            </v-row>
          </v-container>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>
<script>
import axios from 'axios'
import { EventBus } from '../eventbus'

//let base_url = 'http://127.0.0.1:5000/'
let base_url = 'http://mojjijji.iptime.org:7777/'
let api_urls = {
  detect: `${base_url}detect`,
  generate: `${base_url}generate`
}
let default_svgwidth = 360

class ImageWrapper {
  constructor(originalImgFile) {
    this.images = Array(11)
    this.images[10] = new ImageInfo(originalImgFile)
    this.ori = this.images[10]
    console.log(this.images)
  }

  setImageIntoArray(img, idx) {
    this.images[idx] = new ImageInfo(img, this.svg_width)
  }
}

class ImageInfo {
  constructor(imgfile) {
    this.url = URL.createObjectURL(imgfile)
    this.file = imgfile
    let img = new Image()
    this.height = 0
    this.width = 0
    let self = this
    this.faceboxes = null
    img.onload = () => {
      self.height = img.height
      self.width = img.width
      this.detectFacesInImage()
    }
    img.src = this.url
  }

  async detectFacesInImage() {
    try {
      let payload = new FormData()
      payload.append('image', this.file)
      payload.append('detector', 'mtcnn')
      EventBus.$emit('DisplayProgressRing', { flag: true, msg: 'Detect Faces' })
      let res = await axios.post(api_urls['detect'], payload)
      this.faceboxes = res.data
      EventBus.$emit('DisplayProgressRing', { flag: false, msg: '' })
      EventBus.$emit('SetFaceBoxes')
    } catch (error) {
      alert(error)
    }
  }
}

export default {
  data: () => ({
    imgs: [],
    loadedImg: null,
    selectedImg: null,
    ticksLabels: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10],
    selectedTick: 100,
    selectedConfidence: 0,
    flagConfidence: false,
    overlay: false,
    overlayMsg: '',
    faceBoxes: [],
    selectedSvgWidth: default_svgwidth
  }),
  mounted() {
    EventBus.$on('DisplayProgressRing', (payload) =>
      this.toggleLoadingOverlay(payload['flag'], payload['msg'])
    )
    EventBus.$on('SetFaceBoxes', () => this.setFaceBoxes())
  },
  computed: {
    tick() {
      return this.selectedTick / 10
    },
    svgWidth() {
      return this.selectedSvgWidth
    },
    svgHeight() {
      if (this.selectedImg !== undefined && this.selectedImg !== null) {
        let img_height = this.selectedImg.images[this.tick].height
        return img_height * this.scaleForSvg
      } else {
        return 1
      }
    },
    scaleForSvg() {
      if (this.selectedImg !== undefined && this.selectedImg !== null) {
        let img_width = this.selectedImg.images[this.tick].width
        return img_width !== 0 ? this.selectedSvgWidth / img_width : 1
      } else {
        return 1
      }
    },
    imgUrl() {
      return this.selectedImg !== undefined && this.selectedImg !== null
        ? this.selectedImg.images[this.tick].url
        : ''
    }
    // faceBoxes() {

    // }
  },

  watch: {
    selectedTick(val) {
      this.requestNoise(val)
      this.setFaceBoxes()
    }
  },
  methods: {
    selectFile() {
      //let new_img = new ImageInfo(this.loadedImg, default_svgwidth)
      let new_img = new ImageWrapper(this.loadedImg, default_svgwidth)
      this.imgs.push(new_img)
      this.selectedImg = new_img
      console.log(new_img)
    },
    selectImage(i) {
      //this.selectedTick = null
      this.selectedTick = 100
      this.selectedImg = i
      this.setFaceBoxes()
    },
    setFaceBoxes() {
      try {
        if (
          this.selectedImg !== undefined &&
          this.selectedImg !== null &&
          this.selectedImg.images[this.tick] !== undefined &&
          this.selectedImg.images[this.tick] !== null &&
          this.selectedImg.images[this.tick].faceboxes !== null
        ) {
          let original_boxes = this.selectedImg.images[this.tick].faceboxes
          this.faceBoxes = original_boxes.map((face) =>
            face.map((f, i) => {
              if (i < 4) {
                //face box x,y,w,h
                return f * this.scaleForSvg
              } //confidence
              else {
                return f
              }
            })
          )
        } else {
          this.faceBoxes = []
        }
      } catch (error) {
        alert(error)
      }
    },
    displayConfidence(b) {
      this.flagConfidence = true
      this.selectedConfidence = b[4].toFixed(3) * 100
    },
    clearConfidence() {
      this.flagConfidence = false
      this.selectedConfidence = 0
    },
    toggleLoadingOverlay(flag, msg) {
      this.overlay = flag
      this.overlayMsg = flag ? msg : ''
    },
    async requestNoise(threshold) {
      if (this.selectedImg.images[this.tick] === undefined) {
        // EventBus.$emit('DisplayProgressRing', {
        //   flag: true,
        //   msg: 'Generate Noise'
        // })
        let req_url = api_urls['generate']
        let payload = new FormData()
        payload.append('image', this.selectedImg.ori.file)
        payload.append('threshold', 100 - threshold)
        try {
          let res = await axios.post(req_url, payload, { responseType: 'blob' })
          let data = res.data
          let b = new Blob([data])
          let img = new ImageInfo(b, default_svgwidth)
          this.selectedImg.images[this.tick] = img
          // EventBus.$emit('DisplayProgressRing', { flag: false, msg: '' })
        } catch (error) {
          console.log(error)
        }
      }

      //alert(threshold)
    }
  }
}
</script>
<style>
.archiveimg:hover {
  border-style: solid;
  border-color: black;
  color: black;
}
.face:hover {
  stroke-width: 5px;
  stroke: royalblue;
  color: black;
}
</style>
