<template>
  <div> 
    <el-row>
      <el-col :span="6">
        <el-form-item label="Background" label-width="100">
          <el-color-picker v-model="alteka.bg" :predefine="colors"></el-color-picker>
        </el-form-item>
      </el-col>
      <el-col :span="6">
        <el-form-item label="Foreground" label-width="100">
          <el-color-picker v-model="alteka.fg" :predefine="colors"></el-color-picker>
        </el-form-item>
      </el-col>
      <el-col :span="6">
        <el-form-item label="Center Text" label-width="100">
          <el-tooltip effect="dark" content="Only applies to text over custom logo" placement="bottom" :open-delay="500" :disabled="alteka.showLogo">
            <el-color-picker v-model="alteka.textColour" :disabled="!alteka.showLogo" :predefine="colors"></el-color-picker>
          </el-tooltip>
        </el-form-item>
      </el-col>
      <el-col :span="6">
        <el-form-item label="Gradient" label-width="100">
          <el-switch v-model="alteka.gradient"></el-switch>
        </el-form-item>
      </el-col>
    </el-row>
    <el-row>
      <el-col :span="8">
        <el-form-item label="Custom Logo" label-width="100">
          <el-switch v-model="alteka.showLogo"></el-switch>
        </el-form-item>
      </el-col>
      <el-col :span="9" v-if="alteka.showLogo">
        <el-button-group>
          <el-tooltip effect="dark" content="Image will be masked to 3:1 aspect ratio" placement="bottom" :open-delay="500">
            <el-button icon="el-icon-picture" size="small" v-on:click="selectImage()">Select Image</el-button>
          </el-tooltip>
          <el-button size="small" v-on:click="clearImage()">Clear</el-button>
        </el-button-group>
      </el-col>
      <el-col  v-if="alteka.showLogo" :span="7">
        <el-image v-if="alteka.logo != ''" style="width: 135px; height: 45px" :src="alteka.logo" fit="cover"></el-image>
      </el-col>
    </el-row>
  </div>
</template>

<script>
const { ipcRenderer } = require('electron')
  export default {
    props: {
      alteka: Object,
      colors: Array
    },
    methods: {
      selectImage: function() {
        ipcRenderer.send('selectImage')
      },
      clearImage: function() {
        this.alteka.logo = ""
      }
    }
  }
</script>

<style scoped>
  
</style>
