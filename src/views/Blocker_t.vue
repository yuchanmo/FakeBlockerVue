<template>
  <v-container>
    <v-row>
      <v-col cols="12" sm="6">
        <v-file-input
          v-model="loaded_img"
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
        <v-card min-height="700px" flat rounded>
          <v-row align="center" justify="center">
            <svg>
              <image></image>
            </svg>
          </v-row>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>
<script>
let default_svgwidth = 360

class ImageInfo {
  constructor(imgfile, svgwidth = 360) {
    let img = new Image()
    img.onload = () => {
      this.height = img.height
      this.width = img.width
    }
    this.original_file = imgfile
    this.svg_width = svgwidth
    this.url = URL.createObjectURL(imgfile)
  }

  set svgwidth(w) {
    this.svg_width = w
  }

  get scale() {
    if (this.width !== 0) {
      return this.svg_width / this.width
    } else {
      return 1
    }
  }
}

export default {
  data: () => ({
    imgs: [],
    loaded_img: '',
    selected_img: ''
  }),
  methods: {
    selectFile() {
      let new_img = new ImageInfo(this.loaded_img, default_svgwidth)
      this.imgs.push(new_img)
      this.selected_img = new_img
    }
  }
}
</script>
<style></style>
