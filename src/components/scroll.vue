<!-- 使用说明:
  如果嵌套使用，需要指定class，并且绑定class到组件
  通过caninit绑定是否立刻渲染
  如果不需要下拉刷新，上拉加载，绑定:pullDownRefresh="false" :pullUpLoad="false" -->
<template>
  <div ref="wrapper" class="list-wrapper scroll-component">
    <div class="scroll-content" ref="innerContent" id="innerContent">
      <div style="width:100%;height:0px;border:0"></div>
      <slot></slot>
      <slot name="pullup" :pullUpLoad="pullUpLoad" :isPullUpLoad="isPullUpLoad">
        <div class="pullup-wrapper" v-if="pullUpLoad">
          <div class="before-trigger" v-if="!isPullUpLoad">
            <span>{{pullUpTxt}}</span>
          </div>
          <div class="after-trigger" v-else>
            <loading></loading>
          </div>
        </div>
      </slot>
    </div>
    <slot name="pulldown" :pullDownRefresh="pullDownRefresh" :pullDownStyle="pullDownStyle" :beforePullDown="beforePullDown" :isPullingDown="isPullingDown" :bubbleY="bubbleY">
      <div ref="pulldown" class="pulldown-wrapper" :style="pullDownStyle" v-if="pullDownRefresh">
        <div class="before-trigger" v-if="beforePullDown">
          <bubble :y="bubbleY"></bubble>
        </div>
        <div class="after-trigger" v-else>
          <div v-if="isPullingDown" class="loading">
            <loading></loading>
          </div>
          <div v-else><span>{{refreshTxt}}</span></div>
        </div>
      </div>
    </slot>
  </div>
