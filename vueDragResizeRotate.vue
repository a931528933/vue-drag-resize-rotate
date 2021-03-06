<!-- based on kirillmurashov vue-drag-resize
     my version adds rotate function
     the rotate process runs really smooooooth
     下面简述一下旋转原理：
     首先在组件中添加一个handler，初始在该组件的上端
     当点击该handler时在rotatestart数组中保存当前鼠标指针x,y坐标
     rotatecenter保存当前组件中心点x,y坐标
     rotate end 即为当前鼠标指针x,y坐标
     当开始旋转时，rotatestart = 上一次的rotate end
     意即为每次旋转时都会有一个锐角极小的三角形，通过余弦定理计算该小角，该角即为要增加或是减少的角度值

     那么怎么判断是要加还是减少角度值呢？
     需要通过矢量的叉乘
     假设有三角形ABC
     A为旋转中心，B为起始点，C为终止点
     那么向量AB可以获取到，向量AC可以获取到
     设A(x1,y1) B(x2,y2) C(x3,y3)
     那么AB=(x2-x1,y2-y1)
     AC=(x3-x1,y3-y1)
     AB AC叉积=行列式
      |x2-x1,y2-y1|
      |x3-x1,y3-y1|
      计算可得一值
      根据右手法则
      如果该值>0,则为逆时针
      如果该值<0,则为顺时针
      如果该值=0,则未旋转，当然此时旋转也deg = 0，无需考虑
 -->

<template>
  <div
    class="vdr"
    :style="{ ...style, transform: 'rotateZ(' + deg + 'Deg)' }"
    ref="current"
    tabIndex="-1"
    @keyup.delete="delItem"
    @click="clickSon($event)"
    :class="active || isActive ? 'active' : 'inactive'"
    @mousedown.stop.prevent="bodyDown($event)"
    @touchstart.stop.prevent="bodyDown($event)"
  >
    <slot></slot>
    <div
      v-for="(stick, key) in sticks"
      :key="key"
      class="vdr-stick"
      :class="['vdr-stick-' + stick, isResizable ? '' : 'not-resizable']"
      @mousedown.stop.prevent="stickDown(stick, $event)"
      @touchstart.stop.prevent="stickDown(stick, $event)"
      :style="vdrStick(stick)"
    ></div>
    <span
      class="el-icon-refresh-right rotate"
      :style="currentHalfWidth"
      v-show="rotate && active"
      @mousedown.stop.prevent="activeRotate($event)"
    ></span>
  </div>
