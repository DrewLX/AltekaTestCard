<template>
<el-row class="menu">

  <el-col id="enableLabel" :span="8" style="margin-top: 3px;" :class="{ enabledText: config.visible }">
    <span class="pointer" @click="config.visible = !config.visible">
      <i class="fas fa-power-off pointer" :class="{ green: config.visible, red:!config.visible }"></i> Enable </span>
      <el-switch style="margin-left: 4px;" v-model="config.visible"></el-switch>
  </el-col>

  <el-col :span="8">
      <el-button type="success" size="mini" round v-on:click="drawerAudio = true"><i v-if="config.audio.enabled" class="fas fa-volume-up"></i><i v-if="!config.audio.enabled" class="fas fa-volume-mute"></i> Audio</el-button>
      <el-button type="success" size="mini" round v-on:click="drawerImage = true"><i class="fas fa-image"></i> Export</el-button>
  </el-col>

  <el-col :span="8" style="text-align: right">
    <el-popover v-model="confirmResetVisible" ref="confirmReset" placement="top-end" :offset="-12">
      <el-row>Reset all settings to default?</el-row>
      <el-row style="text-align: right; margin-top: 10px;">
        <el-button type="primary" size="mini" round v-on:click="ipcSend('resetDefault'); confirmResetVisible = false"><i class="fas fa-undo"></i> Reset</el-button>
      </el-row>
    </el-popover>

    <el-dropdown size="mini" trigger="click" @command="handleDropdown">
      <el-button size="mini" type="primary">
        More<i class="el-icon-arrow-up el-icon--right"></i>
      </el-button>
      <el-dropdown-menu slot="dropdown">
        <el-dropdown-item v-popover:confirmReset ><i class="fas fa-undo"></i> Reset</el-dropdown-item>
        <el-dropdown-item divided command="openHelp"><i class="fas fa-question"></i> Help</el-dropdown-item>
        <el-dropdown-item divided command="openLogs"><i class="fas fa-clipboard-list"></i> Logs</el-dropdown-item>
        <el-dropdown-item divided command="exportSettings"><i class="fas fa-file-export"></i> Export Settings</el-dropdown-item>
        <el-dropdown-item command="importSettings"><i class="fas fa-file-import"></i> Import Settings</el-dropdown-item>
      </el-dropdown-menu>
    </el-dropdown>
  </el-col>


  <el-drawer :with-header="false" :visible.sync="drawerAudio" direction="btt" size="100px">
    <el-row class="drawerContent">
      <el-checkbox-group v-model="config.audio.options" size="medium">
        <el-checkbox-button label="voice">Voice</el-checkbox-button>
        <el-checkbox-button label="tone">Tone</el-checkbox-button>
        <el-checkbox-button label="pink">Pink Noise</el-checkbox-button>
        <el-checkbox-button label="white">White Noise</el-checkbox-button>
        <el-checkbox-button label="stereo">Stereo</el-checkbox-button>
        <el-checkbox-button label="phase">Phase</el-checkbox-button>
      </el-checkbox-group>
    </el-row>
    <el-row >
      <el-col style="margin-left: 38px; margin-top: 7px; color: #606266;" :span="4" :class="{ enabledText: config.audio.enabled }">
        Enable <el-switch :disabled="config.audio.options.length == 0" v-model="config.audio.enabled"></el-switch>
      </el-col>
      <el-col :span="18">
        <el-form-item label="Audio Device">
        <el-select v-model="config.audio.deviceId" placeholder="Select" style="width: 321px;">
            <el-option v-for="item in audioDevices" :key="item.deviceId" :label="item.label" :value="item.deviceId"></el-option>
        </el-select>
      </el-form-item>
      </el-col>
    </el-row>

  </el-drawer>

  <audio src="~@/assets/audio/stereo.wav" id="stereo" />
  <audio src="~@/assets/audio/phase.wav" id="phase" />
  <audio src="~@/assets/audio/pink.wav" id="pink" />
  <audio src="~@/assets/audio/white.wav" id="white" />
  <audio src="~@/assets/audio/tone.wav" id="tone" />
  <audio :src="config.audio.voiceData" id="voice" />



 <el-drawer :with-header="false" :visible.sync="drawerImage" direction="btt" size="100px">
   <el-row class="drawerContent">
     <el-col :span="10">
       <el-radio-group v-model="config.export.imageSource" size="medium" :disabled="config.cardType=='audioSync'">
         <el-radio-button label="card">Test Card</el-radio-button>
         <el-tooltip :disabled="!config.windowed || config.cardType=='audioSync'" effect="dark" content="Disable windowed output and then disable 'Fill Output' to save test card within larger canvas" placement="bottom" :open-delay="500">
          <el-tooltip :disabled="!config.fullsize || config.cardType=='audioSync'" effect="dark" content="Disable 'Fill Output' to save test card within larger canvas" placement="bottom" :open-delay="500">
            <el-radio-button label="canvas" :disabled="config.fullsize">Whole Canvas</el-radio-button>
          </el-tooltip>
         </el-tooltip>
       </el-radio-group>
     </el-col>
     <el-col :span="11">
       <el-radio-group v-model="config.export.target" size="medium" :disabled="config.cardType=='audioSync'">
         <el-radio-button label="file">Save to File</el-radio-button>
         <el-radio-button label="wallpaper">Set Wallpaper</el-radio-button>
       </el-radio-group>
     </el-col>
     <el-col :span="3">
       <el-button size="medium" :disabled="config.cardType=='audioSync'" v-on:click="exportCard">OK</el-button>
     </el-col>
   </el-row>
   <el-row>
     <el-alert v-if="config.cardType == 'audioSync'" title = "Choose a different card - Even we can't save AV Sync to a  still image..." type="error" center show-icon effect="dark" :closable="false"></el-alert>
     
     
  </el-row>