</template>
<script>
  import BScroll from 'better-scroll'
  import Loading from './loading.vue'
  import Bubble from './bubble.vue'
  const COMPONENT_NAME = 'scroll'
  const DIRECTION_H = 'horizontal'
  const DIRECTION_V = 'vertical'
  export default {
    name: COMPONENT_NAME,
    props: {
      data: {   //列表的数据
        type: Array,
        default: function () {
          return []
        }
      },
      probeType: {
        type: Number,
        default: 1    /* 1 滚动的时候会派发scroll事件，会截流。 * 2 滚动的时候实时派发scroll事件，不会截流。 * 3 除了实时派发scroll事件，在swipe的情况下仍然能实时派发scroll事件 */
      },
      click: {
        type: Boolean,
        default: true  //点击列表是否派发click事件
      },
      scrollX: {
        type: Boolean,
        default: false  //是否开启横向滚动
      },
      listenScroll: {
        type: Boolean,
        default: false  //是否派发滚动事件
      },
      listenBeforeScroll: {
        type: Boolean,
        default: false
      },
      direction: {
        type: String,
        default: DIRECTION_V
      },
      scrollbar: {
        type: null,
        default: false
      },
      pullDownRefresh: {
        type: null,
        default: function(){
          return {
            threshold: 90,
            stop: 40,
            txt: '刷新成功'
          }
        }
      },
      pullUpLoad: {
        type: null,
        default: function(){
          return {
            threshold: 10,
            txt: {more: "", noMore: ''},
          }
        }
      },
      pullup: {
        type: Boolean,
        default: false  //是否派发滚动到底部的事件，用于上拉加载
      },
      pulldown: {
        type: Boolean,
        default: false   //是否派发顶部下拉的事件，用于下拉刷新
      },
      beforeScroll: {
        type: Boolean,
        default: false  //是否派发列表滚动开始的事件
      },
      refreshDelay: {
        type: Number,
        default: 20    //当数据更新后，刷新scroll的延时
      },
      startY: {
        type: Number,
        default: 0
      },
      refreshDelay: {
        type: Number,
        default: 20
      },
      freeScroll: {
        type: Boolean,
        default: false
      },
      topArea : {
        type: Number,
        default: 0,
      },
      bottomArea: {
        type: Number,
        default: 0,
      },
      isHasPull: {
        type: Boolean,
        default: false
      },
      scrollTab: {
        type: String,
        default: ""
      }
    },
    data() {
      return {
        beforePullDown: true,
        isRebounding: false,
        isPullingDown: false,
        isPullUpLoad: false,
        pullUpDirty: true,
        pullDownStyle: '',
        bubbleY: 0,
        rootFontSize: 0
      }
    },
    computed: {
      pullUpTxt() {
        const moreTxt = this.pullUpLoad && this.pullUpLoad.txt && this.pullUpLoad.txt.more;
        const noMoreTxt = this.pullUpLoad && this.pullUpLoad.txt && this.pullUpLoad.txt.noMore;
        return this.pullUpDirty ? moreTxt : noMoreTxt
      },
      refreshTxt() {
        return this.pullDownRefresh && this.pullDownRefresh.txt
      }
    },
    created() {
      this.pullDownInitTop = -80
    },
    mounted() {
      setTimeout(() => {
        this.initScroll()
      }, 20);
    },
    methods: {
      initScroll() {
        if (!this.$refs.wrapper) {
          return
        }
        this.rootFontSize = parseFloat(document.getElementsByTagName('html')[0].style.fontSize);
        if(this.topArea > 0){
          let topMargin = (this.topArea/this.rootFontSize).toFixed(4) + "rem";
          this.$refs.innerContent.style.paddingTop = topMargin;
        }
        if(this.bottomArea > 0){
          let bottomMargin = (this.bottomArea/this.rootFontSize).toFixed(4) + "rem";
          this.$refs.innerContent.style.paddingBottom = bottomMargin;
        }
        if (this.$refs.innerContent) {
            let scrollHeight = document.getElementsByClassName('scroll-component')[0].clientHeight;
            this.$refs.innerContent.style.minHeight = ((scrollHeight + 2)/this.rootFontSize).toFixed(4) + "rem";
        }
         // && (this.pullDownRefresh || this.pullUpLoad)
        let options = {
          probeType: this.probeType,
          click: this.click,
          scrollY: this.freeScroll || this.direction === DIRECTION_V,
          scrollX: this.freeScroll || this.direction === DIRECTION_H,
          scrollbar: this.scrollbar,
          pullDownRefresh: this.pullDownRefresh,
          pullUpLoad: this.pullUpLoad,
          startY: this.startY,
          freeScroll: this.freeScroll
        }
        // better-scroll的初始化
        this.scroll = new BScroll(this.$refs.wrapper, options)
        // 是否派发滚动事件
        if (this.listenScroll) {
          this.scroll.on('scroll', (pos) => {
            this.$emit('scroll', pos)
          })
        }
        if (this.listenBeforeScroll) {
          this.scroll.on('beforeScrollStart', () => {
            this.$emit('beforeScrollStart')
          })
        }
        if (this.pullDownRefresh) {
          this._initPullDownRefresh()
        }
        if (this.pullUpLoad) {
          this._initPullUpLoad()
        }
        if(this.scrollTab) {
          this._initTabScroll()
        }
      },
      disable() {
        this.scroll && this.scroll.disable()
      },
      enable() {
        this.scroll && this.scroll.enable()
      },
      refresh() {
        this.scroll && this.scroll.refresh()
      },
      scrollTo() {
        this.scroll && this.scroll.scrollTo.apply(this.scroll, arguments)
      },
      scrollToElement() {
        this.scroll && this.scroll.scrollToElement.apply(this.scroll, arguments)
      },
      clickItem(e, item) {
        console.log(e)
        this.$emit('click', item)
      },
      destroy() {
        this.scroll.destroy()
      },
      forceUpdate(dirty) {
        if (this.pullDownRefresh && this.isPullingDown) {
          this.isPullingDown = false
          this._reboundPullDown().then(() => {
            this._afterPullDown()
          })
        } else if (this.pullUpLoad && this.isPullUpLoad) {
          this.isPullUpLoad = false
          this.scroll.finishPullUp()
          this.pullUpDirty = dirty
          this.refresh()
        } else {
          this.refresh()
        }
      },
      _initPullDownRefresh() {
        this.scroll.on('pullingDown', () => {
          this.beforePullDown = false
          this.isPullingDown = true
          this.$emit('pullingDown')
        })
        this.scroll.on('scroll', (pos) => {
          if (this.beforePullDown) {
            this.bubbleY = Math.max(0, pos.y + this.pullDownInitTop)
            this.pullDownStyle = `top:${Math.min(pos.y + this.topArea + this.pullDownInitTop , this.topArea)}px`;
          } else {
            this.bubbleY = 0
          }
          if (this.isRebounding) {
            this.pullDownStyle = `top:${(this.pullDownRefresh.stop - pos.y + this.topArea)}px`
          }
        })
      },
      _initPullUpLoad() {
        this.scroll.on('pullingUp', () => {
          this.isPullUpLoad = true;
          this.$emit('pullingUp')
        })
      },
      _reboundPullDown() {
        const stopTime = 600;
        return new Promise((resolve) => {
          setTimeout(() => {
            this.isRebounding = true
            this.scroll.finishPullDown()
            resolve()
          }, stopTime)
        })
      },
      _afterPullDown() {
        setTimeout(() => {
          this.pullDownStyle = `top:${this.pullDownInitTop + this.topArea}px`
          this.beforePullDown = true
          this.isRebounding = false
          this.refresh()
        }, this.scroll.options.bounceTime)
      },
      _initTabScroll() {
        this.scroll.on('scroll', (pos) => {
          let tabTop = document.getElementById('scrollTab').offsetTop + pos.y - this.topArea;
          if( tabTop < 1000 && tabTop > -1000 ){
            if(tabTop > 0){
              this.$emit('hideTab');
            }else {
              this.$emit("showTab")
            }
          }
        })
      }
    },
    watch: {
      data() {
        setTimeout(() => {
          this.forceUpdate(true)
        }, this.refreshDelay)
      }
    },
    components: {
      Loading,
      Bubble
    }
  }
</script>
<style lang="scss" scoped>
  .list-wrapper{
    position: absolute;
    left: 0;
    top: 0;
    right: 0;
    bottom: 0;
    overflow: hidden;
    box-sizing: border-box;
    & .scroll-content {
      box-sizing: border-box;
      position: relative;
      z-index: 1;
    }
    & .list-content {
      position: relative;
      z-index: 10;
      background: #fff;
      & .list-item {
        height: 60px;
        line-height: 60px;
        font-size: 18px;
        padding-left: 20px;
        border-bottom: 1px solid #e5e5e5; /*no*/
      }
    }
  }
  .pulldown-wrapper{
    position: absolute;
    width: 100%;
    left: 0;
    display: flex;
    justify-content: center;
    align-items: center;
    transition: all;
    & .after-trigger{
      margin-top: 10px;
    }
  }
  .pullup-wrapper{
    width: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
    padding: 16px 0;
  }
</style>
