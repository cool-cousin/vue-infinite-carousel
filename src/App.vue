<template>
  <div id="app">
    <h2>Vue Infinite Carousel</h2>
    <div class="options">
      <label>
        <input type="checkbox" v-model="autoplay" />
        <span>autoplay</span>
      </label>
      <label>
        <input type="checkbox" v-model="virtual" />
        <span>virtual</span>
      </label>
      <label>
        <span>width:</span>
        <input type="number" min="0" max="100" v-model="width" />
      </label>
    </div>
    <carousel class="carousel" :slides="slides" :width="Number(width)" @selected="onSelect" :virtual="virtual" :autoplay="autoplay">
      <div slot="slide" slot-scope="{ slide }" class="image" :style="{ backgroundImage: `url(${slide.image})` }">
        <span class="image-title">{{ slide.title }}</span>
      </div>
    </carousel>
  </div>
</template>

<script>
import Carousel from './components/Carousel.vue'

export default {
  name: 'app',
  components: {
    Carousel
  },
  data () {
    return {
      width: 80,
      autoplay: true,
      virtual: true
    }
  },
  computed: {
    slides () {
      return [
        { image: 'https://placekitten.com/g/300/300', title: 'Slide 1' },
        { image: 'https://placekitten.com/g/440/300', title: 'Slide 2' },
        { image: 'https://placekitten.com/g/360/300', title: 'Slide 3' },
        { image: 'https://placekitten.com/g/420/300', title: 'Slide 4' },
        { image: 'https://placekitten.com/g/380/300', title: 'Slide 5' }
      ]
    }
  },
  methods: {
    onSelect (slide) {
      console.log(slide)
    }
  }
}
</script>

<style>
#app {
  font-family: 'Avenir', Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  margin-top: 60px;
}

.options {
  margin-bottom: 20px;
}

.carousel {
  max-width: 500px;
  margin: 0 auto;
}

.image {
  height: 300px;
  margin: 0 10px;
  display: flex;
  justify-content: center;
  align-items: center;
  background-position: center;
  background-repeat: no-repeat;
  background-size: cover;
  user-select: none;
}

label + label {
  margin-left: 1em;
}

label span {
  margin: .4em;
}

.image-title {
  font-weight: bold;
  font-size: 36px;
  color: #fff;
  text-shadow: 1px 1px 0 #000;
}
</style>