</el-drawer>

</el-row>
</template>

<script>
const log = require('electron-log')
const { ipcRenderer, remote } = require('electron')
import { Loading, Notification } from 'element-ui'
let loadingInstance

  export default {
    props: {
      config: Object
    },
    data: function() {
      return {
        confirmResetVisible: false,
        drawerAudio: false,
        drawerImage: false,
        curAudio: null,
        playing: false,
        name: "",
        voiceTimer: null,
        audioDevices: []
      }
    },
    mounted: function() {
      this.updateDevices()
      setInterval(this.updateDevices, 5000)
      setTimeout(this.doNameUpdate, 2000)

      ipcRenderer.on('exportCardCompleted', function(event, msg) {
        if (msg) {
          Notification.warning({title: 'Oops', message: msg, showClose: false, duration: 2500, onClick: function() { this.close() }})
        } 
        loadingInstance.close()
      })

      ipcRenderer.on('importSettings', function(event, msg) {
        Notification.info({title: 'Import Settings', message: msg, showClose: false, duration: 2500, onClick: function() { this.close() }})
      })
      
    },
    watch: {
      config: {
        handler: function (val, oldVal) { 
            document.getElementById('stereo').setSinkId(val.audio.deviceId)
            document.getElementById('phase').setSinkId(val.audio.deviceId)
            document.getElementById('pink').setSinkId(val.audio.deviceId)
            document.getElementById('white').setSinkId(val.audio.deviceId)
            document.getElementById('tone').setSinkId(val.audio.deviceId)
            document.getElementById('voice').setSinkId(val.audio.deviceId)

            if (val.name != this.name) {
              this.name = val.name
              this.doNameUpdate()
            }

            if (val.audio.enabled && !this.playing) {
              log.info('Starting audio output')
              this.curAudio = null // so it starts from the first item
              this.playNext()
            }
            if (val.audio.options.length == 0) {
              this.stopAudio()
              this.config.audio.enabled = false // stop playing if no options selected
            }
            if (!val.audio.enabled && this.playing) {
              this.stopAudio()
            }
            if (val.fullsize == 1) {
              this.imageSource = "card"
            }

            if (val.fullsize || val.windowed ) {
              this.config.export.imageSource = "card"
            }
         },
        deep: true
      },
    },
    methods: {
      updateDevices: function() {
        navigator.mediaDevices.enumerateDevices().then((devices) => {
          this.audioDevices = devices.filter(device => device.kind === 'audiooutput').filter(device => device.deviceId != 'communications')
        })  
      },
      ipcSend: function(val) {
        ipcRenderer.send(val)
      },
      exportCard: function() {
        ipcRenderer.send('exportCard')
        loadingInstance = Loading.service({ fullscreen: true, text:"Capturing Test Card", background: 'rgba(0, 0, 0, 0.85)'})
        this.drawerImage = false        
      },
      handleDropdown: function(command) {
        switch (command) {
          case 'openHelp':
            ipcRenderer.send('openUrl', 'https://alteka.solutions/kards/help')    
            break;

          case 'importSettings':
            ipcRenderer.send('importSettings')
            break;
            
          case 'exportSettings':
            ipcRenderer.send('exportSettings')
            break;

          case 'openLogs':
            ipcRenderer.send('openLogs')
            break;
        }
      },
      stopAudio: function() {
        log.info('Stopping audio output')
        this.stopFile('tone')
        this.stopFile('white')
        this.stopFile('pink')
        this.stopFile('phase')
        this.stopFile('stereo')
        this.playing = false
      },
      stopFile: function(file) {
        let f = document.getElementById(file);
        f.pause()
        f.currentTime = 0
      },
      playNext: function() {
        if (this.config.audio.enabled) {
          this.playing = true
          let opts = this.config.audio.options

          if (this.curAudio == null && opts.length > 0) {
            this.curAudio = opts[0]
          } else if (opts.length > 0) {
            var curIndex = opts.indexOf(this.curAudio)
            if (opts[curIndex + 1] == undefined) {
              this.curAudio = opts[0]
            } else {
              this.curAudio = opts[curIndex + 1]
            }
          }
    
          this.playFile(this.curAudio)
        } else {
          this.playing = false
        }
      },
      playFile: function(file) {
        let vm = this
        var x = document.getElementById(file)
        x.play()
        x.onended = function() {
          setTimeout(vm.playNext(), 500)
        }
      },
      doNameUpdate: function() {
        clearTimeout(this.voiceTimer)
        this.voiceTimer = setTimeout(this.updateName, 1000)
      },
      updateName: function() {
        ipcRenderer.send('createVoice')
      }
    }
  }
</script>

<style scoped>
  .menu {
    padding: 8px;
    background-color: #3D3D3B;
    color: white;
    border-top: 3px solid #6AB42F;
  }
  .drawerContent {
    text-align: center;
    padding: 10px;
    outline: none;
  }
  .enabledText {
    color: #6AB42F !important;
  }
  .pointer:hover {
    cursor:pointer;
  }
  .el-alert {
    width: 95%;
    margin-left: 2.5%;
    margin-bottom: 10px;
  }
  .red {
    color: #d11;
    margin-right: 5px;
  }
</style>
