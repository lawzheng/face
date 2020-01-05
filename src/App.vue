<template>
  <div id="app">
    <img alt="Vue logo" :src="yaoEr" width="400" id="img">
    <canvas id="canvas"/>
  </div>
</template>

<script>
  import * as faceapi from 'face-api.js'
  import hat from './assets/images/hat.png'
  import huzi from './assets/images/huzi.png'
  /**
   * 获取中间的点
   * @param {*} points
   */
  const getMedian = points => points[Math.floor(points.length / 2)]

  /**
   * 获取两点之间的中点
   * @param {*} pa
   * @param {*} pb
   */
  const getMidPoint = (pa, pb) => ({
    x: (pa.x + pb.x) / 2,
    y: (pa.y + pb.y) / 2
  })

  /**
   * 获取两点之间距离
   * @param {*} a
   * @param {*} b
   */
  const getDistance = (a, b) => Math.sqrt(Math.pow(a.x - b.x, 2) + Math.pow(a.y - b.y, 2))

  /**
   * 获取两个眉毛中点
   * @param {*} leftPoints
   * @param {*} rightPoints
   */
  const getMidPointOfEye = (leftPoints, rightPoints) =>
    getMidPoint(getMedian(leftPoints), getMedian(rightPoints))

  /**
   * 获取下颌的最低点
   * @param {*} jawPoints
   */
  const getJawPos = jawPoints => getMedian(jawPoints)

  /**
   * 获取脸的长度 （按照三停五眼）
   * @param {*} jawPos
   * @param {*} midPointOfEyebrows
   */
  const getFaceLength = (jawPos, midPointOfEyebrows) => (5 * getDistance(jawPos, midPointOfEyebrows)) / 3

  /**
   * 获取脸的宽度（即帽子宽度）
   * @param {*} outlinePoints
   */
  const getFaceWith = outlinePoints => getDistance(outlinePoints[0], outlinePoints[outlinePoints.length - 1])

  /**
   * 获取脸的倾斜弧度
   * @param {*} jawPos
   * @param {*} midPointOfEyebrows
   */
  const getFaceRadian = (jawPos, midPointOfEyebrows) =>
    Math.PI - Math.atan2(jawPos.x - midPointOfEyebrows.x, jawPos.y - midPointOfEyebrows.y) // 弧度  0.9272952180016122

  // 计算帽子的位置, 眉心和右上角顶点的中点（考虑到图片绘制是从左上角开始绘制，还需要根据图片中心做个变换）
  // 知道眉心坐标（x1,y1) 知道下颌的坐标(x2, y2)，知道脸宽w，知道脸长l
  /**
   * 已知K，d, 点，求另一个点
   * @param {*} k
   * @param {*} d
   * @param {*} point
   */
  const getPos = (k, d, point) => {
    // 取y变小的那一边
    let y = -Math.sqrt((d * d) / (1 + k * k)) + point.y
    let x = k * (y - point.y) + point.x
    return { x, y }
  }

  /**
   * 获取头顶的坐标
   * @param {*} midPos 眉心点坐标
   * @param {*} jawPos 下巴底点坐标
   */
  const getHeadPos = (midPos, jawPos) => {
    // 获取线的k值
    const k = getK(midPos, jawPos)
    // 获取眉心到下颌的距离
    const distanceOfEye2Jaw = getDistance(midPos, jawPos)
    return getPos(k, distanceOfEye2Jaw / 2, midPos)
  }

  /**
   * 获取K值
   * @param {*} a
   * @param {*} b
   */
  const getK = (a, b) => (a.x - b.x) / (a.y - b.y)
  function getMouthPos (mouthPoints) {
    console.log(mouthPoints)
    return mouthPoints[Math.floor(mouthPoints.length / 2) - 1]
  }
  function getHatInfo (results) {
    function getFaceInfo (leftEyeBrowPoints, rightEyeBrowPoints, outlinePoints, mouthPoints) {
      // 获取眉心的点
      const midPointOfEyebrows = getMidPointOfEye(leftEyeBrowPoints, rightEyeBrowPoints)
      // 获取下颌的点
      const jawPos = getJawPos(outlinePoints)
      // 获取脸的倾斜角度
      const angle = getFaceRadian(midPointOfEyebrows, jawPos)
      // 获取头顶的坐标
      const headPos = getHeadPos(midPointOfEyebrows, jawPos)
      // 获取脸大小信息
      const faceLength = getFaceLength(jawPos, midPointOfEyebrows)
      const faceWidth = getFaceWith(outlinePoints)
      const mouthPos = getMouthPos(mouthPoints)
      console.log(mouthPos)
      return {
        midPointOfEyebrows,
        jawPos,
        headPos,
        angle,
        faceWidth,
        faceLength,
        mouthPos
      }
    }
    return results.map(({ landmarks }) => {
      const rightEyeBrowPoints = landmarks.getRightEyeBrow()
      const leftEyeBrowPoints = landmarks.getLeftEyeBrow()
      const outlinePoints = landmarks.getJawOutline()
      const mouthPoints = landmarks.getMouth() // 嘴巴
      return getFaceInfo(leftEyeBrowPoints, rightEyeBrowPoints, outlinePoints, mouthPoints)
    })
  }
  /**
   * 根据我当前的圣诞帽元素进行一些偏移(我的图片大小是200*130)， 圣诞帽可佩戴部分的中心 (62,60)  这里需要微调
   * 图片可佩戴部分是 0.6 倍图片宽度
   * @param {*} x
   * @param {*} y
   */
  const translateHat = (faceWidth, x, y) => {
    const picSize = { width: faceWidth / 0.6, height: (faceWidth * 0.65) / 0.6 }
    return {
      ...picSize,
      x: x - (62 * picSize.width) / 200,
      y: y - (60 * picSize.height) / 130
    }
  }
  const translateHuZi = (faceWidth, x, y) => {
    const picSize = { width: faceWidth * 0.85, height: faceWidth * 0.35 }
    return {
      ...picSize,
      x: x - (400 * picSize.width) / 861,
      y: y - (300 * picSize.height) / 957
    }
  }

  /**
   * 获取图片
   * @param {*} src 图片地址
   * @param {*} callback
   */
  function getImg (src, callback) {
    const img = new Image()
    img.setAttribute('crossOrigin', 'anonymous')
    img.src = src
    img.onload = () => callback(img)
  }
  /**
   * 绘制帽子
   * @param {*} ctx 画布实例
   * @param {{}} config 配置
   */
  function drawHat (ctx, config) {
    const { headPos, angle, faceWidth } = config
    getImg(hat, img => {
      // 保存画布
      ctx.save()
      // 画布原点移动到画帽子的地方
      ctx.translate(headPos.x, headPos.y)
      // 旋转画布到特定角度
      ctx.rotate(angle)
      // 偏移图片，使帽子中心刚好在原点
      const { x, y, width, height } = translateHat(faceWidth, 0, 0)
      // 我的圣诞帽子实际佩戴部分长度只有0.75倍整个图片长度
      ctx.drawImage(img, x, y, width, height)
      // 还原画布
      ctx.restore()
    })
  }
  /**
   * 绘制帽子
   * @param {*} ctx 画布实例
   * @param {{}} config 配置
   */
  function drawHuzi (ctx, config) {
    const { mouthPos, angle, faceWidth } = config
    getImg(huzi, img => {
      // 保存画布
      ctx.save()
      // 画布原点移动到画帽子的地方
      ctx.translate(mouthPos.x, mouthPos.y)
      // 旋转画布到特定角度
      ctx.rotate(angle)
      // 偏移图片，使帽子中心刚好在原点
      const { x, y, width, height } = translateHuZi(faceWidth, 0, 0)
      // 我的圣诞帽子实际佩戴部分长度只有0.75倍整个图片长度
      ctx.drawImage(img, x, y, width, height)
      // 还原画布
      ctx.restore()
    })
  }
  /**
   * 绘制主流程
   * @param {*} canvas
   * @param {*} options
   */
  function drawing (canvas, options) {
    const { info, width = 200, height = 200, imgSrc = 'images/default.jpg' } = options
    const ctx = canvas.getContext('2d')
    // 重置
    ctx.clearRect(0, 0, width, height)
    // 先把图片绘制上去
    getImg(imgSrc, img => ctx.drawImage(img, 0, 0, width, height))
    // 循环把帽子画到对应的点上
    setTimeout(() => {
      for (let i = 0, len = info.length; i < len; i++) {
        drawHat(ctx, info[i])
      }
    }, 100)
    // 循环把胡子画到对应的点上
    setTimeout(() => {
      for (let i = 0, len = info.length; i < len; i++) {
        drawHuzi(ctx, info[i])
      }
    }, 100)
  }

  export default {
    name: 'app',
    data () {
      return {
        yaoEr: require('@/assets/images/QQ截图20200105185155.png')
      }
    },
    created () {
      this.init()
    },
    methods: {
      async init () {
        // 初始化face-api 这里使用ssd moblile 模型加载
        await faceapi.nets.ssdMobilenetv1.load('/weights')
        // 加载Landmark模型 面部64点模型
        await faceapi.loadFaceLandmarkModel('/weights')
        let inputImg = document.getElementById('img')
        let canvas = document.getElementById('canvas')
        // 获取识别数据
        const results = await faceapi.detectAllFaces(inputImg).withFaceLandmarks()
        // 使canvas和图片大小一致
        faceapi.matchDimensions(canvas, inputImg)
        // 把识别数据根据 展示图片的大小 做转换
        const resizedResults = faceapi.resizeResults(results, inputImg)
        const info = getHatInfo(resizedResults)
        drawing(canvas, {
          info,
          imgSrc: inputImg.src,
          width: canvas.width,
          height: canvas.height
        })
        // setTimeout(() => {
        //   // 直接画出识别的特征点
        //   faceapi.draw.drawFaceLandmarks(canvas, resizedResults)
        //   // 画框
        //   // const drawOptions = {
        //   //   label: 'Hello I am a dog!',
        //   //   lineWidth: 2
        //   // }
        //   // const drawBox = new faceapi.draw.DrawBox(resizedResults[0].detection.box, drawOptions)
        //   // drawBox.draw(canvas)
        // }, 500)
      }
    }
  }
</script>

<style lang="less">
  #app {
    font-family: 'Avenir', Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    text-align: center;
    color: #2c3e50;
    margin-top: 60px;
  }
</style>
