<template>
  <div class="home">
    <div class="btnwrap">
      <div
        @click="showStrokeColorPicker = !showStrokeColorPicker"
        class="btn-color"
        ref="strokeColor"
      >
        <Sketch-picker
          v-if="showStrokeColorPicker"
          class="color-picker"
          :value="strokeColor"
          @input="updateStrokeColor"
        />
      </div>
      <span class="color-label">strokeColor</span>

      <div
        @click="showFillColorPicker = !showFillColorPicker"
        class="btn-color"
        ref="fillColor"
      >
        <Sketch-picker
          v-if="showFillColorPicker"
          class="color-picker"
          :value="fillColor"
          @input="updateFillColor"
        />
      </div>
      <span class="color-label">fillColor</span>

      <div
        @click="showBgColorPicker = !showBgColorPicker"
        class="btn-color"
        ref="bgColor"
      >
        <Sketch-picker
          v-if="showBgColorPicker"
          class="color-picker"
          :value="bgColor"
          @input="updateBgColor"
        />
      </div>
      <span class="color-label">bgColor</span>

      <div class="brushWidth">
        <label>lineSize:{{ lineSize }}</label>
        <input type="range" name="vol" min="1" max="100" v-model="lineSize" />
      </div>
      <div class="brushWidth">
        <label>fontSize:{{ fontSize }}</label>
        <input type="range" name="vol" min="18" max="50" v-model="fontSize" />
      </div>
      <button
        @click="tapToolBtn('brush')"
        :class="{ active: selectTool == 'brush' }"
        class="btn-tool"
      >
        pencil
      </button>
      <button
        @click="tapToolBtn('line')"
        :class="{ active: selectTool == 'line' }"
        class="btn-tool"
      >
        line
      </button>
      <button
        @click="tapToolBtn('rect')"
        :class="{ active: selectTool == 'rect' }"
        class="btn-tool"
      >
        rect
      </button>
      <button
        @click="tapToolBtn('circle')"
        :class="{ active: selectTool == 'circle' }"
        class="btn-tool"
      >
        circle
      </button>
      <button
        @click="tapToolBtn('text')"
        :class="{ active: selectTool == 'text' }"
        class="btn-tool"
      >
        text
      </button>
      <button
        @click="tapToolBtn('eraser')"
        :class="{ active: selectTool == 'eraser' }"
        class="btn-tool"
      >
        eraser
      </button>
      <button
        @click="tapToolBtn('move')"
        :class="{ active: selectTool == 'move' }"
        class="btn-tool"
      >
        move
      </button>
      <button @click="tapScaleBtn(-1)" class="btn-tool">zoom -</button>
      <button @click="tapScaleBtn(1)" class="btn-tool">zoom +</button>
      <button @click="tapHistoryBtn(-1)" class="btn-tool">undo</button>
      <button @click="tapHistoryBtn(1)" class="btn-tool">redo</button>

      <button @click="tapClearBtn" class="btn-tool">clear</button>
      <button @click="tapSaveBtn" class="btn-tool">save</button>
    </div>
    <canvas id="c" width="1200" height="800"></canvas>
  </div>
</template>

