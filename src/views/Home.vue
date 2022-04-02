<template>
  <div class="train">
    <template v-if="mode == 'train'">
        <h2>โปรดทำใบหน้าที่แสดงอารมณ์ของท่านตามตัวเลือกด้านล่าง</h2>
        <h1>{{emotion_choice}}</h1>
    </template>
    <template v-if="mode == 'home'">
        <h1>Face Detection Checker</h1>
    </template>
    <template v-if="mode == 'test'">
        <h1>Take a picture to let us know how you feel about our service</h1>
        <h1>{{randomChoice()}}</h1>
    </template>
    <Camera v-if="mode != 'home'"></Camera>
    <template v-if="mode == 'train'">
        <button v-on:click="trainModel()">ยืนยันใบหน้า</button>
        <h3>กรุณายืนยันใบหน้าหลายๆรอบในแต่ละอารมณ์</h3>
    </template>
    <template v-if="mode == 'home'">
        <button @click="changeOption('train')">Start</button>
    </template>
    <template v-else>
        <button v-if="mode == 'test'" v-on:click="getEmotion()">ตรวจจับใบหน้า</button>
        <button v-if="mode == 'train' && count.angry == 0" @click="changeOption('test')">เริ่มเล่น</Button> 
        <button disabled v-if="mode == 'train' && count.angry != 0" @click="changeOption('test')">เริ่มเล่น</Button> 
        <button v-if="mode != 'home'" @click="changeOption('home')">หน้าหลัก</button>
    </template>

    <h1 v-if="mode == 'test'">{{ detected_e }}</h1>
  </div>
</template>

<script>
// @ is an alias to /src
import Camera from "@/components/Camera.vue";
import * as tf from '@tensorflow/tfjs';
import * as mobilenetModule from '@tensorflow-models/mobilenet';
import * as knnClassifier from '@tensorflow-models/knn-classifier';
import axios from 'axios';

export default {
  name: "Home",
  components: {
    Camera
  },
  data: function(){
      return {
          emotions: ['happy','neutral', 'angry'],
          emotion_choice: 'happy',
          classifier: null,
          mobilenet: null,
          class: null,
          detected_e: null,
          mode: 'home',
          count:{
            happy: 3,
            neutral: 3,
            angry: 3,
          },
      }
  },
  mounted: function(){
      this.init();
  },
  methods: {
      async init(){
        // load the load mobilenet and create a KnnClassifier
        this.classifier = knnClassifier.create();
        this.mobilenet = await mobilenetModule.load();
      },
      prepareModel(){
      
      },
      trainModel(){
        if(this.count.happy > 0){
          this.count.happy -= 1
          if(this.count.happy == 0){
            this.emotion_choice = 'neutral'
          }
          this.class = 0
        }
        else if(this.count.neutral > 0){
          this.emotion_choice = 'neutral'
          this.count.neutral -= 1
          this.class = 1
        }
        else if(this.count.angry > 0){
          this.emotion_choice = 'angry'
          this.count.angry -= 1
          this.class = 2
        }
        console.log(this.class)
        // let selected = document.getElementById("emotion_options");
        // this.class = selected.options[selected.selectedIndex].value;
        this.addExample();
      },
      addExample(){
        const img= tf.fromPixels(this.$children[0].webcam.webcamElement);
        const logits = this.mobilenet.infer(img, 'conv_preds');
        this.classifier.addExample(logits, parseInt(this.class));
      },
      async getEmotion(){
        const img = tf.fromPixels(this.$children[0].webcam.webcamElement);
        const logits = this.mobilenet.infer(img, 'conv_preds');
        const pred = await this.classifier.predictClass(logits);
        console.log(pred);
        if(this.emotions[pred.classIndex] == this.emotion_choice){
          this.detected_e = "correct"
        }
        else{
          this.detected_e = "incorrect"
        }
        // this.detected_e = this.emotions[pred.classIndex];
        this.randomChoice();
        this.registerEmotion();
      },
      randomChoice() {
        this.emotion_choice = this.emotions[Math.floor(Math.random() * this.emotions.length)];
        return this.emotion_choice
      },
      changeOption(mode){
        this.mode = mode;
      },
      registerEmotion(){
          axios.post('http://localhost:3128/callback', {
              'emotion': this.detected_e
          }).then( () => {
              alert('Thanks for letting us know how you feel');
          });
      }
    }
};
</script>
