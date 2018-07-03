<template>
  <div>
    <h1> Plot Canvas</h1>
    <div class="container">
      <div class="box">
        <canvas
          id='plotCanvas' ref='canvas' :width=width :height=height
          @mousedown="mouseDownCanvas" @mousemove="mouseMoveCanvas" @mouseout="mouseOutCanvas"
          @touchstart.prevent="touchStartCanvas" @touchmove.prevent="touchMoveCanvas" @touchend.prevent="touchEndCanvas"
        ></canvas>
      </div>

      <div class="box">
        <h3>点の数: {{ points.length }}</h3>
        <input class="slider" step="1" min="0" max="100" value="50" type="range">
        <h3>描画方法</h3>
        <div class="control">
          <label class="radio">
            <input type="radio" value="alongLine" v-model="drawMethod" checked>
            線に沿って点を打つ
          </label>
          <!--<label class="radio">-->
            <!--<input type="radio" name="draw-method">-->
            <!--範囲を決めて点を打つ-->
          <!--</label>-->
          <label class="radio">
            <input type="radio" value="manual" v-model="drawMethod">
            手動で点を打つ
          </label>
        </div>
        <div>
          <button :click="previous">戻る</button>
          <button :click="next">進む</button>
        </div>
        <h3>力: {{ force }}</h3>
        <h3>半径: {{ radius }}</h3>
        <h3>描画方法: {{ drawMethod }}</h3>
        <h3>履歴</h3>
        <ul>
          <li v-for="history in actionHistory">action: {{ history[0] }}</li>
        </ul>
      </div>
    </div>

    <div class="info">
      <h3>現在の座標: {{ cursor != null ? cursor : "canvas外" }}</h3>
      <ul>
        <li v-for="point in points">
          canvas座標系での座標: ({{ point[0] }} , {{ point[1] }})
        </li>
      </ul>
      <h3>呼び出されたイベント</h3>
        <ul>
          <li v-for="e in events">{{ e }}</li>
        </ul>
    </div>
  </div>
</template>