<script>
// import { fabric } from "@/libs/fabric.min";
import { fabric } from "fabric";
import '@/libs/eraser_brush.mixin'
import { Sketch } from "vue-color";
export default {
  name: "Home",
  components: {
    "Sketch-picker": Sketch,
  },
  data() {
    return {
      canvas: null, // fabric canvas对象

      strokeColor: "#000000", // 线框色
      showStrokeColorPicker: false, // 是否显示 线框色选择器

      fillColor: "rgba(0,0,0,0)", // 填充色
      showFillColorPicker: false, // 是否显示 填充色选择器

      bgColor: "#2F782C", // 背景色
      showBgColorPicker: false, // 是否显示 背景色选择器

      lineSize: 1, // 线条大小 （线条 and 线框）
      fontSize: 18, // 字体大小

      selectTool: "", // 当前用户选择的绘图工具 画笔：brush 直线：line 矩形：rect 圆形 circle 文本 text

      mouseFrom: {}, // 鼠标绘制起点
      mouseTo: {}, // 鼠标绘制重点

      drawingObject: null, // 保存鼠标未松开时用户绘制的临时图像

      textObject: null, // 保存用户创建的文本对象

      isDrawing: false, // 当前是否正在绘制图形（画笔，文本模式除外）
      stateArr: [], // 保存画布的操作记录
      stateIdx: 0, // 当前操作步数
      isRedoing: false // 当前是否在执行撤销或重做操作
    };
  },
  watch: {
    // 监听线条大小变化
    lineSize() {
      this.canvas.freeDrawingBrush.width = parseInt(this.lineSize, 10);
      this.lineSize = parseInt(this.lineSize, 10);
    },
    // 监听背景色变化
    bgColor() {
      this.canvas.setBackgroundColor(this.bgColor, undefined, {
        erasable: false,
      });
      this.canvas.renderAll();
    },
  },
  methods: {
    // 监听线框色选择器 颜色选择
    updateStrokeColor(val) {
      // 保存用户选择的线框色
      this.strokeColor = val.hex;
      // 修改当前选择的颜色指示
      this.$refs.strokeColor.style.backgroundColor = this.strokeColor;
    },
    // 监听填充色选择器 颜色选择
    updateFillColor(val) {
      // 保存用户选择的线框色
      this.fillColor = val.hex8;
      // 修改当前选择的颜色指示
      this.$refs.fillColor.style.backgroundColor = this.fillColor;
    },
    // 监听背景色选择器 颜色选择
    updateBgColor(val) {
      // 保存用户选择的背景色
      this.bgColor = val.hex;
      this.$refs.bgColor.style.backgroundColor = this.bgColor;
    },
    // 初始化画布
    initCanvas() {
      // 初始化线框色 与 指示器
      this.strokeColor = "#000000";
      this.$refs.strokeColor.style.backgroundColor = this.strokeColor;
      // 初始化填充色 与 指示器
      this.fillColor = "rgba(0,0,0,0)";
      this.$refs.fillColor.style.backgroundColor = this.fillColor;
      // 初始化背景色 与 指示器
      this.bgColor = "#2F782C";
      this.$refs.bgColor.style.backgroundColor = this.bgColor;

      // 初始化 fabric canvas对象
      if (!this.canvas) {
        this.canvas = new fabric.Canvas("c");
        // 设置画布背景色 (背景色需要这样设置，否则拓展的橡皮功能会报错)
        this.canvas.setBackgroundColor(this.bgColor, undefined, {
          erasable: false,
        });
        // 设置背景色不受缩放与平移的影响
        this.canvas.set("backgroundVpt", false);
        // 禁止用户进行组选择
        this.canvas.selection = false;
        // 设置当前鼠标停留在
        this.canvas.hoverCursor = "default";
        // 重新渲染画布
        this.canvas.renderAll();
        // 记录画布原始状态
        this.stateArr.push(JSON.stringify(this.canvas))
        this.stateIdx = 0
      }
    },
    // 初始化画布事件
    initCanvasEvent() {
      // 操作类型集合
      let toolTypes = ["line", "rect", "circle", "text", "move"];

      // 监听鼠标按下事件
      this.canvas.on("mouse:down", (options) => {

        if (this.selectTool != 'text' && this.textObject) {
          // 如果当前存在文本对象，并且不是进行添加文字操作 则 退出编辑模式，并删除临时的文本对象
          // 将当前文本对象退出编辑模式
          this.textObject.exitEditing();
          this.textObject.set("backgroundColor", "rgba(0,0,0,0)");
          if (this.textObject.text == "") {
            this.canvas.remove(this.textObject);
          }
          this.canvas.renderAll();
          this.textObject = null;
        }
        // 判断当前是否选择了集合中的操作
        if (toolTypes.indexOf(this.selectTool) != -1) {
          // 记录当前鼠标的起点坐标 (减去画布在 x y轴的偏移，因为画布左上角坐标不一定在浏览器的窗口左上角)
          this.mouseFrom.x = options.e.clientX - this.canvas._offset.left;
          this.mouseFrom.y = options.e.clientY - this.canvas._offset.top;
          // 判断当前选择的工具是否为文本
          if (this.selectTool == "text") {
            // 文本工具初始化
            this.initText();
          } else {
            // 设置当前正在进行绘图 或 移动操作
            this.isDrawing = true;
          }
        }
      });
      // 监听鼠标移动事件
      this.canvas.on("mouse:move", (options) => {
        // 如果当前正在进行绘图或移动相关操作
        if (this.isDrawing) {
          // 记录当前鼠标移动终点坐标 (减去画布在 x y轴的偏移，因为画布左上角坐标不一定在浏览器的窗口左上角)
          this.mouseTo.x = options.e.clientX - this.canvas._offset.left
          this.mouseTo.y = options.e.clientY - this.canvas._offset.top
          switch (this.selectTool) {
            case "line":
              // 当前绘制直线，初始化直线绘制
              this.initLine();
              break;
            case "rect":
              // 初始化 矩形绘制
              this.initRect();
              break;
            case "circle":
              // 初始化 绘制圆形
              this.initCircle();
              break;
            case "move":
              // 初始化画布移动
              this.initMove();
          }
        }
      });
      // 监听鼠标松开事件
      this.canvas.on("mouse:up", () => {
        // 如果当前正在进行绘图或移动相关操作
        if (this.isDrawing) {
          // 清空鼠标移动时保存的临时绘图对象
          this.drawingObject = null;
          // 重置正在绘制图形标志
          this.isDrawing = false;
          // 清空鼠标保存记录
          this.resetMove();
          // 如果当前进行的是移动操作，鼠标松开重置当前视口缩放系数
          if (this.selectTool == "move") {
            this.canvas.setViewportTransform(this.canvas.viewportTransform);
          }
        }
      });
      // 监听画布渲染完成
      this.canvas.on("after:render", () => {
        if (!this.isRedoing) {
          // 当前不是进行撤销或重做操作
          // 在绘画时会频繁触发该回调，所以间隔1s记录当前状态
          if (this.recordTimer) {
            clearTimeout(this.recordTimer)
            this.recordTimer = null
          }
          this.recordTimer = setTimeout(() => {
            this.stateArr.push(JSON.stringify(this.canvas))
            this.stateIdx++
          }, 100)
        }else {
          // 当前正在执行撤销或重做操作，不记录重新绘制的画布
          this.isRedoing = false
        }
      })
    },
    // 初始化画笔工具
    initBruch() {
      // 设置绘画模式画笔类型为 铅笔类型
      this.canvas.freeDrawingBrush = new fabric.PencilBrush(this.canvas);
      // 设置画布模式为绘画模式
      this.canvas.isDrawingMode = true;
      // 设置绘画模式 画笔颜色与画笔线条大小
      this.canvas.freeDrawingBrush.color = this.strokeColor;
      this.canvas.freeDrawingBrush.width = parseInt(this.lineSize, 10);
    },
    // 初始化 绘制直线
    initLine() {
      // 根据保存的鼠标起始点坐标 创建直线对象
      let canvasObject = new fabric.Line(
        [
          this.getTransformedPosX(this.mouseFrom.x),
          this.getTransformedPosY(this.mouseFrom.y),
          this.getTransformedPosX(this.mouseTo.x),
          this.getTransformedPosY(this.mouseTo.y),
        ],
        {
          fill: this.fillColor,
          stroke: this.strokeColor,
          strokeWidth: this.lineSize,
        }
      );
      // 绘制 图形对象
      this.startDrawingObject(canvasObject);
    },
    // 初始化 绘制矩形
    initRect() {
      // 计算矩形长宽
      let left = this.getTransformedPosX(this.mouseFrom.x);
      let top = this.getTransformedPosY(this.mouseFrom.y);
      let width = this.mouseTo.x - this.mouseFrom.x;
      let height = this.mouseTo.y - this.mouseFrom.y;
      // 创建矩形 对象
      let canvasObject = new fabric.Rect({
        left: left,
        top: top,
        width: width,
        height: height,
        stroke: this.strokeColor,
        fill: this.fillColor,
        strokeWidth: this.lineSize,
      });
      // 绘制矩形
      this.startDrawingObject(canvasObject);
    },
    // 初始化绘制圆形
    initCircle() {
      let left = this.getTransformedPosX(this.mouseFrom.x);
      let top = this.getTransformedPosY(this.mouseFrom.y);
      // 计算圆形半径
      let radius =
        Math.sqrt(
          (this.getTransformedPosX(this.mouseTo.x) - left) *
            (this.getTransformedPosY(this.mouseTo.x) - left) +
            (this.getTransformedPosX(this.mouseTo.y) - top) *
              (this.getTransformedPosY(this.mouseTo.y) - top)
        ) / 2;
      // 创建 原型对象
      let canvasObject = new fabric.Circle({
        left: left,
        top: top,
        stroke: this.strokeColor,
        fill: this.fillColor,
        radius: radius,
        strokeWidth: this.lineSize,
      });
      // 绘制圆形对象
      this.startDrawingObject(canvasObject);
    },
    // 初始化文本工具
    initText() {
      if (!this.textObject) {
        // 当前不存在绘制中的文本对象

        // 创建文本对象
        this.textObject = new fabric.Textbox("", {
          left: this.getTransformedPosX(this.mouseFrom.x),
          top: this.getTransformedPosY(this.mouseFrom.y),
          fontSize: this.fontSize,
          fill: this.strokeColor,
          hasControls: false,
          editable: true,
          width: 30,
          backgroundColor: "#fff",
          selectable: false,
        });
        this.canvas.add(this.textObject);
        // 文本打开编辑模式
        this.textObject.enterEditing();
        // 文本编辑框获取焦点
        this.textObject.hiddenTextarea.focus();
      } else {
        // 将当前文本对象退出编辑模式
        this.textObject.exitEditing();
        this.textObject.set("backgroundColor", "rgba(0,0,0,0)");
        if (this.textObject.text == "") {
          this.canvas.remove(this.textObject);
        }
        this.canvas.renderAll();
        this.textObject = null;
        return;
      }
    },
    // 初始化橡皮擦功能
    initEraser() {
      this.canvas.freeDrawingBrush = new fabric.EraserBrush(this.canvas);
      this.canvas.freeDrawingBrush.width = parseInt(this.lineSize, 10);
      this.canvas.isDrawingMode = true;
    },
    // 初始化画布移动
    initMove() {
      var vpt = this.canvas.viewportTransform;
      vpt[4] += this.mouseTo.x - this.mouseFrom.x;
      vpt[5] += this.mouseTo.y - this.mouseFrom.y;
      this.canvas.requestRenderAll();
      this.mouseFrom.x = this.mouseTo.x;
      this.mouseFrom.y = this.mouseTo.y;
    },
    // 绘制图形
    startDrawingObject(canvasObject) {
      // 禁止用户选择当前正在绘制的图形
      canvasObject.selectable = false;
      // 如果当前图形已绘制，清除上一次绘制的图形
      if (this.drawingObject) {
        this.canvas.remove(this.drawingObject);
      }
      // 将绘制对象添加到 canvas中
      this.canvas.add(canvasObject);
      // 保存当前绘制的图形
      this.drawingObject = canvasObject;
    },
    // 清空鼠标移动记录 （起点 与 终点）
    resetMove() {
      this.mouseFrom = {};
      this.mouseTo = {};
    },
    // 绘图工具点击选择
    tapToolBtn(tool) {
      if (this.selectTool == tool) return;
      // 保存当前选中的绘图工具
      this.selectTool = tool;

      // 选择任何工具前进行一些重置工作
      // 禁用画笔模式
      this.canvas.isDrawingMode = false;
      // 禁止图形选择编辑
      let drawObjects = this.canvas.getObjects();
      if (drawObjects.length > 0) {
        drawObjects.map((item) => {
          item.set("selectable", false);
        });
      }
      if (this.selectTool == "brush") {
        // 如果用户选择的是画笔工具，直接初始化，无需等待用户进行鼠标操作
        this.initBruch();
      } else if (this.selectTool == "eraser") {
        // 如果用户选择的是橡皮擦工具，直接初始化，无需等待用户进行鼠标操作
        this.initEraser();
      }
    },
    // 缩放按钮点击
    tapScaleBtn(flag) {
      // flag -1 缩小 1 放大
      let zoom = this.canvas.getZoom();
      if (flag > 0) {
        // 放大
        zoom *= 1.1;
      } else {
        // 缩小
        zoom *= 0.9;
      }
      // zoom 不能大于 20 不能小于0.01
      zoom = zoom > 20 ? 20 : zoom;
      zoom = zoom < 0.01 ? 0.01 : zoom;
      this.canvas.setZoom(zoom);
    },
    // 撤销重做按钮点击
    tapHistoryBtn(flag) {
      this.isRedoing = true

      let stateIdx = this.stateIdx + flag
      // 判断是否已经到了第一步操作
      if (stateIdx < 0) return;
      // 判断是否已经到了最后一步操作
      if (stateIdx >= this.stateArr.length) return;
      if (this.stateArr[stateIdx]) {
        this.canvas.loadFromJSON(this.stateArr[stateIdx])
        if (this.canvas.getObjects().length > 0) {
          this.canvas.getObjects().forEach(item => {
            item.set('selectable', false)
          })
        }
        this.stateIdx = stateIdx
      }
    },
    // 监听画布重新绘制
    tapClearBtn() {
      let children = this.canvas.getObjects()
      if (children.length > 0) {
        this.canvas.remove(...children)
      }
    },
    // 保存按钮点击
    tapSaveBtn() {
      this.canvas.clone(cvs => {
        //遍历所有对对象，获取最小坐标，最大坐标
        let top = 0
        let left = 0
        let width = this.canvas.width
        let height = this.canvas.height

        var objects = cvs.getObjects();
        if(objects.length > 0 ){
          var rect = objects[0].getBoundingRect();
          var minX = rect.left;
          var minY = rect.top;
          var maxX = rect.left + rect.width;
          var maxY = rect.top + rect.height;
          for(var i = 1; i<objects.length; i++){
            rect = objects[i].getBoundingRect();
            minX = Math.min(minX, rect.left);
            minY= Math.min(minY, rect.top);
            maxX = Math.max(maxX, rect.left + rect.width);
            maxY= Math.max(maxY, rect.top + rect.height);
          }
          top = minY - 100
          left = minX - 100
          width = maxX - minX + 200
          height = maxY - minY + 200
          cvs.sendToBack(new fabric.Rect({
            left,
            top,
            width,
            height,
            stroke: 'rgba(0,0,0,0)',
            fill: this.bgColor,
            strokeWidth: 0
          }))
        }
        const dataURL = cvs.toDataURL({
          format: 'png',
          multiplier: cvs.getZoom(),
          left,
          top,
          width,
          height
        });
        const link = document.createElement('a');
        link.download = 'canvas.png';
        link.href = dataURL;
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
      })
    },
    // 计算画布移动之后的x坐标点
    getTransformedPosX(x) {
      let zoom = Number(this.canvas.getZoom())
      return (x - this.canvas.viewportTransform[4]) / zoom;
    },
    getTransformedPosY(y) {
      let zoom = Number(this.canvas.getZoom())
      return (y - this.canvas.viewportTransform[5]) / zoom;
    },
  },
  mounted() {
    // 初始化 画布
    this.initCanvas();
    // 默认开启画笔模式
    this.tapToolBtn("brush");
    // 初始化 画布 事件
    this.initCanvasEvent();
  },
};
</script>
<style lang="less" scoped>
.home {
  overflow: hidden;
  height: 100vh;
}
.btnwrap {
  height: 50px;
  display: flex;
  align-items: center;
  .btn-color {
    width: 40px;
    height: 40px;
    position: relative;
    border: 1px solid #999;
    margin-left: 20px;
    .color-picker {
      position: absolute;
      left: 0;
      top: 40px;
      z-index: 1000;
    }
  }
  .color-label {
    padding-left: 4px;
  }

  .brushWidth {
    margin-left: 10px;
  }

  .btn-tool {
    margin-left: 10px;
    padding: 0 4px;
    &:hover {
      cursor: pointer;
    }
    &.active {
      background: #2962ff;
      color: #fff;
      border-color: #2962ff;
    }
  }
}
</style>
