<template>
  
  <div class="train">
  <img v-if="mode == 'home'" class="logo" alt="Logo logo" src="../assets/logo.jpg" height="200px">
  <img v-if="mode != 'home'"  alt="Logo logo" src="../assets/logo.jpg" height="70px"
       v-bind:class="{'new-logo':mode != 'home'}">
    <template v-if="mode == 'train'">
        <h2 class="head">Please show your face expression according to the mood below</h2>
        <h1 class="feel-text">{{emotion_choice}}</h1>
    </template>
    <template v-if="mode == 'home'">
        <h1 class="app-name">Face Detection Checker</h1>
    </template>
    <template v-if="mode == 'test'">
        <h1 class="head">Please show your face expression according to the mood below</h1>
        <h1 class="feel-text">{{randomChoice()}}</h1>
    </template>
    <Camera v-if="mode != 'home'"></Camera>
    <template v-if="mode == 'train'">
        <button class="train-button" v-on:click="trainModel()">Face record</button>
        <h3 class="sub-head">Please confirm the face several times for each option</h3>
    </template>
    <template v-if="mode == 'home'">
        <button class="start" @click="changeOption('train')">START</button>
    </template>
    <template v-else>
        <button v-if="mode == 'test'" class="train-button" v-on:click="getEmotion()">Detect</button>
        <button v-if="mode == 'train' && count.angry == 0" class="train-button mg-10" @click="changeOption('test')">Start</Button> 
        <button disabled v-if="mode == 'train' && count.angry != 0" class="mg-10 disabled-button" @click="changeOption('test')">Start</Button> 
        <button v-if="mode != 'home'" class="train-button mg-10" @click="changeOption('home')">Home Page</button>
    </template>
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
      async trainModel(){
        if(this.count.happy > 0){
          this.count.happy -= 1
          if(this.count.happy == 0){
            this.emotion_choice = 'neutral'
          }
          this.class = 0
        }
        else if(this.count.neutral > 0){
          this.count.neutral -= 1
          if(this.count.neutral == 0){
            this.emotion_choice = 'angry'
          }
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
        await this.addExample();
      },
      async addExample(){
        const img = tf.fromPixels(this.$children[0].webcam.webcamElement);
        const logits = this.mobilenet.infer(img, 'conv_preds');
        await this.classifier.addExample(logits, parseInt(this.class));
        if(this.class == 0 && this.count.happy == 2){
          this.$swal('Detect your face complete', '( 1/3 )','success');
        }
        else if(this.class == 0 && this.count.happy == 1){
          this.$swal('Detect your face complete', '( 2/3 )','success');
        }
        else if(this.class == 0 && this.count.happy == 0){
          this.$swal('Detect your face complete', '( 3/3 )','success');
        }
        else if(this.class == 1 && this.count.neutral == 2){
          this.$swal('Detect your face complete', '( 1/3 )','success');
        }
        else if(this.class == 1 && this.count.neutral == 1){
          this.$swal('Detect your face complete', '( 2/3 )','success');
        }
        else if(this.class == 1 && this.count.neutral == 0){
          this.$swal('Detect your face complete', '( 3/3 )','success');
        }
        else if(this.class == 2  && this.count.angry == 2){
          this.$swal('Detect your face complete', '( 1/3 )','success');
        }
        else if(this.class == 2  && this.count.angry == 1){
          this.$swal('Detect your face complete', '( 2/3 )','success');
        }
        else if(this.class == 2  && this.count.angry == 0){
          this.$swal('Detect your face complete', '( 3/3 )','success');
        }
      },
      async getEmotion(){
        const img = tf.fromPixels(this.$children[0].webcam.webcamElement);
        const logits = this.mobilenet.infer(img, 'conv_preds');
        const pred = await this.classifier.predictClass(logits);
        console.log(pred);
        if(this.emotions[pred.classIndex] == this.emotion_choice){
          this.detected_e = "correct"
          this.$swal('Correct!!!', '','success');
        }
        else{
          this.detected_e = "incorrect"
          this.$swal('Incorrect!!!', '','error');
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

<style>
.new-logo{
  display: flex;
  border-radius: 100%;
  padding-top: 20px;
  margin: 10px;
}
.app-name{
  font-size: 3rem;
  color:#F0F3F4;
}
.start{
  width: 200px;
  color:#BDC3C7;
  cursor: pointer;
  background-color: #212F3D;
  font-size:1.5rem;
  border: 4px solid #F7DC6F;
  border-radius: 50%;
}
.start:hover{
  background-color: #fff;
  color:#F1C40F;
  border:4px solid #F1C40F;
}
.head{
  font-size:2rem;
  color:#F0F3F4;
  margin: 0px;
}
.sub-head{
  margin-top: 0px;
  color:#F0F3F4;
}
.feel-text{
  font-size:3rem;
  color:#F7DC6F;
  margin: 10px;
}
.train-button{
  width: 100px;
  margin-top: 10px;
  margin-bottom: 10px;
  color:#fff;
  padding: 5px 10px;
  cursor: pointer;
  background-color: #212F3D;
  border: 2px solid #F7DC6F;
  border-radius: 20px;
}
.train-button:hover{
  background-color: #fff;
  color:#F1C40F;
  border:2px solid #F1C40F;
}
.disabled-button{
  width: 100px;
  margin-top: 10px;
  margin-bottom: 10px;
  color: #CCD1D1;
  cursor: default;
  padding: 5px 10px;
  background-color: #212F3D;
  border: 2px solid #FCF3CF;
  border-radius: 20px;
}
.disabled-button:hover{
  border: 2px solid #FCF3CF;
}
.mg-10{
  margin: 0px 10px;
}
</style>