<script>
  // canvas操作

  // 座標変換
  // let convertCoordinate = (coord) => {
  //   let canvasX = (coord[0] - xMin) / (xMax - xMin) * width;
  //   let canvasY = (coord[1] - yMax) / (yMin - yMax) * height;
  //   return [canvasX, canvasY];
  // };

  let canvas = null;
  let ctx = null;

  let timer;
  let throttle = function (targetFunc, time) {
    let _time = time || 100;
    clearTimeout(timer);
    timer = setTimeout(function () {
      targetFunc();
    }, _time);
  }

  // canvasのイベントの座標を取得
  let getCoordinate = function (event) {
    let rect = event.target.getBoundingClientRect();
    let x = event.clientX - rect.left;
    let y = event.clientY - rect.top;
    return [x, y];
  };

  // canvasをタッチした場合の座標を取得
  let getTouchCoordinate = function (event) {
    let touch = event.touches[0];
    let rect = event.target.getBoundingClientRect();
    let x = touch.clientX - rect.left;
    let y = touch.clientY - rect.top;
    return [x, y];
  };

  // 点を描画する範囲の線を引くために座標を取得
  let getAuxiliaryCoordinate = function (startX, startY, endX, endY, width=10) {
    let diffX = endX - startX;
    let diffY = startY - endY;  // canvasの座標系ではy軸は下向き正

    let scaleFactor = width / Math.sqrt(diffX * diffX + diffY * diffY);

    let startAux1X = startX - diffY * scaleFactor;
    let startAux1Y = startY + diffX * scaleFactor;
    let startAux2X = startX + diffY * scaleFactor;
    let startAux2Y = startY - diffX * scaleFactor;

    let endAux1X = startAux1X + diffX;
    let endAux1Y = startAux1Y + diffY;
    let endAux2X = startAux2X + diffX;
    let endAux2Y = startAux2Y + diffY;

    return [[startAux1X, startAux1Y], [endAux1X, endAux1Y], [startAux2X, startAux2Y], [endAux2X, endAux2Y]];
  };

  // canvas上に線を描画
  let drawLine = function (startX, startY, endX, endY, lineWidth=3, strokeStyle='black') {
    ctx.beginPath();
    ctx.moveTo(startX, startY);
    ctx.lineTo(endX, endY);
    ctx.lineCap = "round";
    ctx.lineWidth = lineWidth;
    ctx.strokeStyle = strokeStyle;
    ctx.stroke();
  };

  // canvas上に点線を描画
  let drawDashLine = function (startX, startY, endX, endY, {lineWidth=3, strokeStyle='black', segments=[0.1, 0.1]}) {
    ctx.beginPath();
    ctx.setLineDash(segments);
    ctx.moveTo(startX, startY);
    ctx.lineTo(endX, endY);
    ctx.lineCap = "round";
    ctx.lineWidth = lineWidth;
    ctx.strokeStyle = strokeStyle;
    ctx.stroke();
  };
  
  // canvas上に丸を描画
  let drawCircle = function (centerX, centerY, radius, method='stroke', color='black') {
    ctx.beginPath();
    ctx.arc(centerX, centerY, radius, 0, 2*Math.PI, true);
    if (method === 'stroke') {
      ctx.strokeStyle = color;
      ctx.stroke();
    } else if (method === 'fill') {
      ctx.fillStyle = color;
      ctx.fill();
    }
  };

  let addPoints = function (points, actionHistory, pointsToAdd) {
    // points: list of point (list of [x, y])
    // this.pointsとactionHistoryに追加し, 描画する
    points = points.concat(pointsToAdd);
    actionHistory.push(['add', pointsToAdd]);
    return [points, actionHistory];

  };
  let deletePoints = function (points, actionHistory, pointsToDelete) {
    // points: list of point
    for (let pointToDelete of pointsToDelete) {
      points = points.filter(function (point) {
        return point !== pointToDelete;
      });
    }
    actionHistory.push(['delete', pointsToDelete]);
  };

  export default {
    name: 'PlotCanvas',
    props: {
      size: Array,  // (height, width)
      xRange: Array,  // x軸の値の範囲
      yRange: Array,  // y軸の値の範囲
      strokeWidth: Number,
    },
    computed: {
      height() {return this.size[0]},
      width() {return this.size[1]},
    },
    data() {
      return {
        points: [],
        actionHistory: [],  // list of [action_name, points]
        currentIndex: -1,
        candPoints: [],
        cursor: null,
        count: 0,
        events: [],
        force: 0,
        drawMethod: "alongLine",
        radius: 0,
      }
    },
    mounted() {
      canvas = this.$refs.canvas;
      ctx = canvas.getContext('2d');
    },
    methods: {
      previous() {
        if (this.currentIndex >= 0) {
          let targetAction = this.actionHistory[this.currentIndex]
        }
      },
      next() {
        let x = 1;
      },
      mouseDownCanvas(event) {
        // drawMethod === "alongLine のとき, this.cursorを現在の座標にセット
        // drawMethod === "manual" のとき, 点を描画

        let [x, y] = getCoordinate(event);

        if (this.drawMethod === "alongLine") {
          this.cursor = [x, y];
        } else if (this.drawMethod === "manual") {
          // addの処理
          let points = [[x, y]];
          [this.points, this.actionHistory] = addPoints(this.points, this.actionHistory, points);

          // 点を描画
          drawCircle(x, y, this.strokeWidth, 'fill', 'black');
        }

        this.events.push(event.type);
      },
      touchStartCanvas(event) {
        // drawMethod === "alongLine のとき, this.cursorを現在の座標にセット
        // drawMethod === "manual" のとき, 点を描画

        let [x, y] = getTouchCoordinate(event);

        if (this.drawMethod === "alongLine") {
          this.cursor = [x, y];
        }
        else if (this.drawMethod === "manual") {
          this.points.push([x, y]);
          ctx.beginPath();
          ctx.arc(x, y, this.strokeWidth, 0, 2*Math.PI, true);
          ctx.fill();
        }

        this.events.push(event.type);
      },
      mouseMoveCanvas(event) {
        // drawMethod === "alongLine のとき, ユーザーが引いた線を挟むような補助線を引き囲まれた領域を異なる色に変える
        // drawMethod === "manual" のとき, 何もしない

        let [x, y] = getCoordinate(event);

        if (this.drawMethod === "alongLine") {
          // 座標が変化していて, ドラッグ中の場合, 描画する
          if ((event.buttons === 1 || event.which === 1) && this.cursor !== [x, y]) {
            // if (this.cursor == null) {
            //   drawLine(x, y, x, y);
            // } else {
            //   drawLine(this.cursor[0], this.cursor[1], x, y);
            // }
            let [startAux1, endAux1, startAux2, endAux2] = getAuxiliaryCoordinate(this.cursor[0], this.cursor[1], x, y);
            console.log(startAux1);
            console.log(endAux1);
            drawLine(this.cursor[0], this.cursor[1], x, y);
            drawLine(this.cursor[0], this.cursor[1], startAux1, endAux1, 3, 'red');
            drawLine(this.cursor[0], this.cursor[1], startAux2, endAux2, 3, 'red');
          }
          this.cursor = [x, y];
        } else if (this.drawMethod === 'manuall' ) {
          // カーソルを移動させる
          if (this.cursor != null) {
            // 一つ前のcursorの位置をクリアする
            ctx.beginPath();
            ctx.globalCompositeOperation = 'destination-out';
            ctx.arc(this.cursor[0], this.cursor[1], this.strokeWidth * 10, 0, 2*Math.PI, true);
            ctx.stroke();
            ctx.globalCompositeOperation = 'source-out';
          }
          let [x, y] = getCoordinate(event);
          this.cursor = [x, y];

          // 大きな円を描画
          ctx.beginPath();
          ctx.arc(x, y, this.strokeWidth * 10, 0, 2*Math.PI, true);
          ctx.stroke();
        }

        if (! (this.events.length > 0 && this.events[this.events.length-1] === "mousemove")) {
          this.events.push(event.type);
        }
      },
      touchMoveCanvas(event) {
        // drawMethod === "alongLine のとき, ユーザーが引いた線を挟むような補助線を引き囲まれた領域を異なる色に変える
        // drawMethod === "manual" のとき, 何もしない

        // 圧力を取得してみる
        let touch = event.touches[0];
        this.force = touch.force;

        // 圧力の大きさを円で表現
        // ctx.beginPath();
        // ctx.globalCompositeOperation = 'destination-out';
        // ctx.arc(this.cursor[0], this.cursor[1], this.radius, 0, 2*Math.PI, true);
        // ctx.stroke();
        // ctx.globalCompositeOperation = 'source-out';
        this.radius = this.force * 100;
        ctx.arc(this.cursor[0], this.cursor[1], this.radius, 0, 2*Math.PI, true);
        ctx.stroke();

        let [x, y] = getTouchCoordinate(event);
        // if (this.cursor == null) {
        //   drawLine(x, y, x, y);
        // } else {
        //   drawLine(this.cursor[0], this.cursor[1], x, y);
        // }
        drawLine(this.cursor[0], this.cursor[1], x, y);
        this.cursor = [x, y];
        if (! (this.events.length > 0 && this.events[this.events.length-1] === "touchmove")) {
          this.events.push(event.type);
        }
      },
      mouseOutCanvas() {
        // drawMethod === "alongLine" のとき, 作成した領域内の候補の点をthis.pointsに加え, 現在の座標をクリアする
        // drawMethod === "manual" のとき, 現在の座標をクリアする
        if (this.drawMethod === "alongLine") {
          // this.candPointsを描画
          // this.points += this.candPoints;
          this.candPoints = [];
        } else if (this.drawMethod === "manual") {
          // ブラシの表示をクリア
          if (this.cursor != null) {
            // 一つ前のcursorの位置をクリアする
            ctx.beginPath();
            ctx.globalCompositeOperation = 'destination-out';
            ctx.arc(this.cursor[0], this.cursor[1], this.strokeWidth * 10, 0, 2*Math.PI, true);
            ctx.stroke();
            ctx.globalCompositeOperation = 'source-out';
          }
          
        }
        this.cursor = null;
        this.events.push(event.type);
      },
      touchEndCanvas() {
        // drawMethod === "alongLine" のとき, 作成した領域内の候補の点をthis.pointsに加え, 現在の座標をクリアする
        // drawMethod === "manual" のとき, 現在の座標をクリアする
        if (this.drawMethod === "alongLine") {
          // this.candPointsを描画
          // this.points += this.candPoints;
          this.candPoints = [];
        }
        this.cursor = null;
        this.events.push(event.type);
      },
    }
  }
</script>

<style>
  #plotCanvas {
    border: solid;
    border-width: 3px;
    border-color: black;
  }

  li {
    list-style-type: none;
  }
  
  .container {
    display: -webkit-flex;
    display: -moz-flex;
    display: -ms-flex;
    display: -o-flex;
    display: flex;
  }

  .box {
    margin: 10px;
  }
</style>