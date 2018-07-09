<template>
  <div>
    <h1 class="title is-1">Plott</h1>
    <div class="container">
      <div class="box">
        <div>
          <button @click="previous" class="button" id="previousButton" :disabled="isFirst">← 戻る</button>
          <button @click="next" class="button" id="nextButton" :disabled="isLast">進む →</button>
          <label class="subtitle is-5">点の数: {{ points.length }} / 1000</label>
        </div>
        <canvas
          id='plotCanvas' ref='canvas' :width=width :height=height
          @mousedown="mouseDownCanvas" @mousemove="mouseMoveCanvas" @mouseout="mouseOutCanvas"
          @mouseup="mouseUpCanvas"
          @touchstart.prevent="touchStartCanvas" @touchmove.prevent="touchMoveCanvas" @touchend.prevent="touchEndCanvas"
        ></canvas>
      </div>

      <div class="box">
        <p class="subtitle is-4">描画方法</p>
        <Tabs isBoxed="true" isToggle="true" @select-tab="setMode">
          <Tab name="ジェスチャー">
            <div class="sizeSliderContainer">
              <label>密度: </label>
              <input class="slider" step="0.001" min="0.001" max="0.01" value="0.001" type="range" v-model="density" />
            </div>
          </Tab>
          <Tab name="マニュアル" @click="setMode" :selected="true">
            <Tabs isBoxed="true" @select-tab="setManualMode">
              <Tab name="ブラシ" :selected="true">
                <div class="sizeSliderContainer">
                  <label>サイズ: </label>
                  <input class="slider" step="1" min="1" max="100" value="25" type="range" v-model="cursorRadius" />
                </div>
              </Tab>
              <Tab name="一点ずつ"></Tab>
              <Tab name="消しゴム">
                <div class="sizeSliderContainer">
                  <label>サイズ: </label>
                  <input class="slider" step="1" min="1" max="100" value="25" type="range" v-model="cursorRadius" />
                </div>
              </Tab>
            </Tabs>
          </Tab>
        </Tabs>

        <div v-show="isDev">
          <h3>力: {{ force }}</h3>
          <h3>半径: {{ radius }}</h3>
          <h3>描画方法: {{ mode }}</h3>
          <h3>マニュアルモードでの描画方法: {{ mode === "マニュアル" ? manualMode : "ジェスチャーモード"}}</h3>
          <h3>現在の履歴のindex: {{ currentIndex }}</h3>
          <h3>履歴</h3>
          <ul>
            <li v-for="history in actionHistory">action: {{ history[0] }}</li>
          </ul>
        </div>

      </div>
    </div>
    <button @click="saveImage" class="button">キャンバスの画像を出力</button>

    <div class="info" v-show="isDev">
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
  import Tabs from './ui/Tabs.vue'
  import Tab from './ui/Tab.vue'
  // canvas操作

  // 座標変換
  // let convertCoordinate = (coord) => {
  //   let canvasX = (coord[0] - xMin) / (xMax - xMin) * width;
  //   let canvasY = (coord[1] - yMax) / (yMin - yMax) * height;
  //   return [canvasX, canvasY];
  // };

  let canvas = null;
  let ctx = null;
  let mouseMoveFlag = true;

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

  // (x, y) から半径r以内にランダムに点を一つ打つ
  let getRandomCoordinate = function (x, y, r) {
    let randomX = (Math.random() - 0.5) * 2 * r;
    let randomY = (Math.random() - 0.5) * 2 * r;
    return [x + randomX, y + randomY];
  };

  let getRandomCoordinatesInPolygon = function (pointsOfPolygon, n) {
    // pointsOfPolygonが作る多角形の内部に, n個の座標を一様に生成する.
    let [leftTop, length] = getBoundingSquare(pointsOfPolygon);
    let originX = leftTop[0] + length / 2;
    let originY = leftTop[1] + length / 2;
    let areaOfSquare = length ** 2;
    let areaOfPolygon = calcArea(pointsOfPolygon);
    let m = areaOfSquare / areaOfPolygon * n;

    let randomCoordinates = [];

    for (let i = 0; i < m; i++) {
      let x = originX + (Math.random() - 0.5) * length;
      let y = originY + (Math.random() - 0.5) * length;
      randomCoordinates.push([x, y]);
    }

    return randomCoordinates.filter(function (point) {
      return isPointInPolygon(point, pointsOfPolygon);
    });
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

  // canvas上に円を描画
  let strokeCircle = function (centerX, centerY, radius, lineWidth=2, color='black') {
    ctx.beginPath();
    ctx.arc(centerX, centerY, radius, 0, 2*Math.PI, true);
    ctx.strokeStyle = color;
    ctx.lineWidth = lineWidth;
    ctx.stroke();
  };

  // canvas上に円を描画し, 内部を塗りつぶす
  let fillCircle = function (centerX, centerY, radius, color='black') {
    ctx.beginPath();
    ctx.arc(centerX, centerY, radius, 0, 2*Math.PI, true);
    ctx.fillStyle = color;
    ctx.fill();
  };

  // canvas上の線を消去
  let eraseLine = function (startX, startY, endX, endY, lineWidth=3) {
    ctx.beginPath();
    ctx.globalCompositeOperation = 'destination-out';
    ctx.moveTo(startX, startY);
    ctx.lineTo(endX, endY);
    ctx.lineWidth = lineWidth;
    ctx.stroke();
    ctx.globalCompositeOperation = 'source-over';
  };

  let erasePolygon = function (points) {
    for (let i = 0; i < points.length - 1; i++) {
      eraseLine(points[i][0], points[i][1], points[i+1][0], points[i+1][1], 5);
    }
  };

  let eraseStrokedCircle = function (centerX, centerY, radius, lineWidth=2) {
    ctx.beginPath();
    ctx.globalCompositeOperation = 'destination-out';
    ctx.arc(centerX, centerY, radius, 0, 2*Math.PI, true);
    ctx.lineWidth = lineWidth;
    ctx.stroke();
    ctx.globalCompositeOperation = 'source-over';
  };

  let eraseFilledCircle = function (centerX, centerY, radius) {
    ctx.beginPath();
    ctx.globalCompositeOperation = 'destination-out';
    ctx.arc(centerX, centerY, radius+1, 0, 2*Math.PI, true);
    ctx.fill();
    ctx.globalCompositeOperation = 'source-over';
  };

  let isPointInPolygon = function (point, pointsOfPolygon) {
    // pointsOfPolygonが作る多角形の内部にpointがあるかを判定する
    let count = 0;
    for (let i = 0; i < pointsOfPolygon.length-1; i++) {
      let x1 = pointsOfPolygon[i][0];
      let y1 = pointsOfPolygon[i][1];
      let x2 = pointsOfPolygon[i+1][0];
      let y2 = pointsOfPolygon[i+1][1];

      if (y1 !== y2) {
        if (x1 === x2) {
          if (point[0] <= x1 && Math.max(y1, y2) - point[1] >= 0 && point[1] - Math.min(y1, y2) >= 0)
            count += 1;
        } else {
          let x = (x1 - x2) / (y1 - y2) * (point[1] - y1) + x1;
          if (x >= point[0] && Math.max(x1, x2) - x >= 0 && x - Math.min(x1, x2) >= 0)
            count += 1;
        }
      }
    }
    // countが奇数なら内側に入っている
    return count % 2 === 1;
  };

  let calcArea = function (pointsOfPolygon) {
    // pointsOfPolygonで作られる多角形の面積を計算
    let area = 0;
    for (let i = 0; i < pointsOfPolygon.length-1; i++) {
      let p1 = pointsOfPolygon[i];
      let p2 = pointsOfPolygon[i+1];
      area += 1/2 * (p1[0]*p2[1]-p1[1]*p2[0]);
    }
    return Math.abs(area);
  };

  let getCornerCoordinates = function (pointsOfPolygon) {
    // pointsOfPolygonで作られる多角形の角となる座標を求める

    let maxX = 0;
    let maxXCoordinate = null;
    let minX = 1000000000;
    let minXCoordinate = null;
    let maxY = 0;
    let maxYCoordinate = null;
    let minY = 1000000000;
    let minYCoordinate = null;

    for (let point of pointsOfPolygon) {
      let x = point[0];
      let y = point[1];

      if (x > maxX) {
        maxX = x;
        maxXCoordinate = point;
      }

      if (x < minX) {
        minX = x;
        minXCoordinate = point;
      }

      if (y > maxY) {
        maxY = y;
        maxYCoordinate = point;
      }

      if (y < minY) {
        minY = y;
        minYCoordinate = point;
      }
    }

    return [maxXCoordinate, minXCoordinate, maxYCoordinate, minYCoordinate];
  };

  let getBoundingSquare = function (pointsOfPolygon) {
    // pointsOfPolygonで作られる多角形を覆う正方形を取得
    let [maxXPoint, minXPoint, maxYPoint, minYPoint] = getCornerCoordinates(pointsOfPolygon);
    let maxX = maxXPoint[0];
    let minX = minXPoint[0];
    let maxY = maxYPoint[1];
    let minY = minYPoint[1];
    let length = Math.max(maxX - minX, maxY - minY);
    return [[minX, minY], length]
  };

  export default {
    name: 'PlotCanvas',
    components: {
      Tabs, Tab
    },
    props: {
      size: Array,  // (height, width)
    },
    computed: {
      height() {return this.size[0]},
      width() {return this.size[1]},
      isFirst() {return this.currentIndex === -1},
      isLast() {return this.currentIndex === this.actionHistory.length - 1}
    },
    data() {
      return {
        points: [],
        actionHistory: [],  // list of [action_name, points]
        currentIndex: -1,
        candidatePoints: [],
        candidateCount: 0,
        pointsOfPolygon: [],
        cursor: null,
        cursorHistory: [],
        count: 0,
        events: [],
        force: 0,
        mode: "マニュアル",
        manualMode: "ブラシ",
        radius: 0,
        pointRadius: 3,
        cursorRadius: 25,
        density: 0.005,
        isDev: false,
      }
    },
    mounted() {
      canvas = this.$refs.canvas;
      ctx = canvas.getContext('2d');
    },
    methods: {
      setMode(mode) {
        this.mode = mode;
      },
      setManualMode(mode) {
        this.manualMode = mode;
      },
      drawPoints() {
        for (let point of this.points) {
          fillCircle(point[0], point[1], this.pointRadius);
        }
      },
      addPoints(pointsToAdd) {
        // points: list of point (list of [x, y])
        // 1. this.pointsを更新
        // 2. 描画

        this.points = this.points.concat(pointsToAdd);

        // 描画
        for (let point of pointsToAdd) {
          fillCircle(point[0], point[1], this.pointRadius);
        }
      },
      deletePoints(pointsToDelete) {
        // points: list of point
        // 1. this.pointsを更新
        // 2. 描画

        for (let pointToDelete of pointsToDelete) {
          this.points = this.points.filter(function (point) {
            return point[0] !== pointToDelete[0] & point[1] !== pointToDelete[1];
          });
        }
        // 描画
        for (let point of pointsToDelete) {
          eraseFilledCircle(point[0], point[1], this.pointRadius);
        }
      },
      add(points) {
        this.addPoints(points);
        this.actionHistory = this.actionHistory.slice(0, this.currentIndex + 1);
        this.actionHistory.push(['add', points]);
        this.currentIndex += 1;
      },
      delete(points) {
        this.deletePoints(points);
        this.actionHistory = this.actionHistory.slice(0, this.currentIndex + 1);
        this.actionHistory.push(['delete', points]);
        this.currentIndex += 1;
      },
      undo(action_name, points) {
        // action_nameが 'add'ならdeleteを, 'delete'ならaddを対象のpointsに対して実行する

        if (action_name === 'add') {
          this.deletePoints(points);
        } else if (action_name === 'delete') {
          this.addPoints(points);
        }
        this.currentIndex -= 1;
      },
      redo(action_name, points) {
        if (action_name === 'add') {
          this.addPoints(points);
        } else if (action_name === 'delete') {
          this.deletePoints(points);
        }
        this.currentIndex += 1;
      },
      previous() {
        if (this.currentIndex >= 0) {
          let [action_name, points] = this.actionHistory[this.currentIndex];
          this.undo(action_name, points);
        }
      },
      next() {
        if (this.currentIndex + 1 < this.actionHistory.length) {
          let [action_name, points] = this.actionHistory[this.currentIndex + 1];
          this.redo(action_name, points);
        }
      },
      addPointsFromRegion() {
        // ジェスチャーモードで領域を指定したあと, 点を追加
        let area = calcArea(this.pointsOfPolygon);
        let points = getRandomCoordinatesInPolygon(this.pointsOfPolygon, Math.round(this.density * area));
        this.add(points);
        erasePolygon(this.pointsOfPolygon);
        this.pointsOfPolygon = [];
      },
      deltePointsFromRegion() {
        // ジェスチャーモードで領域を指定したあと, 点を削除
        let points = this.points.filter(function (point) {
          return isPointInPolygon(point, this.pointsOfPolygon);
        });
        this.delete(points);
        erasePolygon(this.pointsOfPolygon);
        this.pointsOfPolygon = [];
      },
      mouseDownCanvas(event) {
        // mode === "alongLine のとき, this.cursorを現在の座標にセット
        // mode === "manual" のとき, 点を描画

        let [x, y] = getCoordinate(event);

        if (this.mode === "ジェスチャー") {
          this.cursor = [x, y];
          this.pointsOfPolygon.push(this.cursor);
        } else if (this.mode === "マニュアル") {
          if (this.manualMode === "一点ずつ") {
            this.add([[x, y]]);
          }
        }
        this.events.push(event.type);
      },
      touchStartCanvas(event) {
        // mode === "alongLine のとき, this.cursorを現在の座標にセット
        // mode === "manual" のとき, 点を描画

        let [x, y] = getTouchCoordinate(event);

        if (this.mode === "ジェスチャー") {
          this.cursor = [x, y];
        }
        else if (this.mode === "マニュアル") {
          if (this.manualMode === "一点ずつ") {
            this.add([[x, y]]);
          }
        } else {
          alert("ERROR: invalid mode")
        }

        this.events.push(event.type);
      },
      mouseMoveCanvas(event) {
        // mode === "alongLine のとき, ユーザーが引いた線を挟むような補助線を引き囲まれた領域を異なる色に変える
        // mode === "manual" のとき, 円形のカーソルを動かす. ドラッグしているときに点を描画

        if (this.mode === "ジェスチャー") {
          if (this.cursor != null) {
            let [x, y] = getCoordinate(event);
            drawLine(this.cursor[0], this.cursor[1], x, y, 3, 'blue');
            this.pointsOfPolygon.push([x, y]);
            this.cursor = [x, y];
          }
        } else if (this.mode === 'マニュアル') {

          // カーソルを移動させる
          if (this.cursor != null) {
            // 一つ前のcursorの位置をクリアする
            eraseStrokedCircle(this.cursor[0], this.cursor[1], this.cursorRadius, 6);
          }
          this.drawPoints();
          let [x, y] = getCoordinate(event);
          this.cursor = [x, y];

          if (this.manualMode === 'ブラシ') {
            // ブラシで描画
            // 大きな円を描画
            strokeCircle(x, y, this.cursorRadius);

            // ドラッグ中の場合カーソル内の点にplotする
            if ((event.buttons === 1 || event.which === 1) && this.cursor !== [x, y]) {
              let [x, y] = getRandomCoordinate(this.cursor[0], this.cursor[1], this.cursorRadius);
              this.candidatePoints.push([x, y]);
              this.candidateCount += 1;
              this.add([[x, y]]);
            }
          } else if (this.manualMode === '一点ずつ') {
            // 一点ずつは何もしない
          } else if (this.manualMode === '消しゴム') {
            // 消しゴム
            strokeCircle(x, y, this.cursorRadius);
            if ((event.buttons === 1 || event.which === 1) && this.cursor !== [x, y]) {
              let pointsToDelete = this.pointsInCursor();
              for (let point of pointsToDelete) {
                if (! ([point[0], point[1]] in this.candidatePoints))
                  this.candidatePoints.push(point);
              }
              this.candidateCount += 1;
              this.delete(pointsToDelete)
            }
          } else {
            alert('ERROR: ' + this.manualMode + ' is invalid manualMode')
          }
        }
        this.drawPoints();

        if (!(this.events.length > 0 && this.events[this.events.length - 1] === "mousemove")) {
          this.events.push(event.type);
        }
      },
      touchMoveCanvas(event) {
        // mode === "alongLine のとき, ユーザーが引いた線を挟むような補助線を引き囲まれた領域を異なる色に変える
        // mode === "manual" のとき, 何もしない

        // 圧力を取得してみる
        let touch = event.touches[0];
        this.force = touch.force;

        // 圧力の大きさを円で表現
        // ctx.beginPath();
        // ctx.globalCompositeOperation = 'destination-out';
        // ctx.arc(this.cursor[0], this.cursor[1], this.radius, 0, 2*Math.PI, true);
        // ctx.stroke();
        // ctx.globalCompositeOperation = 'source-over';
        this.radius = this.force * 100;
        ctx.arc(this.cursor[0], this.cursor[1], this.radius, 0, 2 * Math.PI, true);
        ctx.stroke();

        let [x, y] = getTouchCoordinate(event);
        // if (this.cursor == null) {
        //   drawLine(x, y, x, y);
        // } else {
        //   drawLine(this.cursor[0], this.cursor[1], x, y);
        // }
        drawLine(this.cursor[0], this.cursor[1], x, y);
        this.cursor = [x, y];
        if (!(this.events.length > 0 && this.events[this.events.length - 1] === "touchmove")) {
          this.events.push(event.type);
        }
      },
      mouseUpCanvas(event) {
        if (this.mode === 'ジェスチャー') {
          if (this.cursor != null) {
            drawLine(this.cursor[0], this.cursor[1], this.pointsOfPolygon[0][0], this.pointsOfPolygon[0][1], 3, 'blue');
            this.pointsOfPolygon.push(this.pointsOfPolygon[0]);
            this.cursor = null;
            this.addPointsFromRegion();
          }
        } else if (this.mode === 'マニュアル') {
          // this.candidatePointsをまとめる
          if (this.manualMode === 'ブラシ') {
            // add
            this.actionHistory = this.actionHistory.slice(0, this.actionHistory.length - this.candidateCount);
            this.actionHistory.push(['add', this.candidatePoints]);
            this.currentIndex = this.actionHistory.length - 1;
          } else if (this.manualMode === '消しゴム') {
            // delete
            this.actionHistory = this.actionHistory.slice(0, this.actionHistory.length - this.candidateCount);
            this.actionHistory.push(['delete', this.candidatePoints]);
            this.currentIndex = this.actionHistory.length - 1;
          }
          this.candidatePoints = [];
          this.candidateCount = 0;
        }
        this.drawPoints();
      },
      mouseOutCanvas() {
        // mode === "alongLine" のとき, 作成した領域内の候補の点をthis.pointsに加え, 現在の座標をクリアする
        // mode === "manual" のとき, 現在の座標をクリアする
        if (this.mode === "ジェスチャー") {
          // ジェスチャーモード
          if (this.cursor != null) {
            drawLine(this.cursor[0], this.cursor[1], this.pointsOfPolygon[0][0], this.pointsOfPolygon[0][1], 3, 'blue');
            this.pointsOfPolygon.push(this.pointsOfPolygon[0]);
            this.cursor = null;
            this.addPointsFromRegion();
          }
        } else if (this.mode === "マニュアル") {
          // ブラシの表示をクリア
          if (this.cursor != null) {
            // cursorをクリアする
            eraseStrokedCircle(this.cursor[0], this.cursor[1], this.cursorRadius, 6);
            this.drawPoints();
          }
        }
        this.cursor = null;
        this.events.push(event.type);
      },
      touchEndCanvas() {
        // mode === "alongLine" のとき, 作成した領域内の候補の点をthis.pointsに加え, 現在の座標をクリアする
        // mode === "manual" のとき, 現在の座標をクリアする
        if (this.mode === "alongLine") {
          // this.candPointsを描画
          // this.points += this.candPoints;
          this.candPoints = [];
        }
        this.cursor = null;
        this.events.push(event.type);
      },
      pointsInCursor() {
        // this.pointsの中でcursor内の点を返す
        let x = this.cursor[0];
        let y = this.cursor[1];
        let r = this.cursorRadius;
        return this.points.filter(function (point) {
          return (point[0] - x) ** 2 + (point[1] - y) ** 2 <= r**2;
        })
      },
      saveImage() {
        let base64 = canvas.toDataURL("image/png");
        let blob = this.base64ToBlob(base64);
        this.saveBlob(blob, 'plot.png')
      },
      base64ToBlob(base64) {
        let tmp = base64.split(',');
        // base64データの文字列をデコード
        let data = atob(tmp[1]);
        // tmp[0]の文字列（data:image/png;base64）からコンテンツタイプ（image/png）部分を取得
        let mime = tmp[0].split(':')[1].split(';')[0];
        //  1文字ごとにUTF-16コードを表す 0から65535 の整数を取得
        let buf = new Uint8Array(data.length);
        for (let i = 0; i < data.length; i++) {
          buf[i] = data.charCodeAt(i);
        }
        // blobデータを作成
        return new Blob([buf], {type: mime});
      },
      saveBlob(blob, fileName) {
        let url = (window.URL || window.webkitURL);
        // ダウンロード用のURL作成
        let dataUrl = url.createObjectURL(blob);
        // イベント作成
        let event = document.createEvent("MouseEvents");
        event.initMouseEvent("click", true, false, window, 0, 0, 0, 0, 0, false, false, false, false, 0, null);
        // a要素を作成
        let a = document.createElementNS("http://www.w3.org/1999/xhtml", "a");
        // ダウンロード用のURLセット
        a.href = dataUrl;
        // ファイル名セット
        a.download = fileName;
        // イベントの発火
        a.dispatchEvent(event);
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

  #previousButton {
    text-align: left;
    margin-right: 10px;
  }

  #nextButton {
    text-align: left;
    margin-right: 100px;
  }

  label.subtitle {
    text-align: right;
  }

  .sizeSliderContainer {
    margin-top: 50px;
  }
</style>