</template>
<script>
const stickSize = 8
const styleMapping = {
  y: {
    t: 'top',
    m: 'marginTop',
    b: 'bottom'
  },
  x: {
    l: 'left',
    m: 'marginLeft',
    r: 'right'
  }
}
export default {
  name: 'vue-drag-resize-rotate',
  props: {
    rotate: {
      type: Boolean,
      default: true
    },
    border: {},
    bgColor: {
      default: '#000'
    },
    deg: {
      type: Number,
      default: 0
    },
    item: {},
    parentScaleX: {
      type: Number,
      default: 1
    },
    parentScaleY: {
      type: Number,
      default: 1
    },
    isActive: {
      type: Boolean,
      default: false
    },
    preventActiveBehavior: {
      type: Boolean,
      default: false
    },
    isDraggable: {
      type: Boolean,
      default: true
    },
    isResizable: {
      type: Boolean,
      default: true
    },
    aspectRatio: {
      type: Boolean,
      default: false
    },
    parentLimitation: {
      type: Boolean,
      default: false
    },
    parentW: {
      type: Number,
      default: 0,
      validator: function(val) {
        return val >= 0
      }
    },
    parentH: {
      type: Number,
      default: 0,
      validator: function(val) {
        return val >= 0
      }
    },
    w: {
      type: Number,
      default: 100,
      validator: function(val) {
        return val > 0
      }
    },
    h: {
      type: Number,
      default: 30,
      validator: function(val) {
        return val > 0
      }
    },
    minw: {
      type: Number,
      default: 50,
      validator: function(val) {
        return val > 0
      }
    },
    minh: {
      type: Number,
      default: 10,
      validator: function(val) {
        return val > 0
      }
    },
    x: {
      type: Number,
      default: 0,
      validator: function(val) {
        return typeof val === 'number'
      }
    },
    y: {
      type: Number,
      default: 0,
      validator: function(val) {
        return typeof val === 'number'
      }
    },
    z: {
      type: [String, Number],
      default: 'auto',
      validator: function(val) {
        let valid = typeof val === 'string' ? val === 'auto' : val >= 0
        return valid
      }
    },
    dragHandle: {
      type: String,
      default: null
    },
    dragCancel: {
      type: String,
      default: null
    },
    sticks: {
      type: Array,
      default: function() {
        return ['tl', 'tm', 'tr', 'mr', 'br', 'bm', 'bl', 'ml']
      }
    },
    axis: {
      type: String,
      default: 'both',
      validator: function(val) {
        return ['x', 'y', 'both', 'none'].indexOf(val) !== -1
      }
    },
    type: {}
  },
  data: function() {
    return {
      rawDeg: 0,
      rotateCenter: [],
      rotateEnd: [],
      rotateStart: [],
      active: this.isActive,
      rawWidth: this.w,
      rawHeight: this.h,
      rawLeft: this.x,
      rawTop: this.y,
      rawRight: null,
      rawBottom: null,
      zIndex: this.z,
      aspectFactor: this.w / this.h,
      parentWidth: null,
      parentHeight: null,
      left: this.x,
      top: this.y,
      right: null,
      bottom: null,
      minWidth: this.minw,
      minHeight: this.minh
    }
  },
  created: function() {
    this.stickDrag = false
    this.bodyDrag = false
    this.stickAxis = null
    this.stickStartPos = { mouseX: 0, mouseY: 0, x: 0, y: 0, w: 0, h: 0 }
    this.limits = {
      minLeft: null,
      maxLeft: null,
      minRight: null,
      maxRight: null,
      minTop: null,
      maxTop: null,
      minBottom: null,
      maxBottom: null
    }
    this.currentStick = []
  },
  mounted: function() {
    this.cacuFather()
    document.documentElement.addEventListener('mousemove', this.move)
    document.documentElement.addEventListener('mouseup', this.up)
    document.documentElement.addEventListener('mouseleave', this.up)
    document.documentElement.addEventListener('mousedown', this.deselect)
    document.documentElement.addEventListener('touchmove', this.move, true)
    document.documentElement.addEventListener('touchend touchcancel', this.up, true)
    document.documentElement.addEventListener('touchstart', this.up, true)
    this.getCenter()
    if (this.dragHandle) {
      let dragHandles = Array.prototype.slice.call(this.$el.querySelectorAll(this.dragHandle))
      for (let i in dragHandles) {
        dragHandles[i].setAttribute('data-drag-handle', this._uid)
      }
    }
    if (this.dragCancel) {
      let cancelHandles = Array.prototype.slice.call(this.$el.querySelectorAll(this.dragCancel))
      for (let i in cancelHandles) {
        cancelHandles[i].setAttribute('data-drag-cancel', this._uid)
      }
    }
  },
  beforeDestroy: function() {
    document.documentElement.removeEventListener('mousemove', this.move)
    document.documentElement.removeEventListener('mouseup', this.up)
    document.documentElement.removeEventListener('mouseleave', this.up)
    document.documentElement.removeEventListener('mousedown', this.deselect)
    document.documentElement.removeEventListener('touchmove', this.move, true)
    document.documentElement.removeEventListener('touchend touchcancel', this.up, true)
    document.documentElement.removeEventListener('touchstart', this.up, true)
  },
  methods: {
    debounce(method, context, params) {
      clearTimeout(method.timeout)
      method.timeout = setTimeout(function() {
        method.call(context, params)
      }, 300)
    },
    getCenter() {
      var el = this.$refs.current
      this.rotateCenter = [
        el.getBoundingClientRect().left + el.getBoundingClientRect().width / 2,
        el.getBoundingClientRect().top + el.getBoundingClientRect().height / 2
      ]
    },
    activeRotate(e) {
      this.getCenter()
      this.rotateStart = [e.clientX, e.clientY]
      const listener = event => {
        var el = this.$refs.current
        let a = this.calculLength(this.rotateStart[0], event.clientX, this.rotateStart[1], event.clientY)
        let c = this.calculLength(this.rotateStart[0], this.rotateCenter[0], this.rotateStart[1], this.rotateCenter[1])
        let b = this.calculLength(this.rotateCenter[0], event.clientX, this.rotateCenter[1], event.clientY)
        // eslint-disable-next-line prettier/prettier
        let direct = this.calculClock(this.rotateCenter[0], this.rotateCenter[1], this.rotateStart[0], this.rotateStart[1], event.clientX, event.clientY) >= 0
        let rawDeg = this.calculrawDegA(a, b, c)
        rawDeg = Math.abs(rawDeg)
        //判断转向 顺时针or 逆时针
        if (!direct) {
          rawDeg = 0 - rawDeg
        }
        var srawDeg = this.rawDeg
        srawDeg += rawDeg
        Math.abs(srawDeg) > 360 ? (srawDeg %= 360) : true
        this.rawDeg = srawDeg
        this.rotateStart[0] = event.clientX
        this.rotateStart[1] = event.clientY
      }
      document.body.addEventListener('mousemove', listener)
      document.body.addEventListener('mouseup', () => {
        document.body.removeEventListener('mousemove', listener)
      })
    },
    getAngle(cen, first, second) {
      console.log(hDis / h, wDis / w)
    },
    // delItem() {
    //   this.$emit('delreport', this.customId)
    // },
    clickSon(e) {
      //console.log(e.target);
      e.target.focus()
      e.stopPropagation()
    },
    cacuFather() {
      this.parentElement = this.$el.parentNode
      this.parentWidth = this.parentW ? this.parentW : this.parentElement.clientWidth
      this.parentHeight = this.parentH ? this.parentH : this.parentElement.clientHeight
      this.rawRight = this.parentWidth - this.rawWidth - this.rawLeft
      this.rawBottom = this.parentHeight - this.rawHeight - this.rawTop
    },
    deselect() {
      if (this.preventActiveBehavior) {
        return
      }
      this.active = false
    },
    move(ev) {
      if (!this.stickDrag && !this.bodyDrag) {
        return
      }
      ev.stopPropagation()
      if (this.stickDrag) {
        this.stickMove(ev)
      }
      if (this.bodyDrag) {
        this.bodyMove(ev)
      }
    },
    up(ev) {
      if (this.stickDrag) {
        this.stickUp(ev)
      }
      if (this.bodyDrag) {
        this.bodyUp(ev)
      }
    },
    bodyDown: function(ev) {
      let target = ev.target || ev.srcElement
      if (!this.preventActiveBehavior) {
        this.active = true
      }
      if (ev.button && ev.button !== 0) {
        return
      }
      this.$emit('clicked', ev)
      if (!this.isDraggable || !this.active) {
        return
      }
      if (this.dragHandle && target.getAttribute('data-drag-handle') !== this._uid.toString()) {
        return
      }
      if (this.dragCancel && target.getAttribute('data-drag-cancel') === this._uid.toString()) {
        return
      }
      this.bodyDrag = true
      this.stickStartPos.mouseX = ev.pageX || ev.touches[0].pageX
      this.stickStartPos.mouseY = ev.pageY || ev.touches[0].pageY
      this.stickStartPos.left = this.left
      this.stickStartPos.right = this.right
      this.stickStartPos.top = this.top
      this.stickStartPos.bottom = this.bottom
      if (this.parentLimitation) {
        this.limits = this.calcDragLimitation()
      }
    },
    calcDragLimitation() {
      const parentWidth = this.parentWidth
      const parentHeight = this.parentHeight
      return {
        minLeft: 0,
        maxLeft: parentWidth - this.width,
        minRight: 0,
        maxRight: parentWidth - this.width,
        minTop: 0,
        maxTop: parentHeight - this.height,
        minBottom: 0,
        maxBottom: parentHeight - this.height
      }
    },
    bodyMove(ev) {
      const stickStartPos = this.stickStartPos
      let delta = {
        x:
          (this.axis !== 'y' && this.axis !== 'none' ? stickStartPos.mouseX - (ev.pageX || ev.touches[0].pageX) : 0) /
          this.parentScaleX,
        y:
          (this.axis !== 'x' && this.axis !== 'none' ? stickStartPos.mouseY - (ev.pageY || ev.touches[0].pageY) : 0) /
          this.parentScaleY
      }
      this.rawTop = stickStartPos.top - delta.y
      this.rawBottom = stickStartPos.bottom + delta.y
      this.rawLeft = stickStartPos.left - delta.x
      this.rawRight = stickStartPos.right + delta.x
      this.$emit('dragging', this.rect)
    },
    bodyUp() {
      this.bodyDrag = false
      this.$emit('dragging', this.rect)
      this.$emit('dragstop', this.rect)
      this.stickStartPos = { mouseX: 0, mouseY: 0, x: 0, y: 0, w: 0, h: 0 }
      this.limits = {
        minLeft: null,
        maxLeft: null,
        minRight: null,
        maxRight: null,
        minTop: null,
        maxTop: null,
        minBottom: null,
        maxBottom: null
      }
    },
    stickDown: function(stick, ev) {
      if (!this.isResizable || !this.active) {
        return
      }
      this.stickDrag = true
      this.stickStartPos.mouseX = ev.pageX || ev.touches[0].pageX
      this.stickStartPos.mouseY = ev.pageY || ev.touches[0].pageY
      this.stickStartPos.left = this.left
      this.stickStartPos.right = this.right
      this.stickStartPos.top = this.top
      this.stickStartPos.bottom = this.bottom
      this.currentStick = stick.split('')
      this.stickAxis = null
      switch (this.currentStick[0]) {
        case 'b':
          this.stickAxis = 'y'
          break
        case 't':
          this.stickAxis = 'y'
          break
      }
      switch (this.currentStick[1]) {
        case 'r':
          this.stickAxis = this.stickAxis === 'y' ? 'xy' : 'x'
          break
        case 'l':
          this.stickAxis = this.stickAxis === 'y' ? 'xy' : 'x'
          break
      }
      this.limits = this.calcResizeLimitation()
    },
    calcResizeLimitation() {
      let minw = this.minWidth
      let minh = this.minHeight
      const aspectFactor = this.aspectFactor
      const width = this.width
      const height = this.height
      const bottom = this.bottom
      const top = this.top
      const left = this.left
      const right = this.right
      const stickAxis = this.stickAxis
      const parentLim = this.parentLimitation ? 0 : null
      if (this.aspectRatio) {
        if (minw / minh > aspectFactor) {
          minh = minw / aspectFactor
        } else {
          minw = aspectFactor * minh
        }
      }
      let limits = {
        minLeft: parentLim,
        maxLeft: left + (width - minw),
        minRight: parentLim,
        maxRight: right + (width - minw),
        minTop: parentLim,
        maxTop: top + (height - minh),
        minBottom: parentLim,
        maxBottom: bottom + (height - minh)
      }
      if (this.aspectRatio) {
        const aspectLimits = {
          minLeft: left - Math.min(top, bottom) * aspectFactor * 2,
          maxLeft: left + ((height - minh) / 2) * aspectFactor * 2,
          minRight: right - Math.min(top, bottom) * aspectFactor * 2,
          maxRight: right + ((height - minh) / 2) * aspectFactor * 2,
          minTop: top - (Math.min(left, right) / aspectFactor) * 2,
          maxTop: top + ((width - minw) / 2 / aspectFactor) * 2,
          minBottom: bottom - (Math.min(left, right) / aspectFactor) * 2,
          maxBottom: bottom + ((width - minw) / 2 / aspectFactor) * 2
        }
        if (stickAxis === 'x') {
          limits = {
            minLeft: Math.max(limits.minLeft, aspectLimits.minLeft),
            maxLeft: Math.min(limits.maxLeft, aspectLimits.maxLeft),
            minRight: Math.max(limits.minRight, aspectLimits.minRight),
            maxRight: Math.min(limits.maxRight, aspectLimits.maxRight)
          }
        } else if (stickAxis === 'y') {
          limits = {
            minTop: Math.max(limits.minTop, aspectLimits.minTop),
            maxTop: Math.min(limits.maxTop, aspectLimits.maxTop),
            minBottom: Math.max(limits.minBottom, aspectLimits.minBottom),
            maxBottom: Math.min(limits.maxBottom, aspectLimits.maxBottom)
          }
        }
      }
      return limits
    },
    stickMove(ev) {
      const stickStartPos = this.stickStartPos
      const delta = {
        x: (stickStartPos.mouseX - (ev.pageX || ev.touches[0].pageX)) / this.parentScaleX,
        y: (stickStartPos.mouseY - (ev.pageY || ev.touches[0].pageY)) / this.parentScaleY
      }
      switch (this.currentStick[0]) {
        case 'b':
          this.rawBottom = stickStartPos.bottom + delta.y
          break
        case 't':
          this.rawTop = stickStartPos.top - delta.y
          break
      }
      switch (this.currentStick[1]) {
        case 'r':
          this.rawRight = stickStartPos.right + delta.x
          break
        case 'l':
          this.rawLeft = stickStartPos.left - delta.x
          break
      }
      this.$emit('resizing', this.rect)
    },
    stickUp() {
      this.stickDrag = false
      this.stickStartPos = {
        mouseX: 0,
        mouseY: 0,
        x: 0,
        y: 0,
        w: 0,
        h: 0
      }
      this.limits = {
        minLeft: null,
        maxLeft: null,
        minRight: null,
        maxRight: null,
        minTop: null,
        maxTop: null,
        minBottom: null,
        maxBottom: null
      }
      this.rawTop = this.top
      this.rawBottom = this.bottom
      this.rawLeft = this.left
      this.rawRight = this.right
      this.stickAxis = null
      this.$emit('resizing', this.rect)
      this.$emit('resizestop', this.rect)
    },
    aspectRatioCorrection() {
      if (!this.aspectRatio) {
        return
      }
      const bottom = this.bottom
      const top = this.top
      const left = this.left
      const right = this.right
      const width = this.width
      const height = this.height
      const aspectFactor = this.aspectFactor
      const currentStick = this.currentStick
      if (width / height > aspectFactor) {
        let newWidth = aspectFactor * height
        if (currentStick[1] === 'l') {
          this.left = left + width - newWidth
        } else {
          this.right = right + width - newWidth
        }
      } else {
        let newHeight = width / aspectFactor
        if (currentStick[0] === 't') {
          this.top = top + height - newHeight
        } else {
          this.bottom = bottom + height - newHeight
        }
      }
    },
    //朴素的两点间距离公式
    calculLength(x1, x2, y1, y2) {
      return Math.sqrt(Math.pow(x1 - x2, 2) + Math.pow(y1 - y2, 2))
    },
    //余弦定理求角度
    calculrawDegA(a, b, c) {
      return Math.acos((b * b + c * c - a * a) / (2 * b * c)) * (180 / Math.PI)
    },
    //六个点坐标，通过矢量叉积获取旋转方向
    calculClock(x1, y1, x2, y2, x3, y3) {
      return (x2 - x1) * (y3 - y1) - (y2 - y1) * (x3 - x1)
    }
  },
  computed: {
    currentHalfWidth() {
      return {
        'font-size': '40px',
        position: 'absolute',
        top: '-50px',
        cursor: 'pointer',
        right: this.w / 2 - 20 + 'px',
        color: '#888888'
      }
    },
    style() {
      return {
        top: this.top + 'px',
        left: this.left + 'px',
        width: this.width + 'px',
        height: this.height + 'px',
        zIndex: this.zIndex
      }
    },
    vdrStick() {
      return stick => {
        const stickStyle = {
          width: `${stickSize / this.parentScaleX}px`,
          height: `${stickSize / this.parentScaleY}px`
        }
        stickStyle[styleMapping.y[stick[0]]] = `${stickSize / this.parentScaleX / -2}px`
        stickStyle[styleMapping.x[stick[1]]] = `${stickSize / this.parentScaleX / -2}px`
        return stickStyle
      }
    },
    width() {
      return this.parentWidth - this.left - this.right
    },
    height() {
      return this.parentHeight - this.top - this.bottom
    },
    rect() {
      return {
        left: Math.round(this.left),
        top: Math.round(this.top),
        width: Math.round(this.width),
        height: Math.round(this.height)
      }
    }
  },
  watch: {
    rawDeg() {
      this.$emit('rotate', this.rawDeg)
    },
    rawLeft(newLeft) {
      const limits = this.limits
      const stickAxis = this.stickAxis
      const aspectFactor = this.aspectFactor
      const aspectRatio = this.aspectRatio
      const left = this.left
      const bottom = this.bottom
      const top = this.top
      if (limits.minLeft !== null && newLeft < limits.minLeft) {
        newLeft = limits.minLeft
      } else if (limits.maxLeft !== null && limits.maxLeft < newLeft) {
        newLeft = limits.maxLeft
      }
      if (aspectRatio && stickAxis === 'x') {
        const delta = left - newLeft
        this.rawTop = top - delta / aspectFactor / 2
        this.rawBottom = bottom - delta / aspectFactor / 2
      }
      this.left = newLeft
    },
    rawRight(newRight) {
      const limits = this.limits
      const stickAxis = this.stickAxis
      const aspectFactor = this.aspectFactor
      const aspectRatio = this.aspectRatio
      const right = this.right
      const bottom = this.bottom
      const top = this.top
      if (limits.minRight !== null && newRight < limits.minRight) {
        newRight = limits.minRight
      } else if (limits.maxRight !== null && limits.maxRight < newRight) {
        newRight = limits.maxRight
      }
      if (aspectRatio && stickAxis === 'x') {
        const delta = right - newRight
        this.rawTop = top - delta / aspectFactor / 2
        this.rawBottom = bottom - delta / aspectFactor / 2
      }
      this.right = newRight
    },
    rawTop(newTop) {
      const limits = this.limits
      const stickAxis = this.stickAxis
      const aspectFactor = this.aspectFactor
      const aspectRatio = this.aspectRatio
      const right = this.right
      const left = this.left
      const top = this.top
      if (limits.minTop !== null && newTop < limits.minTop) {
        newTop = limits.minTop
      } else if (limits.maxTop !== null && limits.maxTop < newTop) {
        newTop = limits.maxTop
      }
      if (aspectRatio && stickAxis === 'y') {
        const delta = top - newTop
        this.rawLeft = left - (delta * aspectFactor) / 2
        this.rawRight = right - (delta * aspectFactor) / 2
      }
      this.top = newTop
    },
    rawBottom(newBottom) {
      const limits = this.limits
      const stickAxis = this.stickAxis
      const aspectFactor = this.aspectFactor
      const aspectRatio = this.aspectRatio
      const right = this.right
      const left = this.left
      const bottom = this.bottom
      if (limits.minBottom !== null && newBottom < limits.minBottom) {
        newBottom = limits.minBottom
      } else if (limits.maxBottom !== null && limits.maxBottom < newBottom) {
        newBottom = limits.maxBottom
      }
      if (aspectRatio && stickAxis === 'y') {
        const delta = bottom - newBottom
        this.rawLeft = left - (delta * aspectFactor) / 2
        this.rawRight = right - (delta * aspectFactor) / 2
      }
      this.bottom = newBottom
    },
    width() {
      this.aspectRatioCorrection()
    },
    height() {
      this.aspectRatioCorrection()
    },
    active(isActive) {
      if (isActive) {
        this.$emit('activated')
      } else {
        this.$emit('deactivated')
      }
    },
    isActive(val) {
      this.active = val
    },
    z(val) {
      if (val >= 0 || val === 'auto') {
        this.zIndex = val
      }
    },
    aspectRatio(val) {
      if (val) {
        this.aspectFactor = this.width / this.height
      }
    },
    minw(val) {
      if (val > 0 && val <= this.width) {
        this.minWidth = val
      }
    },
    minh(val) {
      if (val > 0 && val <= this.height) {
        this.minHeight = val
      }
    },
    x() {
      if (this.stickDrag || this.bodyDrag) {
        return
      }
      if (this.parentLimitation) {
        this.limits = this.calcDragLimitation()
      }
      let delta = this.x - this.left
      this.rawLeft = this.x
      this.rawRight = this.right - delta
    },
    y() {
      if (this.stickDrag || this.bodyDrag) {
        return
      }
      if (this.parentLimitation) {
        this.limits = this.calcDragLimitation()
      }
      let delta = this.y - this.top
      this.rawTop = this.y
      this.rawBottom = this.bottom - delta
    },
    w() {
      if (this.stickDrag || this.bodyDrag) {
        return
      }
      this.currentStick = ['m', 'r']
      this.stickAxis = 'x'
      if (this.parentLimitation) {
        this.limits = this.calcResizeLimitation()
      }
      let delta = this.width - this.w
      this.rawRight = this.right + delta
    },
    h() {
      if (this.stickDrag || this.bodyDrag) {
        return
      }
      this.currentStick = ['b', 'm']
      this.stickAxis = 'y'
      if (this.parentLimitation) {
        this.limits = this.calcResizeLimitation()
      }
      let delta = this.height - this.h
      this.rawBottom = this.bottom + delta
    },
    parentW(val) {
      this.right = val - this.width - this.left
      this.parentWidth = val
    },
    parentH(val) {
      this.bottom = val - this.height - this.top
      this.parentHeight = val
    }
  }
}
</script>
<style>
.vdr {
  position: absolute;
  box-sizing: border-box;
}
span.rotate {
  font-size: 20px;
  position: absolute;
  top: -30px;
  cursor: pointer;
  right: -30px;
  color: #000000;
}
span.rotate:hover {
  color: #28f0ff;
}
.vdr.active:before {
  content: '';
  width: 100%;
  height: 100%;
  position: absolute;
  top: -3px;
  left: -3px;
  box-sizing: content-box;
  outline: 6px solid transparent;
  border: 2px solid #022d30;
}
.vdr-stick {
  box-sizing: border-box;
  position: absolute;
  font-size: 1px;
  background: #ffffff;
  border: 1px solid #6c6c6c;
  box-shadow: 0 0 2px #bbb;
}
.inactive .vdr-stick {
  display: none;
}
.vdr-stick-tl,
.vdr-stick-br {
  cursor: nwse-resize;
}
.vdr-stick-tm,
.vdr-stick-bm {
  left: 50%;
  cursor: ns-resize;
}
.vdr-stick-tr,
.vdr-stick-bl {
  cursor: nesw-resize;
}
.vdr-stick-ml,
.vdr-stick-mr {
  top: 50%;
  cursor: ew-resize;
}
.vdr-stick.not-resizable {
  display: none;
}
.vdr.active:before {
  cursor: move;
  /*显示移动光标*/
}
</style>
