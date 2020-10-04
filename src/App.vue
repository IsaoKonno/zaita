<template>
  <v-app>
    <template v-if="isAuth">
      
      <v-app-bar app color="white">
        <template v-if="isAuth">
          <v-img id="logo" ref="logo" src="/img_logo.png" height="50" position="left" contain/>

          <v-spacer></v-spacer>
          <v-btn style="margin-right: .5rem" class="info" @click="showSetting()">設定</v-btn>
          <v-btn style="margin-right: .5rem" class="info" @click="showHistory()">履歴表示</v-btn>
          <v-btn style="margin-right: .5rem" class="info" @click="capture()">再撮影</v-btn>
          <span v-if='this.timer_id'>
            <v-btn class="success">自動撮影 ON</v-btn>
          </span>
          <span v-else>
            <v-btn class="secondary">自動撮影 OFF</v-btn>
          </span>
          <v-btn style="margin-right: .5rem; margin-left: 1rem;" class="green" @click="setStatus('出社')">出社</v-btn>
          <v-btn style="margin-right: .5rem" class="red" @click="setStatus('退社')">退社</v-btn>
          <v-btn style="margin-right: .5rem" class="grey" @click="setStatus('離席')">離席</v-btn>
          <v-btn style="margin-right: .5rem" class="blue" @click="setStatus('在席')">在席</v-btn>
        </template>
      </v-app-bar>
    </template>
      <v-content>
        <template v-if="isAuth">
          <v-container fluid>   
            <v-row no-gutters>
              <!-- <v-col cols="10">
                <v-row no-gutters> -->
                  <v-col v-for="(member, key) in this.sortedMembers" v-bind:key="key">
                    <v-card width="16rem">

                      <template v-if="member.status == '離席'">
                        <v-img src="/leave_seat.png" style="max-width: 100%; max-height 100%;" position="center" />
                      </template>
                      <template v-else-if="member.status == '退社'">
                        <v-img src="/leave_work.png" style="max-width: 100%; max-height 100%;" position="center" />
                      </template>
                      <template v-else-if="member.last_updated_at < (Date.now() - 600000)">
                        <v-img src="/leave_seat_auto.png" style="max-width: 100%; max-height 100%;" position="center" />
                      </template>
                      <template v-else>
                        <img :src="member.image_url" style="max-width: 100%; max-height 100%;" />
                      </template>
                      <div style="position: absolute; top: 0">
                      <v-chip label color="dark"> {{member.name | kondo() }} </v-chip>
                      <v-chip label v-if="member.status == '出社'" color="green">出社 ({{ member.last_updated_at | moment("MM/DD HH:mm")}})</v-chip>
                      <v-chip label v-if="member.status == '退社'" color="red" text-color="white">退社 ({{ member.last_updated_at | moment("MM/DD HH:mm")}})</v-chip>
                      <v-chip label v-if="member.status == '離席'" color="grey" text-color="white">離席 ({{ member.last_updated_at | moment("MM/DD HH:mm")}})</v-chip>
                      <v-chip label v-if="member.status == '在席'" color="blue">在席 ({{ member.last_updated_at | moment("MM/DD HH:mm")}})</v-chip>
                      </div>
                    </v-card>
                  </v-col>
                <!-- </v-row>
              </v-col> -->
              <!-- <v-col cols="2" style="background: red; ">
                ここにボイスチャンネルのツリー的なものが表示される
              </v-col> -->
            </v-row>
          </v-container>
        </template>

        <template v-else>
          <v-btn @click="signIn" class="button-green">signIn</v-btn>
        </template>

        <div id="history-overlay" v-show="isShowHistory">
          <div id="history-content">
            <v-simple-table style="height: 90%; overflow: scroll;" >
              <thead>
              <tr>
                <th>時刻</th>

                <th>ステータス</th>
              </tr>
              </thead>
              <tr v-for="(history, key) in this.histories" v-bind:key="key">
                <td>{{ history.timestamp | moment("YYYY/MM/DD HH:mm:ss") }}</td>
                <td>{{ history.status }}</td>
              </tr>
            </v-simple-table>
            <v-btn color="blue" @click="hideHistory()">閉じる</v-btn>
          </div>
        </div>

        <div id="setting-overlay" v-show="isShowSetting">
          <div id="setting-content">
            <v-row>
              <v-col>
                <header>背景</header>
              </v-col>
            </v-row>
            <v-row>
              <v-col>
                <v-radio-group v-model="bgType" row>
                  <v-radio label="そのまま" value="0"></v-radio>
                  <v-radio label="もやっと" value="1"></v-radio>
                  <v-radio label="バーチャル" value="2"></v-radio>
                </v-radio-group>
              </v-col>
            </v-row>
            <v-row>
              <v-col>
                <header>バーチャル背景ファイル</header>
              </v-col>
            </v-row>
            <v-row>
              <v-col>
                <v-file-input name="vimage" @change="onFileChange"  accept="image/*" />
              </v-col>
              <v-col>
                <img ref="preview" id="preview" width="320" height="240"/>
              </v-col>
            </v-row>
            <v-row>
              <v-col>
                <v-select
                  item-text="label"
                  item-value="deviceId"
                  :items="videoDevices"
                  v-model="activeVideoDeviceId"
                >
                </v-select>
              </v-col>
            </v-row>
            <v-btn color="blue" @click="hideSetting()">閉じる</v-btn>
          </div>
        </div>

        </v-content>
        <v-footer app color="white">
          <div>
            <video ref="video" id="video" width="320" height="240" autoplay muted></video>            
            <canvas ref="canvas" id="canvas" width="320" height="240"></canvas>
            <canvas ref="canvas2" id="canvas2" width="320" height="240"></canvas>
          </div>
        </v-footer>
  </v-app>

