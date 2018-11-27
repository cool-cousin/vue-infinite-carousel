<template>
  <div class="carousel-wrapper" :class="{ 'align-top': alignTop, dragging, draggable }">
    <div class="carousel" @pointerdown="onPointerDown" @touchstart.passive="onTouchStart" @touchmove="onTouchMove" @touchend="onTouchEnd" ref="carousel">
      <div class="slides" :style="style" @transitionend.self="onTransitionEnd">
        <div v-for="(slide, index) in $slides" class="slide" :style="slideStyle" :key="slide.key">
          <slot v-if="slide.value" name="slide" :prop="slide.key" :slide="slide.value" :index="index" :selected="index === $current" :previous="selectPrevious" :next="selectNext" :dragging="dragging"></slot>
        </div>
      </div>
    </div>
    <slot :select="select" :previous="selectPrevious" :next="selectNext" :total="total" :current="current" :transform="style" :slides="$slides"></slot>
  </div>
</template>

<script>
import { isBoolean, map, size } from 'lodash'
import { disableBodyScroll, enableBodyScroll, clearAllBodyScrollLocks } from 'body-scroll-lock';

const KEY_LEFT = 37
const KEY_RIGHT = 39

export default {
  name: 'Carousel',
  props: {
    slides: null,
    disableKeys: {
      type: Boolean,
      default: false
    },
    loop: {
      type: Boolean,
      default: true
    },
    autoplay: {
      type: Boolean,
      default: false
    },
    interval: {
      type: Number,
      default: 5000
    },
    draggable: {
      type: Boolean,
      default: true
    },
    duration: {
      type: Number,
      default: 300,
      validator: ms => ms >= 0
    },
    start: {
      type: Number,
      default: 0
    },
    virtual: {
      type: Boolean,
      default: false
    },
    width: {
      type: Number,
      default: 100,
      validator: width => width >= 0
    },
    alignTop: {
      type: Boolean,
      default: false
    }
  },
  data () {
    return {
      pointerEventsSupported: false,
      mounted: false,
      current: this.start,
      offsetX: 0,
      clientX: 0,
      indexOffset: 0,
      threshold: 60,
      animate: false,
      dragging: false,
      easeOut: false,
      carouselWidth: 0,
      timeoutId: null
    }
  },
  mounted () {
    this.mounted = true
    this.pointerEventsSupported = 'onpointerdown' in window
    window.addEventListener('keyup', this.keyHandler)
    window.addEventListener('resize', this.calculateCarouselWidth)
    window.addEventListener('pointermove', this.onPointerMove)
    window.addEventListener('pointerup', this.onPointerUp)
    window.addEventListener('pointercancel', this.onPointerUp)
    this.$nextTick(this.calculateCarouselWidth)

    if (this.autoplay) {
      this.timeoutId = setTimeout(() => this.selectNext(), this.interval)
      // prevent autoplay while dragging
      this.$watch('dragging', dragging => {
        if (dragging) clearTimeout(this.timeoutId)
      }, { immediate: true })
    }
  },
  beforeDestroy () {
    window.removeEventListener('keyup', this.keyHandler)
    window.removeEventListener('resize', this.calculateCarouselWidth)
    window.removeEventListener('pointermove', this.onPointerMove)
    window.removeEventListener('pointerup', this.onPointerUp)
    window.removeEventListener('pointercancel', this.onPointerUp)
    clearAllBodyScrollLocks()
    clearTimeout(this.timeoutId)
  },
  computed: {
    enabled () {
      return this.mounted && this.draggable
    },
    transition () {
      if (this.easeOut) {
        return `transform ${this.duration / 2}ms ease-out`
      } else {
        return `transform ${this.duration}ms ease`
      }
    },
    slideOffset () {
      return (100 - this.width) / 2
    },
    slideStyle () {
      const { width, slideOffset } = this
      return {
        width: `${width}%`,
        margin: slideOffset && `0 -${slideOffset}% 0 ${slideOffset}%`
      }
    },
    style () {
      const percent = this.carouselWidth > 0 ? (this.offsetX / this.carouselWidth * 100) : 0
      const translateX = (this.$current * -this.width) + percent

      return {
        transition: this.animate ? this.transition : 'none',
        transform: `translateX(${translateX}%)`
      }
    },
    last () {
      return this.total - 1
    },
    total () {
      return size(this.slides)
    },
    showPagination () {
      return this.total > 1
    },
    loopable () {
      return this.total >= 3
    },
    $current () {
      if (this.$virtual) return this.indexOffset + 1
      if (this.$loop) return this.current + this.indexOffset + 1
      return this.current
    },
    $loop () {
      return this.loop && this.loopable
    },
    $virtual () {
      return this.virtual && this.loopable
    },
    $$slides () {
      return map(this.slides, (value, key) => ({ key, value }))
    },
    $slides () {
      const slides = this.$$slides
      const isBackwards = this.indexOffset < 0
      const isForwards = this.indexOffset > 0
      if (this.$virtual) {
        const currentSlide = slides[this.current]
        const nextSlide = slides[this.next]
        const previousSlide = slides[this.previous]
        if (isBackwards) return [currentSlide, nextSlide, previousSlide]
        if (isForwards) return [nextSlide, previousSlide, currentSlide]
        return [previousSlide, currentSlide, nextSlide]
      } else if (this.$loop) {
        const firstSlide = slides[0]
        const lastSlide = slides[this.last]

        const lastFirst = [lastSlide, ...slides.slice(0, -1)]
        const firstLast = [{ key: 'virtual-0' }, { key: 'virtual-1' }, ...slides.slice(1), firstSlide]
        if (this.isLast) return isBackwards ? lastFirst : firstLast
        if (this.isFirst) return isForwards ? firstLast : lastFirst

        return [{ key: 'virtual-2' }, ...slides]
      } else {
        return slides
      }
    },
    currentSlide () {
      const { $current, $slides } = this
      return $slides[$current]
    },
    isFirst () {
      return this.current === 0
    },
    isLast () {
      return this.current === this.last
    },
    next () {
      return this.isLast ? 0 : Math.min(this.current + 1, this.last)
    },
    previous () {
      return this.isFirst ? this.last : Math.max(this.current - 1, 0)
    }
  },
  watch: {
    currentSlide (slide) {
      if (this.indexOffset === 0) this.$emit('selected', { ...slide, index: this.current })
    },
    start (index) {
      if (index === -1 || index === this.current) return
      this.select(index, false)
    },
    last (index) {
      if (this.current > index) this.select(index, false)
    }
  },
  methods: {
    onTouchStart ({ changedTouches }) {
      if (this.pointerEventsSupported) return
      disableBodyScroll(this.$refs.carousel)
      const [touch] = changedTouches
      if (touch) this.onPointerDown(touch)
    },
    onTouchMove ({ changedTouches }) {
      if (this.pointerEventsSupported) return
      const [touch] = changedTouches
      if (touch) this.onPointerMove(touch)
    },
    onTouchEnd ({ changedTouches }) {
      if (this.pointerEventsSupported) return
      enableBodyScroll(this.$refs.carousel)
      const [touch] = changedTouches
      if (touch) this.onPointerUp(touch)
    },
    onPointerDown ({ clientX }) {
      if (!this.enabled) return
      this.clientX = clientX
      this.panStart()
    },
    onPointerMove ({ clientX }) {
      if (!this.enabled || !this.dragging) return
      const deltaX = clientX - this.clientX
      this.panMove({ deltaX })
    },
    onPointerUp ({ clientX, type }) {
      if (!this.enabled || !this.dragging) return
      const distance = type === 'pointercancel' ? 0 : Math.abs(clientX - this.clientX)
      const deltaTime = 0
      this.clientX = 0
      this.panEnd({ deltaTime, distance })
    },
    calculateCarouselWidth () {
      const { carousel } = this.$refs
      this.carouselWidth = carousel ? carousel.getBoundingClientRect().width : 0
    },
    onTransitionEnd () {
      this.easeOut = false
      this.indexOffset = 0
      this.animate = false
      if (this.autoplay) this.timeoutId = setTimeout(() => this.selectNext(), this.interval)
    },
    panStart () {
      if (this.carouselWidth === 0) this.calculateCarouselWidth()
      this.dragging = true
      this.animate = false
    },
    panMove ({ deltaX }) {
      this.offsetX = deltaX
    },
    panEnd ({ deltaTime, distance }) {
      this.dragging = false
      if (distance > this.threshold) {
        this.easeOut = deltaTime <= this.duration
        if (this.offsetX > 0) {
          this.selectPrevious()
        } else {
          this.selectNext()
        }
      } else {
        this.animate = true
      }
      this.offsetX = 0
    },
    selectNext (animate) {
      const { isLast, next } = this
      this.select(next, animate)
      if (this.$virtual) {
        this.indexOffset = 1
      } else if (this.$loop) {
        this.indexOffset = isLast ? this.total : 0
      }
    },
    selectPrevious (animate) {
      const { isFirst, previous } = this
      this.select(previous, animate)
      if (this.$virtual) {
        this.indexOffset = -1
      } else if (this.$loop) {
        this.indexOffset = isFirst ? -this.total : 0
      }
    },
    select (index, animate) {
      if (this.autoplay && this.timeoutId && this.current !== index) clearTimeout(this.timeoutId)
      this.current = index
      this.indexOffset = 0
      this.animate = isBoolean(animate) ? animate : true
    },
    keyHandler (e) {
      if (this.disableKeys) return
      if (e.keyCode === KEY_LEFT) this.selectPrevious()
      else if (e.keyCode === KEY_RIGHT) this.selectNext()
    }
  }
}
</script>

<style scoped>
.carousel-wrapper {
  position: relative;
  overflow-x: hidden;
  pointer-events: none;
}

.draggable > .carousel {
  pointer-events: auto;
  touch-action: pan-y;
  cursor: grab;
}

.draggable.dragging > .carousel {
  cursor: grabbing;
}

.slides {
  white-space: nowrap;
  will-change: transform;
  backface-visiblity: hidden;
}

.slide {
  white-space: normal;
  display: inline-block;
  vertical-align: middle;
}

.align-top > .carousel .slide {
  vertical-align: top; 
}

.carousel, .slides, .slide {
  width: 100%;
  height: 100%;
}
</style>