</template>

<script>
import firebase from 'firebase'
import moment from "moment"
import * as bodyPix from '@tensorflow-models/body-pix'

export default {
  name: 'App',
  created: function() {
    this.storage = firebase.storage();

    this.database = firebase.database();
    this.membersRef = this.database.ref('member-status');

    var _this = this;
    this.membersRef.on('value', function(snapshot) {
      _this.members = snapshot.val(); // データに変化が起きたときに再取得する
    });
  },
  async mounted () {
    this.loadSettings()
    await this.loadBodyPix()
    this.video = this.$refs.video
    this.canvas = this.$refs.canvas
    this.canvasContext = this.canvas.getContext("2d")

    var _this = this
    navigator.mediaDevices.enumerateDevices().then(function(devices) {
        devices.forEach(function(device) {
          if(device.kind === 'videoinput'){
            _this.videoDevices.push(device)
            console.log(device)
          }
        });
      }
    )
    .catch(function(err) {
        console.error('enumerateDevide ERROR:', err);
      }
    );

    firebase.auth().onAuthStateChanged((user) => { 
      this.isAuth = !!user
      this.user = user
      var _this = this;
      this.database.ref('member-status' + '/' + this.user.uid).once('value', function(snapshot) {
        console.log("onAuthStateChanged : %s", snapshot.val())
        console.log(snapshot.val())
        _this.status = snapshot.val().status
        _this.last_updated_at = snapshot.val().last_updated_at
        _this.my_image =  snapshot.val().image_url
        if(_this.isPresent(snapshot.val())){
          console.log("user is present. start capture.")
          setTimeout(_this.startCapture, 5 * 1000)
        }
      })
    } )

  },
  methods: {
    signIn: function () {
      const provider = new firebase.auth.GoogleAuthProvider()
      firebase.auth().signInWithRedirect(provider)
    },
    async capture () {
      console.log("capture at : %s", moment().format('YYYYMMDDHHmmss'))
      var canvas2 = this.$refs.canvas2
      this.canvasContext.drawImage(this.video, 0, 0, 320, 240)
      var imgData = this.canvasContext.getImageData(0, 0, 320, 240)
      if(! this.isPhotoTaken(imgData) ){
        this.takePhotoFailedCount++
        console.log("take photo failed. count: " + this.takePhotoFailedCount)
        if(this.takePhotoFailedCount > 3){
          this.takePhotoFailedCount = 0
          location.reload(true);
        }
      }
      if(this.bgType == 1){
        var segment = await this.segmentPerson(this.canvas , this.net)
        await this.bokehEffect(canvas2, this.canvas, segment);
        this.my_image = canvas2.toDataURL("image/jpeg") 

      }else if(this.bgType == 2){
        var bgImg = this.$localStorage.get('bgImage')
        if (! bgImg ){
          bgImg = "/bg.jpg"
        }
        await this.virtualBackground(this.canvas, this.canvasContext, canvas2, bgImg)
        // this.my_image = this.canvas.toDataURL("image/jpeg")
        this.my_image = canvas2.toDataURL("image/jpeg") 
      }else{
        this.my_image = this.canvas.toDataURL("image/jpeg")
      }

      this.last_uodated_at = Date.now()
      var currentDate = moment().format('YYYYMMDD')
      var currentTime = moment().format('YYYYMMDDHHmmss')

      if(this.user){
        var imgRef = this.storage.ref().child("images/" + currentDate + "/" +  this.user.uid + "/" + currentTime +  ".jpg")
        var _this = this
        imgRef.putString(this.my_image, 'data_url').then(function(snapshot){
          console.log(snapshot)
          imgRef.getDownloadURL().then(function(url){
            console.log(url)
            _this.database.ref('member-status/' + _this.user.uid ).set({
              name: _this.user.displayName,
              status: _this.status,
              image_url: url,
              last_updated_at: Date.now()
            })          
          })
        })
      }
    },
    setStatus(status) {
      this.status = status
      if(status == '退社' || status == '離席'){
        this.stopCapture()
        this.capture()
      }else{
        this.startCapture()
      }

      var currentTime = moment().format('YYYYMMDDHHmmss')
      var currentTimeMillis = Date.now()

      this.database.ref('status-history/' + this.user.uid + '/' + currentTime ).set({
        status: this.status,
        timestamp: currentTimeMillis,
        name: this.user.displayName
      })

    },
    startCapture() {
      if(this.timer_id){
        clearTimeout(this.timer_id)
      }

      if (this.video.srcObject == null){
        if(navigator.mediaDevices && navigator.mediaDevices.getUserMedia) {
          var constraints = { video :true }
          if(this.activeVideoDeviceId){
            constraints = { video: { deviceId: this.activeVideoDeviceId }}
          }
          console.log(this.activeVideoDeviceId)
          console.log(constraints)
          navigator.mediaDevices.getUserMedia(constraints).then(stream => {
            this.video.srcObject = stream
            this.video.play()

            this.stream = stream
            this.timer_id = setTimeout(this.startCapture, 5 * 1000)
            return
          })
        }
      }else{
        this.capture()
        this.timer_id = setTimeout(this.startCapture, 180 * 1000)
      }
    },
    stopCapture() {
      if(this.timer_id){
        clearTimeout(this.timer_id)
        this.timer_id = null;
      }

      if (this.stream.getVideoTracks()){
        console.log("close stream videoTrack..")
        this.stream.getVideoTracks().forEach(track => {
          if (track.readyState == 'live') {
            track.stop();
            this.video.srcObject = null;
            return;
          }
        });
      }
    },
    showHistory(){
      var _this = this
      this.database.ref('status-history/' + this.user.uid).once('value', function(snapshot) {
        console.log(snapshot.val())
        _this.histories = snapshot.val()
      })
      this.isShowHistory = true
    },
    hideHistory(){
      this.isShowHistory = false
    },
    showSetting(){
      var bgImg = this.$localStorage.get('bgImage')
      if (! bgImg ){
        bgImg = "/bg.jpg"
      }
      this.$refs.preview.src = bgImg
      this.isShowSetting = true
    },
    hideSetting(){
      this.$localStorage.set('bgType', this.bgType) 
      this.$localStorage.set('activeVideoDeviceId', this.activeVideoDeviceId) 
      this.isShowSetting = false
    },
    async loadBodyPix() {
      this.net = await bodyPix.load()
    },
    async segmentPerson(img) {
      const option = {
        flipHorizontal: false,
        internalResolution: 'medium',
        segmentationThreshold: 0.5,
        maxDetections: 4,
        scoreThreshold: 0.5,
        nmsRadius: 20,
        minKeypointScore: 0.3,
        refineSteps: 10
      };

      return await this.net.segmentPerson(img, option);
      // return this.net.segmentPerson(img);
    },
    isPhotoTaken(imgData){
      for(var i = 0; i < imgData.data.length; i++){
        if(imgData.data[i] > 0){
          console.log("imagedata ok index: " + i + " value: " + imgData.data[i])
          return true
        }
      } 
      return false
    },
    trasparentBackground(imgData, segmentation){
      var segmentData = segmentation.data
      for(var i = 0; i < segmentData.length; i++){
        if(segmentData[i] == 0){
          imgData.data[i * 4 + 3] = 0
        }
      } 
      return imgData
    },
    async virtualBackground(srcCanvas, srcCanvasContext, destCanvas, bgSrc){
      var segment = await this.segmentPerson(srcCanvas , this.net)
      console.log(segment.data)

//独自実装
      var destCanvasContext = destCanvas.getContext('2d')

      var imageData = srcCanvasContext.getImageData(0,0,320,240)
      imageData = this.trasparentBackground(imageData, segment)
      console.log(imageData.data)
      
      srcCanvasContext.putImageData(imageData, 0, 0)

      destCanvasContext.clearRect(0,0,320,240)
      var bgimg = await this.loadImage(bgSrc)
      console.log(bgimg)
      destCanvasContext.drawImage(bgimg, 0, 0, 320, 240)
      destCanvasContext.drawImage(srcCanvas, 0, 0, 320, 240)


//TFでなんとかならないか。。。
      // this.maskCanvas(destCanvas, srcCanvas, segment)
      // srcCanvasContext.clearRect(0,0,320,240)
      // var bgimg = await this.loadImage(bgSrc)
      // console.log(bgimg)
      // srcCanvasContext.drawImage(bgimg, 0, 0, 320, 240)
      // srcCanvasContext.drawImage(destCanvas, 0, 0, 320, 240)
    },
    maskCanvas(canvas, img, segmentation) {
      // マスクを作成
      const fgColor = { r: 0, g: 0, b: 0, a: 0 };  // 人物
      const bgColor = { r: 0, g: 0, b: 0, a: 255 }; // 背景
      const colorMask = bodyPix.toMask(segmentation, fgColor, bgColor);

      // マスクを使って描画
      const opacity = 1.0;
      const flipHorizontal = false;
      const maskBlurAmount = 0;
      bodyPix.drawMask(
        canvas, img, colorMask, opacity, maskBlurAmount,
        flipHorizontal);

    },
    async bokehEffect(canvas, img, segment) {
      const backgroundBlurAmount = 7;
      const edgeBlurAmount = 3;
      const flipHorizontal = false;

      console.log(segment)

      bodyPix.drawBokehEffect(
          canvas, img, segment, backgroundBlurAmount,
          edgeBlurAmount, flipHorizontal);
    },
    loadImage(src) {
      return new Promise((resolve, reject) => {
        const img = new Image();
        img.onload = () => resolve(img);
        img.onerror = (e) => reject(e);
        img.src = src;
      })
    },
    saveSettings(){
      //noop
    },
    loadSettings(){
      this.bgType = this.$localStorage.get('bgType')
      this.activeVideoDeviceId = this.$localStorage.get('activeVideoDeviceId')
    },
    async onFileChange(evt){
      if(! evt ){
        console.log("file not found.")
        return
      }
      var fileData = await this.loadFile(evt)
      this.$localStorage.set('bgImage', fileData)

      this.$refs.preview.src = fileData
      // var img = await this.loadImage(fileData)
      
      // var canvas2 = this.$refs.canvas2
      // canvas2.getContext('2d').drawImage(img, 0, 0, 320, 240)
    },
    loadFile(file){
      return new Promise((resolve, reject) => {
        const reader = new FileReader();
        reader.onload = (e) => resolve(e.target.result);
        reader.onerror = (e) => reject(e);
        reader.readAsDataURL(file);
      })
    },
    isPresent(member){
      if(member.status === "在席" || member.status === "出社"){
        return true
      }
      return false
    }
  },
  computed:{
    sortedMembers() {
      return Object.values(this.members).sort((a, b) => {
        if(this.isPresent(a)){
          return -1
        }else if(this.isPresent(b)){
          return 1
        }
        return 0
      })
    }
    
  },
  localStrage:{
    isBokeh: {
      type: Boolean
    },
    bgType: {
      type: Number
    },
    bgImage: {
      type: String
    },
    activeVideoDeviceId: {
      type: String
    }
  },
  data () {
    return {
      storage: null,
      database: null,
      membersRef: null,
      audio: null,
      members: [],
      videoDevices: [],
      activeVideoDeviceId: null,
      video: {},
      canvas: {},
      canvasContext: null,
      my_image: null,
      captures: [],
      isAuth: false,
      user: null,
      status: "ステータス",
      last_uodated_at: "最終更新日時",
      timer_id: null,
      isShowHistory: false,
      histories: null,
      isShowSetting: false,
      net: null,
      isBokeh: false,
      bgType: 0, // 0:そのまま, 1:もやっと, 2:バーチャル
      takePhotoFailedCount: 0
    }
  },
  filters: {
    /**
     * @param {Date} value    - Date オブジェクト
     * @param {string} format - 変換したいフォーマット
     */
    moment(value, format) {
      return moment(value).format(format);
    },
    kondo(value){
      var result = value;
      if (Date.now() % 2 == 0){
        result = value.replace('今野', '近藤')
        result = value.replace('貴哉', '拓哉')
      }
      return result 
    }
  }
}
</script>

<style>
#canvas {
  display: none;
}
#canvas2 {
  display: none;
}
#video {
  display: none;
  background-color: #000000;
}
.col {
  margin-bottom: 0.5rem;
}
#history-overlay{
  z-index:1;

  position:fixed;
  top:0;
  left:0;
  width:100%;
  height:100%;
  background-color:rgba(0,0,0,0.5);

  display: flex;
  align-items: center;
  justify-content: center;

}
#history-content{
  z-index:2;
  width:50%;
  height:75%;
  padding: 1em;
  background:#fff;
}

#setting-overlay{
  z-index:1;

  position:fixed;
  top:0;
  left:0;
  width:100%;
  height:100%;
  background-color:rgba(0,0,0,0.5);

  display: flex;
  align-items: center;
  justify-content: center;

}
#setting-content{
  z-index:2;
  width:50%;
  height:75%;
  padding: 1em;
  background:#fff;
}
</style>

