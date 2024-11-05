<template>
  <div class="container-fluid">
    <div class="row">
      <div class="col-2">
        <div id="pwm-time-plot"></div>
      </div>
      <div class="col-2">
        <div id="pwm-loop-plot"></div>
      </div>
      <div class="col-2">
        <div id="current-loop-plot"></div>
      </div>
      <div class="col-2">
        <div id="clark-loop-plot"></div>
      </div>
      <div class="col-2">
        <div id="park-loop-plot"></div>
      </div>
      <div class="col-2">
        <div id="six-area-plot"></div>
      </div>
      <div class="col-2">
        <div id="-repwm-time-plot"></div>
      </div>
      <div class="col-2">
        <div id="re-pwm-loop-plot"></div>
      </div>
      <div class="col-2">
        <div id="re-clark-loop-plot"></div>
      </div>
      <div class="col-2">
        <div id="re-park-loop-plot"></div>
      </div>
    </div>
  </div>
  <!-- <div class="">{{ CalcCurrent }}</div>。 -->
  <input class="input" type="range" v-model="nowAng" min="0" max="359" @input="AngChange"> {{ nowAng }}
  <input class="input" type="range" v-model="power" min="0" max="100"  @input="PowerChange"> {{ power }}
</template>

<script>
export default {
  data () {
  return {
    SixAreaHeight: window.innerWidth/5,
    SixAreaWidth: window.innerWidth/5,

    PwmTimerHeight: window.innerWidth/5,
    PwnTimer_LowStatusOffest: 5,
    PwnTimer_HighStatusOffest: 10,
    // PwmTimerWidth: window.innerWidth-500,
    PwmTimerWidth: window.innerWidth/5,
    PwnTimer_XLeft: 50,
    PwnTimer_TitleFontSize : 24,
    
    PwmLoop_Width: window.innerWidth/5,

    CurrentLoop_Width: window.innerWidth/5,
    ParkLoop_Width: window.innerWidth/5,
    ClarkLoop_Width: window.innerWidth/5,
    
    nowAng: 90,
    power: 100,
  }
  },
  computed:{
    CalcPWNTime() {
      return this.calcPWMTimeFunction(this.nowAng)
    },
    CalcPWMForce(){
      return this.calcPwmForceFunction(this.nowAng)
    },
    CalcCurrent(){
      return this.calcCurrentFunction(this.nowAng)
    },
    Clark(){
      var Current = this.CalcCurrent
      // [I_alpha, I_beta]
      return [
        Current[0] - 0.5*Current[1] - 0.5*Current[2],
        0.86602540378*Current[1] - 0.86602540378*Current[2]
      ]
    },

    testParkToPart(){
      var A, B, C, N, Part
      var U_alpha = this.Clark[0]
      var U_Beta = this.Clark[1]
      A = U_Beta > 0 ? 1 : 0
      B = -U_alpha*Math.sqrt(3) - U_Beta > 0 ? 1 : 0
      C =  U_alpha*Math.sqrt(3) - U_Beta > 0 ? 1 : 0
      // console.log(A, B, C)
      N = 4*C + 2*B + A
      if (N === 5) {Part=1}
      else if (N === 1) {Part=2}
      else if (N === 3) {Part=3}
      else if (N === 2) {Part=4}
      else if (N === 6) {Part=5}
      else if (N === 4) {Part=6}
      return [N, Part]
    },

    Park() {
      var clark_result = this.Clark
      // [I_d, I_q]
      // I_d 越大，花越多力氣在維持當前狀態，是浪費的
      // I_d 越大，代表花越多力氣在旋轉，效率越高
      return [
        clark_result[0] * Math.cos(this.nowAng*Math.PI/180.0) + clark_result[1] * Math.sin(this.nowAng*Math.PI/180.0),
        clark_result[0] * -Math.sin(this.nowAng*Math.PI/180.0) + clark_result[1] * Math.cos(this.nowAng*Math.PI/180.0),
      ]
    },
    R_Park(){
      // 理想上 旋轉時 Id 越小越好 最好為0
      var Id = 0
      var Iq = 1;
      // var Id = this.Park[0]
      // var Iq = this.Park[1]

      return [
        Id * Math.cos(this.nowAng*Math.PI/180.0) - Iq * Math.sin(this.nowAng*Math.PI/180.0),
        Id * Math.sin(this.nowAng*Math.PI/180.0) + Iq * Math.cos(this.nowAng*Math.PI/180.0),
      ]
    },

    R_Clark () {
      return this.calc_R_ClarkFunction(this.R_Park[0], this.R_Park[1])
    },


  },
  methods:{
    AngChange(){
      this.plotSixAreaAngWay()
      this.plotPwmTimePaths()
      this.plotPwmLoopAng()
      this.plotSixAreaForceWay()
      this.plotCurrentLoopAng()
      this.plotClarkLoopAng()
      this.plotParkLoopAng()
      this.plotReParkLoopAng()
    },
    PowerChange(){
      this.plotPwmTimePaths()
      this.plotSixAreaForceWay()
      this.replotPwmLoopCycle()
      this.rePlotCurrentCycle()
    },



    //? PWM 時序圖相關
    poltPwmTimeBasePlot(){
      const width = this.PwmTimerWidth
      const height = this.PwmTimerHeight
      const svg = d3.select('#pwm-time-plot').append('svg')
        .attr("id","pwm-time-plot-svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [0, 0, width, height])
        .attr("style", "max-width: 100%; height: auto;");

      const xLeft = this.PwnTimer_XLeft
      const TitleFontSize = this.PwnTimer_TitleFontSize

      svg.append("line").attr("x1", xLeft).attr("y1", height-this.PwnTimer_LowStatusOffest).attr("x2", width).attr("y2", height-this.PwnTimer_LowStatusOffest).attr("stroke", "black").attr("stroke-width", 2);
      svg.append("line").attr("x1", xLeft).attr("y1", height/3*2+this.PwnTimer_HighStatusOffest).attr("x2", width).attr("y2", height/3*2+this.PwnTimer_HighStatusOffest).attr("stroke", "gray").attr("stroke-width", 1);
      
      svg.append("line").attr("x1", xLeft).attr("y1", height/3*2-this.PwnTimer_LowStatusOffest).attr("x2", width).attr("y2", height/3*2-this.PwnTimer_LowStatusOffest).attr("stroke", "black").attr("stroke-width", 2);
      svg.append("line").attr("x1", xLeft).attr("y1", height/3+this.PwnTimer_HighStatusOffest).attr("x2", width).attr("y2", height/3+this.PwnTimer_HighStatusOffest).attr("stroke", "gray").attr("stroke-width", 1);
      
      svg.append("line").attr("x1", xLeft).attr("y1", height/3-this.PwnTimer_LowStatusOffest).attr("x2", width).attr("y2", height/3-this.PwnTimer_LowStatusOffest).attr("stroke", "black").attr("stroke-width", 2);
      svg.append("line").attr("x1", xLeft).attr("y1", this.PwnTimer_HighStatusOffest).attr("x2", width).attr("y2", this.PwnTimer_HighStatusOffest).attr("stroke", "gray").attr("stroke-width", 1);
      
      svg.append("text").attr("x", xLeft - TitleFontSize).attr("y", height/6+TitleFontSize/2).attr("text-anchor", "middle").attr("font-size", TitleFontSize+"px").attr("fill", "black").text("U");
      svg.append("text").attr("x", xLeft - TitleFontSize).attr("y", height/6*3+TitleFontSize/2).attr("text-anchor", "middle").attr("font-size", TitleFontSize+"px").attr("fill", "black").text("V");
      svg.append("text").attr("x", xLeft - TitleFontSize).attr("y", height/6*5+TitleFontSize/2).attr("text-anchor", "middle").attr("font-size", TitleFontSize+"px").attr("fill", "black").text("W");
      
      svg.append("path").attr("id", "u-pwm-time-path").attr("fill", "none").attr("stroke", "steelblue").attr("stroke-width", "2")
      svg.append("path").attr("id", "v-pwm-time-path").attr("fill", "none").attr("stroke", "green").attr("stroke-width", "2")
      svg.append("path").attr("id", "w-pwm-time-path").attr("fill", "none").attr("stroke", "red").attr("stroke-width", "2")
    },
    plotPwmTimePaths(){
      const width = this.PwmTimerWidth
      const height = this.PwmTimerHeight
      const xLeft = this.PwnTimer_XLeft
      const TitleFontSize = this.PwnTimer_TitleFontSize
      var L_type = ["u", "v", "w"]
      for (var index in L_type) {
        var timeStr = ""
        if (this.CalcPWNTime[index] === 100) {
          timeStr = `M${xLeft} ${height/3*index+this.PwnTimer_HighStatusOffest} L${width} ${height/3*parseInt(index)+this.PwnTimer_HighStatusOffest}`
        } else if (this.CalcPWNTime[index] === 0) {
          timeStr = `M${xLeft} ${height/3*(parseInt(index)+1)-this.PwnTimer_LowStatusOffest} L${width} ${height/3*(parseInt(index)+1)-this.PwnTimer_LowStatusOffest}`
        } else {
          var t1_Width = (width-xLeft)*(100-this.CalcPWNTime[index])/100/2
          timeStr = `M${xLeft} ${height/3*(parseInt(index)+1)-this.PwnTimer_LowStatusOffest} `
          timeStr += `L${xLeft+t1_Width} ${height/3*(parseInt(index)+1)-this.PwnTimer_LowStatusOffest} `
          timeStr += `L${xLeft+t1_Width} ${height/3*parseInt(index)+this.PwnTimer_HighStatusOffest} `
          timeStr += `L${width-t1_Width} ${height/3*parseInt(index)+this.PwnTimer_HighStatusOffest} `
          timeStr += `L${width-t1_Width} ${height/3*(parseInt(index)+1)-this.PwnTimer_LowStatusOffest} `
          timeStr += `L${width} ${height/3*(parseInt(index)+1)-this.PwnTimer_LowStatusOffest} `
        }
        document.getElementById(L_type[index]+"-pwm-time-path").setAttribute("d", timeStr)
      }
    },


    //? 象限圖相關
    plotSixAreaBasePlot(){
      const width = this.SixAreaHeight
      const height = this.SixAreaWidth
      const svg = d3.select('#six-area-plot').append('svg')
        .attr("id","six-area-plot-svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [0, 0, width, height])
        .attr("style", "max-width: 100%; height: auto;");

      const centerX = width / 2;
      const centerY = height / 2;
      const radius = 100; // 半徑

      // 計算六邊形的頂點
      const vertices = [];
      const sides = 6; // 六邊形
      const angleStep = 2 * Math.PI / sides;

      for (let i = sides; i > 0; i--) {
          const angle = i * angleStep;
          const x = centerX + radius * Math.cos(angle);
          const y = centerY + radius * Math.sin(angle);
          vertices.push([x, y]);
      }

      // 繪製六邊形
      svg.append("polygon")
        .attr("points", vertices.map(d => d.join(",")).join(" "))
        .attr("fill", "none")
        .attr("stroke", "gray")
        .attr("stroke-width", 2);
      
      // 繪製 6 條線條
      for (let i = 0; i < sides; i++) {
        const angle = i * angleStep;
        const x2 = centerX + radius * Math.cos(angle);
        const y2 = centerY + radius * Math.sin(angle);
        svg.append("line")
          .attr("x1", centerX)
          .attr("y1", centerY)
          .attr("x2", x2)
          .attr("y2", y2)
          .attr("stroke", "gray")
          .attr("stroke-width", 2-i%2);
      }
      // 在每個頂點上添加文字
      const labels = ["100 U", "110", "010 V", "011", "001 W", "101"]; // 文字標籤
      var textPosition = []
      vertices.forEach((vertex, i) => {
        if (i === 0) {
          textPosition.push([vertex[0]+25, vertex[1]+5])
        }
        else if (i ===1 | i===2) {
          textPosition.push([vertex[0], vertex[1]-5])
        }
        else if (i ===3) {
          textPosition.push([vertex[0]-25, vertex[1]+5])
        }
        else if (i ===4 | i===5) {
          textPosition.push([vertex[0], vertex[1]+15])
        }
        else {
          textPosition.push([vertex[0], vertex[1]])
        }
        svg.append("text")
          .attr("x", textPosition[i][0])
          .attr("y", textPosition[i][1]) // 調整文字的垂直位置
          .attr("text-anchor", "middle")
          .attr("font-size", "12px")
          .attr("fill", "black")
          .text(labels[i]);
      });

      const x3 = centerX + 150 * Math.cos(-this.nowAng/180*Math.PI);
      const y3 = centerY + 150 * Math.sin(-this.nowAng/180*Math.PI);
      svg.append("line")
          .attr("id", "six-area-plot-svg-ang-way")
          .attr("x1", centerX)
          .attr("y1", centerY)
          .attr("x2", x3)
          .attr("y2", y3)
          .attr("stroke-dasharray", "5,5")
          .attr("stroke", "red")
          .attr("stroke-width", 1);

      var forceList = this.CalcPWMForce
      svg.append("line")
          .attr("id", "six-area-plot-svg-force-way")
          .attr("x1", centerX)
          .attr("y1", centerY)
          .attr("x2", centerX+forceList[0]*100)
          .attr("y2", centerY+forceList[1]*100)
          .attr("marker-end", "url(#arrow)")
          .attr("stroke", "black")
          .attr("stroke-width", 3);


    },
    plotSixAreaAngWay(){
      const width = this.SixAreaHeight
      const height = this.SixAreaWidth
      const centerX = width / 2;
      const centerY = height / 2;
      const x2 = centerX + 150 * Math.cos(-this.nowAng/180*Math.PI);
      const y2 = centerY + 150 * Math.sin(-this.nowAng/180*Math.PI);
      var pathItem = document.getElementById("six-area-plot-svg-ang-way")
      pathItem.setAttribute("x2", x2)
      pathItem.setAttribute("y2", y2)
    },
    plotSixAreaForceWay(){
      const width = this.SixAreaHeight
      const height = this.SixAreaWidth
      const centerX = width / 2;
      const centerY = height / 2;
      var forceList = this.CalcPWMForce
      var pathItem = document.getElementById("six-area-plot-svg-force-way")
      pathItem.setAttribute("x2", centerX+forceList[0]*100)
      pathItem.setAttribute("y2", centerY+forceList[1]*100)
    },

    //? PWM 時序週期圖
    plotPwmLoopCycle() {
      const height = this.SixAreaHeight
      const width = this.PwmLoop_Width
      const svg = d3.select('#pwm-loop-plot').append('svg')
        .attr("id","pwm-loop-plot-svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [0, 0, width, height])
        .attr("style", "max-width: 100%; height: auto;");
      var L_xList = [...Array(360).keys()]
      var L_U_pwmTime = []
      var L_V_pwmTime = []
      var L_W_pwmTime = []
      for (var angIndex in L_xList) {
        var ansList = this.calcPWMTimeFunction(L_xList[angIndex])
        L_U_pwmTime.push({"ang":L_xList[angIndex], "value": ansList[0]})
        L_V_pwmTime.push({"ang":L_xList[angIndex], "value": ansList[1]})
        L_W_pwmTime.push({"ang":L_xList[angIndex], "value": ansList[2]})
      }
      
      const y = d3.scaleLinear([0, 100], [height-30, 30]);
      const x = d3.scaleLinear([0, 360], [40, width-40]);
      const line = d3.line()
        .x(d => x(d.ang))
        .y(d => y(d.value));
      svg.append("g")
        .attr("transform", `translate(0,${height - 30})`)
        .call(d3.axisBottom(x).ticks(width / 80).tickSizeOuter(0));
      svg.append("g")
        .attr("transform", `translate(${40},0)`)
        .call(d3.axisLeft(y).ticks(height / 40))
        .call(g => g.select(".domain").remove())
        .call(g => g.selectAll(".tick line").clone()
            .attr("x2", width - 40 - 40)
            .attr("stroke-opacity", 0.1))
        .call(g => g.append("text")
            .attr("x", -20)
            .attr("y", 10)
            .attr("fill", "currentColor")
            .attr("text-anchor", "start")
            .text("%"));

      svg.append("path")
        .attr("id", "pwm-loop-plot-svg-u")
        .attr("fill", "none")
        .attr("stroke", "steelblue")
        .attr("stroke-width", 1)
        .attr("d", line(L_U_pwmTime));
      svg.append("path")
        .attr("id", "pwm-loop-plot-svg-v")
        .attr("fill", "none")
        .attr("stroke", "green")
        .attr("stroke-width", 1)
        .attr("d", line(L_V_pwmTime));
      svg.append("path")
        .attr("id", "pwm-loop-plot-svg-w")
        .attr("fill", "none")
        .attr("stroke", "red")
        .attr("stroke-width", 1)
        .attr("d", line(L_W_pwmTime));


      svg.append("line")
        .attr("id", "pwm-loop-plot-svg-ang")
        .attr("x1", x(this.nowAng)) 
        .attr("y1", y(0))
        .attr("x2", x(this.nowAng))
        .attr("y2", y(100))
        .attr("stroke-dasharray", "2,2")
        .attr("stroke", "black")
        .attr("stroke-width", 1);
    },
    plotPwmLoopAng(){
      var pathItem = document.getElementById("pwm-loop-plot-svg-ang")
      const width = this.PwmLoop_Width
      const x = d3.scaleLinear([0, 360], [40, width-40]);
      pathItem.setAttribute("x1", x(this.nowAng)) 
      pathItem.setAttribute("x2", x(this.nowAng)) 
    },
    replotPwmLoopCycle(){
      const height = this.SixAreaHeight
      const width = this.PwmLoop_Width
      var L_xList = [...Array(360).keys()]
      var L_U_pwmTime = []
      var L_V_pwmTime = []
      var L_W_pwmTime = []
      for (var angIndex in L_xList) {
        var ansList = this.calcPWMTimeFunction(L_xList[angIndex])
        L_U_pwmTime.push({"ang":L_xList[angIndex], "value": ansList[0]})
        L_V_pwmTime.push({"ang":L_xList[angIndex], "value": ansList[1]})
        L_W_pwmTime.push({"ang":L_xList[angIndex], "value": ansList[2]})
      }
      const y = d3.scaleLinear([0, 100], [height-30, 30]);
      const x = d3.scaleLinear([0, 360], [40, width-40]);
      const line = d3.line()
        .x(d => x(d.ang))
        .y(d => y(d.value));
      var pathItem = document.getElementById("pwm-loop-plot-svg-u")
      pathItem.setAttribute("d", line(L_U_pwmTime)) 
      var pathItem = document.getElementById("pwm-loop-plot-svg-v")
      pathItem.setAttribute("d", line(L_V_pwmTime)) 
      var pathItem = document.getElementById("pwm-loop-plot-svg-w")
      pathItem.setAttribute("d", line(L_W_pwmTime)) 
    },

    //? 電流時序圖
    plotCurrentLoopCycle() {
      const height = this.SixAreaHeight
      const width = this.CurrentLoop_Width
      const svg = d3.select('#current-loop-plot').append('svg')
        .attr("id","current-loop-plot-svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [0, 0, width, height])
        .attr("style", "max-width: 100%; height: auto;");
      const y = d3.scaleLinear([-1, 1], [height-30, 30]);
      const x = d3.scaleLinear([0, 360], [40, width-40]);
      const line = d3.line()
        .x(d => x(d.ang))
        .y(d => y(d.value));
      svg.append("g")
        .attr("transform", `translate(0,${height - 30})`)
        .call(d3.axisBottom(x).ticks(width / 80).tickSizeOuter(0));
      svg.append("g")
        .attr("transform", `translate(${40},0)`)
        .call(d3.axisLeft(y).ticks(height / 40))
        .call(g => g.select(".domain").remove())
        .call(g => g.selectAll(".tick line").clone()
            .attr("x2", width - 40 - 40)
            .attr("stroke-opacity", 0.1))
        .call(g => g.append("text")
            .attr("x", -20)
            .attr("y", 10)
            .attr("fill", "currentColor")
            .attr("text-anchor", "start")
            .text("A"));
            var L_U_pwmTime = []
      var L_xList = [...Array(360).keys()]
      var L_V_current = []
      var L_W_current = []
      var L_U_current = []
      for (var angIndex in L_xList) {
        var ansList = this.calcCurrentFunction(L_xList[angIndex])
        L_U_current.push({"ang":L_xList[angIndex], "value": ansList[0]})
        L_V_current.push({"ang":L_xList[angIndex], "value": ansList[1]})
        L_W_current.push({"ang":L_xList[angIndex], "value": ansList[2]})
      }
      svg.append("path")
        .attr("id", "current-loop-plot-svg-u")
        .attr("fill", "none")
        .attr("stroke", "steelblue")
        .attr("stroke-width", 1)
        .attr("d", line(L_U_current));
      svg.append("path")
        .attr("id", "current-loop-plot-svg-v")
        .attr("fill", "none")
        .attr("stroke", "green")
        .attr("stroke-width", 1)
        .attr("d", line(L_V_current));
      svg.append("path")
        .attr("id", "current-loop-plot-svg-w")
        .attr("fill", "none")
        .attr("stroke", "red")
        .attr("stroke-width", 1)
        .attr("d", line(L_W_current));

      svg.append("line")
        .attr("id", "current-loop-plot-svg-ang")
        .attr("x1", x(this.nowAng)) 
        .attr("y1", y(-1))
        .attr("x2", x(this.nowAng))
        .attr("y2", y(1))
        .attr("stroke-dasharray", "2,2")
        .attr("stroke", "black")
        .attr("stroke-width", 1);
    },
    plotCurrentLoopAng(){
      var pathItem = document.getElementById("current-loop-plot-svg-ang")
      const width = this.CurrentLoop_Width
      const x = d3.scaleLinear([0, 360], [40, width-40]);
      pathItem.setAttribute("x1", x(this.nowAng)) 
      pathItem.setAttribute("x2", x(this.nowAng)) 
    },
    rePlotCurrentCycle(){
      const height = this.SixAreaHeight
      const width = this.CurrentLoop_Width
      const y = d3.scaleLinear([-1, 1], [height-30, 30]);
      const x = d3.scaleLinear([0, 360], [40, width-40]);
      const line = d3.line()
        .x(d => x(d.ang))
        .y(d => y(d.value));

      var L_xList = [...Array(360).keys()]
      var L_V_current = []
      var L_W_current = []
      var L_U_current = []
      for (var angIndex in L_xList) {
        var ansList = this.calcCurrentFunction(L_xList[angIndex])
        L_U_current.push({"ang":L_xList[angIndex], "value": ansList[0]})
        L_V_current.push({"ang":L_xList[angIndex], "value": ansList[1]})
        L_W_current.push({"ang":L_xList[angIndex], "value": ansList[2]})
      }

      var pathItem = document.getElementById("current-loop-plot-svg-u")
      pathItem.setAttribute("d", line(L_U_current)) 
      var pathItem = document.getElementById("current-loop-plot-svg-v")
      pathItem.setAttribute("d", line(L_V_current)) 
      var pathItem = document.getElementById("current-loop-plot-svg-w")
      pathItem.setAttribute("d", line(L_W_current)) 
    },

    //? Clark 轉換後時序圖
    plotClarkLoopCycle() {
      const height = this.SixAreaHeight
      const width = this.ParkLoop_Width
      const svg = d3.select('#clark-loop-plot').append('svg')
        .attr("id","clark-loop-plot-svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [0, 0, width, height])
        .attr("style", "max-width: 100%; height: auto;");

      var L_xList = [...Array(360).keys()]
      var L_I_alpha = []
      var L_I_bera = []
      
      var maxY = 0
      var minY = 0

      for (var angIndex in L_xList) {
        var currentList = this.calcCurrentFunction(L_xList[angIndex])
        var ansList = this.calcClarkFunction(currentList[0],currentList[1],currentList[2])
        L_I_alpha.push({"ang":L_xList[angIndex], "value": ansList[0]})
        L_I_bera.push({"ang":L_xList[angIndex], "value": ansList[1]})
        maxY = Math.max(maxY, ansList[0], ansList[1])
        minY = Math.min(minY, ansList[0], ansList[1])
      }
      const y = d3.scaleLinear([minY*1.1, maxY*1.1], [height-30, 30]);
      const x = d3.scaleLinear([0, 360], [40, width-40]);
      const line = d3.line()
        .x(d => x(d.ang))
        .y(d => y(d.value));

      svg.append("g")
        .attr("transform", `translate(0,${height - 30})`)
        .call(d3.axisBottom(x).ticks(width / 80).tickSizeOuter(0));

      svg.append("g")
        .attr("transform", `translate(${40},0)`)
        .call(d3.axisLeft(y).ticks(height / 40))
        .call(g => g.select(".domain").remove())
        .call(g => g.selectAll(".tick line").clone()
            .attr("x2", width - 40 - 40)
            .attr("stroke-opacity", 0.1))
        .call(g => g.append("text")
            .attr("x", -20)
            .attr("y", 10)
            .attr("fill", "currentColor")
            .attr("text-anchor", "start")
            .text("A"));

      svg.append("path")
        .attr("id", "current-loop-plot-svg-i-alpha")
        .attr("fill", "none")
        .attr("stroke", "steelblue")
        .attr("stroke-width", 1)
        .attr("d", line(L_I_alpha));
      svg.append("path")
        .attr("id", "current-loop-plot-svg-i-beta")
        .attr("fill", "none")
        .attr("stroke", "green")
        .attr("stroke-width", 1)
        .attr("d", line(L_I_bera));

      svg.append("line")
        .attr("id", "clark-loop-plot-svg-ang")
        .attr("x1", x(this.nowAng)) 
        .attr("y1", y(minY))
        .attr("x2", x(this.nowAng))
        .attr("y2", y(maxY))
        .attr("stroke-dasharray", "2,2")
        .attr("stroke", "black")
        .attr("stroke-width", 1);
    },
    plotClarkLoopAng(){
      var pathItem = document.getElementById("clark-loop-plot-svg-ang")
      const width = this.ParkLoop_Width
      const x = d3.scaleLinear([0, 360], [40, width-40]);
      pathItem.setAttribute("x1", x(this.nowAng)) 
      pathItem.setAttribute("x2", x(this.nowAng)) 
    },

    //? Park 轉換後時序圖
    plotParkLoopCycle() {
      const height = this.SixAreaHeight
      const width = this.ParkLoop_Width
      const svg = d3.select('#park-loop-plot').append('svg')
        .attr("id","park-loop-plot-svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [0, 0, width, height])
        .attr("style", "max-width: 100%; height: auto;");

      var L_xList = [...Array(360).keys()]
      var L_I_d = []
      var L_I_q = []
      
      var maxY = 0
      var minY = 0

      for (var angIndex in L_xList) {
        var currentList = this.calcCurrentFunction(L_xList[angIndex])
        var clarkList = this.calcClarkFunction(currentList[0],currentList[1],currentList[2])
        var ansList = this.calcParkFunction(clarkList[0],clarkList[1],L_xList[angIndex])
        L_I_d.push({"ang":L_xList[angIndex], "value": ansList[0]})
        L_I_q.push({"ang":L_xList[angIndex], "value": ansList[1]})
        maxY = Math.max(maxY, ansList[0], ansList[1])
        minY = Math.min(minY, ansList[0], ansList[1])
      }
      const y = d3.scaleLinear([minY*1.1, maxY*1.1], [height-30, 30]);
      const x = d3.scaleLinear([0, 360], [40, width-40]);
      const line = d3.line()
        .x(d => x(d.ang))
        .y(d => y(d.value));

      svg.append("g")
        .attr("transform", `translate(0,${height - 30})`)
        .call(d3.axisBottom(x).ticks(width / 80).tickSizeOuter(0));

      svg.append("g")
        .attr("transform", `translate(${40},0)`)
        .call(d3.axisLeft(y).ticks(height / 40))
        .call(g => g.select(".domain").remove())
        .call(g => g.selectAll(".tick line").clone()
            .attr("x2", width - 40 - 40)
            .attr("stroke-opacity", 0.1))
        .call(g => g.append("text")
            .attr("x", -20)
            .attr("y", 10)
            .attr("fill", "currentColor")
            .attr("text-anchor", "start")
            .text("A"));

      svg.append("path")
        .attr("id", "park-loop-plot-svg-i-d")
        .attr("fill", "none")
        .attr("stroke", "steelblue")
        .attr("stroke-width", 1)
        .attr("d", line(L_I_d))
      svg.append("text").attr("x", x(360)).attr("y", y(maxY*1.05)).attr("text-anchor", "middle").attr("font-size", "18px").attr("fill", "steelblue").text("I_d");

      svg.append("path")
        .attr("id", "park-loop-plot-svg-i-q")
        .attr("fill", "none")
        .attr("stroke", "green")
        .attr("stroke-width", 1)
        .attr("d", line(L_I_q));
      svg.append("text").attr("x", x(360)).attr("y", y(minY*0.95)).attr("text-anchor", "middle").attr("font-size", "18px").attr("fill", "green").text("I_q");

      svg.append("line")
        .attr("id", "park-loop-plot-svg-ang")
        .attr("x1", x(this.nowAng)) 
        .attr("y1", y(minY))
        .attr("x2", x(this.nowAng))
        .attr("y2", y(maxY))
        .attr("stroke-dasharray", "2,2")
        .attr("stroke", "black")
        .attr("stroke-width", 1);
    },
    plotParkLoopAng(){
      var pathItem = document.getElementById("park-loop-plot-svg-ang")
      const width = this.ParkLoop_Width
      const x = d3.scaleLinear([0, 360], [40, width-40]);
      pathItem.setAttribute("x1", x(this.nowAng)) 
      pathItem.setAttribute("x2", x(this.nowAng)) 
    },

    //? 反 Park 轉換後時序圖
    plotReParkLoopCycle() {
      const height = this.SixAreaHeight
      const width = this.ParkLoop_Width
      const svg = d3.select('#re-park-loop-plot').append('svg')
        .attr("id","re-park-loop-plot-svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [0, 0, width, height])
        .attr("style", "max-width: 100%; height: auto;");

      var L_xList = [...Array(360).keys()]
      var L_rPark_alpha = []
      var L_rPark_beta = []
      var maxY = 0
      var minY = 0

      for (var angIndex in L_xList) {
        var reParkList = this.calc_R_ParkFunction(-1,1,L_xList[angIndex])
        L_rPark_alpha.push({"ang":L_xList[angIndex], "value": reParkList[0]})
        L_rPark_beta.push({"ang":L_xList[angIndex], "value": reParkList[1]})
        maxY = Math.max(maxY, reParkList[0], reParkList[1])
        minY = Math.min(minY, reParkList[0], reParkList[1])
      }
      const y = d3.scaleLinear([minY*1.1, maxY*1.1], [height-30, 30]);
      const x = d3.scaleLinear([0, 360], [40, width-40]);
      const line = d3.line()
        .x(d => x(d.ang))
        .y(d => y(d.value));

      svg.append("g")
        .attr("transform", `translate(0,${height - 30})`)
        .call(d3.axisBottom(x).ticks(width / 80).tickSizeOuter(0));

      svg.append("g")
        .attr("transform", `translate(${40},0)`)
        .call(d3.axisLeft(y).ticks(height / 40))
        .call(g => g.select(".domain").remove())
        .call(g => g.selectAll(".tick line").clone()
            .attr("x2", width - 40 - 40)
            .attr("stroke-opacity", 0.1))
        .call(g => g.append("text")
            .attr("x", -20)
            .attr("y", 10)
            .attr("fill", "currentColor")
            .attr("text-anchor", "start")
            .text("A"));

      svg.append("path")
        .attr("id", "re-park-loop-plot-svg-i-beta")
        .attr("fill", "none")
        .attr("stroke", "red")
        .attr("stroke-width", 1)
        .attr("d", line(L_rPark_beta));

      svg.append("path")
        .attr("id", "re-park-loop-plot-svg-i-alpha")
        .attr("fill", "none")
        .attr("stroke", "black")
        .attr("stroke-width", 1)
        .attr("d", line(L_rPark_alpha));

      svg.append("line")
        .attr("id", "re-park-loop-plot-svg-ang")
        .attr("x1", x(this.nowAng)) 
        .attr("y1", y(minY))
        .attr("x2", x(this.nowAng))
        .attr("y2", y(maxY))
        .attr("stroke-dasharray", "2,2")
        .attr("stroke", "black")
        .attr("stroke-width", 1);
    },
    plotReParkLoopAng(){
      var pathItem = document.getElementById("re-park-loop-plot-svg-ang")
      const width = this.ParkLoop_Width
      const x = d3.scaleLinear([0, 360], [40, width-40]);
      pathItem.setAttribute("x1", x(this.nowAng)) 
      pathItem.setAttribute("x2", x(this.nowAng)) 
    },

    //? 反 Clark 轉換後時序圖
    plotReClarkLoopCycle() {
      const height = this.SixAreaHeight
      const width = this.CurrentLoop_Width
      const svg = d3.select('#re-clark-loop-plot').append('svg')
        .attr("id","re-clark-loop-plot-svg")
        .attr("width", width)
        .attr("height", height)
        .attr("viewBox", [0, 0, width, height])
        .attr("style", "max-width: 100%; height: auto;");
      const y = d3.scaleLinear([-1, 1], [height-30, 30]);
      const x = d3.scaleLinear([0, 360], [40, width-40]);
      const line = d3.line()
        .x(d => x(d.ang))
        .y(d => y(d.value));
      svg.append("g")
        .attr("transform", `translate(0,${height - 30})`)
        .call(d3.axisBottom(x).ticks(width / 80).tickSizeOuter(0));
      svg.append("g")
        .attr("transform", `translate(${40},0)`)
        .call(d3.axisLeft(y).ticks(height / 40))
        .call(g => g.select(".domain").remove())
        .call(g => g.selectAll(".tick line").clone()
            .attr("x2", width - 40 - 40)
            .attr("stroke-opacity", 0.1))
        .call(g => g.append("text")
            .attr("x", -20)
            .attr("y", 10)
            .attr("fill", "currentColor")
            .attr("text-anchor", "start")
            .text("A"));
            var L_U_pwmTime = []
      var L_xList = [...Array(360).keys()]
      var L_V_current = []
      var L_W_current = []
      var L_U_current = []
      for (var angIndex in L_xList) {
        var ReparkResult = this.calc_R_ParkFunction(-1,1,L_xList[angIndex])
        var ansList = this.calc_R_ClarkFunction(ReparkResult[0],ReparkResult[1])
        L_U_current.push({"ang":L_xList[angIndex], "value": ansList[0]})
        L_V_current.push({"ang":L_xList[angIndex], "value": ansList[1]})
        L_W_current.push({"ang":L_xList[angIndex], "value": ansList[2]})
      }
      svg.append("path")
        .attr("id", "re-clark-loop-plot-svg-u")
        .attr("fill", "none")
        .attr("stroke", "steelblue")
        .attr("stroke-width", 1)
        .attr("d", line(L_U_current));
      svg.append("path")
        .attr("id", "re-clark-loop-plot-svg-v")
        .attr("fill", "none")
        .attr("stroke", "green")
        .attr("stroke-width", 1)
        .attr("d", line(L_V_current));
      svg.append("path")
        .attr("id", "re-clark-loop-plot-svg-w")
        .attr("fill", "none")
        .attr("stroke", "red")
        .attr("stroke-width", 1)
        .attr("d", line(L_W_current));

      svg.append("line")
        .attr("id", "re-clark-loop-plot-svg-ang")
        .attr("x1", x(this.nowAng)) 
        .attr("y1", y(-1))
        .attr("x2", x(this.nowAng))
        .attr("y2", y(1))
        .attr("stroke-dasharray", "2,2")
        .attr("stroke", "black")
        .attr("stroke-width", 1);
    },


    calcUVMTimeFunction(angIN){
      return [
        Math.sin( ( 60 - (angIN % 60)) * Math.PI / 180.0) * 100 * this.power / 100,
        Math.sin( (angIN % 60) * Math.PI / 180.0) * 100 * this.power/ 100
      ]
    },
    calcCurrentFunction(angIN){
      var UVM = this.calcUVMTimeFunction(angIN)
      var U_current, V_current, W_current
      if (angIN < 60.) {
        var time_100 = UVM[0]
        var time_110 = UVM[1]
        var time_000_111 = 100-UVM[0]-UVM[1]
        U_current = time_100*1/100+time_110*0.5/100
        V_current = time_100*-0.5/100+time_110*0.5/100
        // U_current = (time_100-time_000_111/2)*1/100+(time_110-time_000_111/2)*0.5/100
        // V_current = (time_100-time_000_111/2)*-0.5/100+(time_110-time_000_111/2)*0.5/100
      }
      else if (angIN >= 60. & angIN < 120) {
        var time_110 = UVM[0]
        var time_010 = UVM[1]
        var time_000_111 = 100-UVM[0]-UVM[1]
        U_current = time_110*0.5/100+time_010*-0.5/100
        V_current = time_110*0.5/100+time_010*1/100
        // U_current = (time_110-time_000_111/2)*0.5/100+(time_010-time_000_111/2)*-0.5/100
        // V_current = (time_110-time_000_111/2)*0.5/100+(time_010-time_000_111/2)*1/100
      }
      else if (angIN >= 120. & angIN < 180) {
        var time_010 = UVM[0]
        var time_011 = UVM[1]
        var time_000_111 = 100-UVM[0]-UVM[1]
        U_current = time_010*-0.5/100+time_011*-1/100
        V_current = time_010*1/100+time_011*0.5/100
        // U_current = (time_010-time_000_111/2)*-0.5/100+(time_011-time_000_111/2)*-1/100
        // V_current = (time_010-time_000_111/2)*1/100+(time_011-time_000_111/2)*0.5/100
      }
      else if (angIN >= 180. & angIN < 240) {
        var time_011 = UVM[0]
        var time_001 = UVM[1]
        var time_000_111 = 100-UVM[0]-UVM[1]
        U_current = time_011*-1/100+time_001*-0.5/100
        V_current = time_011*0.5/100+time_001*-0.5/100
        // U_current = (time_011-time_000_111/2)*-1/100+(time_001-time_000_111/2)*-0.5/100
        // V_current = (time_011-time_000_111/2)*0.5/100+(time_001-time_000_111/2)*-0.5/100
      }
      else if (angIN >= 240. & angIN < 300) {
        var time_001 = UVM[0]
        var time_101 = UVM[1]
        var time_000_111 = 100-UVM[0]-UVM[1]
        U_current = time_001*-0.5/100+time_101*0.5/100
        V_current = time_001*-0.5/100+time_101*-1/100
        // U_current = (time_001-time_000_111/2)*-0.5/100+(time_101-time_000_111/2)*0.5/100
        // V_current = (time_001-time_000_111/2)*-0.5/100+(time_101-time_000_111/2)*-1/100
      }
      else if (angIN >= 300. & angIN < 360) {
        var time_101 =  UVM[0]
        var time_100 = UVM[1]
        var time_000_111 = 100-UVM[0]-UVM[1]
        U_current = time_101*0.5/100+time_100*1/100
        V_current = time_101*-1/100+time_100*-0.5/100
        // U_current = (time_101-time_000_111/2)*0.5/100+(time_100-time_000_111/2)*1/100
        // V_current = (time_101-time_000_111/2)*-1/100+(time_100-time_000_111/2)*-0.5/100
      }
      W_current = -(U_current+V_current)
      return [U_current, V_current, W_current]
    },
    calcPWMTimeFunction(angIN){
      var UVM = this.calcUVMTimeFunction(angIN)
      if (angIN < 60.) {
        var time_100 = UVM[0]
        var time_110 = UVM[1]
        var time_111 = 100-UVM[0]-UVM[1]
        // return [100, 100.-time_100, 100-(time_100+time_110)]
        return [time_100+time_110+time_111/2, time_110+time_111/2, time_111/2]
      }
      else if (angIN >= 60. & angIN < 120) {
        var time_110 = UVM[0]
        var time_010 = UVM[1]
        var time_000 = 100-UVM[0]-UVM[1]
        // return [time_110, time_110+time_010, 0]
        return [time_110+time_000/2, time_110+time_010+time_000/2, time_000/2]
      }
      else if (angIN >= 120. & angIN < 180) {
        var time_010 = UVM[0]
        var time_011 = UVM[1]
        var time_111 = 100-UVM[0]-UVM[1]
        // return [100-(time_010+time_011), 100, 100-time_010]
        return [time_111/2, time_010+time_011+time_111/2, time_011+time_111/2]
      }
      else if (angIN >= 180. & angIN < 240) {
        var time_011 = UVM[0]
        var time_001 = UVM[1]
        var time_000 = 100-UVM[0]-UVM[1]
        // return [0, time_011, time_001+time_011]
        return [time_000/2, time_011+time_000/2, time_011+time_001+time_000/2]
      }
      else if (angIN >= 240. & angIN < 300) {
        var time_001 = UVM[0]
        var time_101 = UVM[1]
        var time_111 = 100-UVM[0]-UVM[1]
        return [time_101+time_111/2, time_111/2, time_001+time_101+time_111/2]
      }
      else if (angIN >= 300. & angIN < 360) {
        var time_101 = UVM[0]
        var time_100 = UVM[1]
        var time_000 = 100-UVM[0]-UVM[1]
        return [time_101+time_100+time_000/2, time_000/2, time_101+time_000/2]
      }
      return []
    },
    calcPwmForceFunction(angIN){
      var UVM = this.calcUVMTimeFunction(angIN)
      if (angIN < 60.) {
        //? 100 -> 110 -> 111 -> 110 -> 100
        var time_100 = UVM[0]
        var force_100 = [1*time_100/100, 0*time_100/100]
        var time_110 = UVM[1]
        var force_110 = [Math.cos(60*Math.PI/180.0)*time_110/100, Math.sin(60*Math.PI/180.0)*time_110/100]
        return [force_100[0]+force_110[0],-force_100[1]-force_110[1]]
      }
      else if (angIN >= 60. & angIN < 120) {
        var time_110 = UVM[0]
        var force_110 = [Math.cos(60*Math.PI/180.0)*time_110/100, Math.sin(60*Math.PI/180.0)*time_110/100]
        var time_010 = UVM[1]
        var force_010 = [Math.cos(120*Math.PI/180.0)*time_010/100, Math.sin(120*Math.PI/180.0)*time_010/100]
        return [force_010[0]+force_110[0],-force_010[1]-force_110[1]]
      }
      else if (angIN >= 120. & angIN < 180) {
        var time_010 = UVM[0]
        var force_010 = [Math.cos(120*Math.PI/180.0)*time_010/100, Math.sin(120*Math.PI/180.0)*time_010/100]
        var time_011 = UVM[1]
        var force_011 = [Math.cos(180*Math.PI/180.0)*time_011/100, Math.sin(180*Math.PI/180.0)*time_011/100]
        return [force_010[0]+force_011[0],-force_010[1]-force_011[1]]
      }
      else if (angIN >= 180. & angIN < 240) {
        var time_011 = UVM[0]
        var force_011 = [Math.cos(180*Math.PI/180.0)*time_011/100, Math.sin(180*Math.PI/180.0)*time_011/100]
        var time_001 = UVM[1]
        var force_001 = [Math.cos(240*Math.PI/180.0)*time_001/100, Math.sin(240*Math.PI/180.0)*time_001/100]
        return [force_011[0]+force_001[0],-force_011[1]-force_001[1]]
      }
      else if (angIN >= 240. & angIN < 300) {
        var time_001 = UVM[0]
        var force_001 = [Math.cos(240*Math.PI/180.0)*time_001/100, Math.sin(240*Math.PI/180.0)*time_001/100]
        var time_101 = UVM[1]
        var force_101 = [Math.cos(300*Math.PI/180.0)*time_101/100, Math.sin(300*Math.PI/180.0)*time_101/100]
        return [force_001[0]+force_101[0],-force_001[1]-force_101[1]]
      }
      else if (angIN >= 300. & angIN < 360) {
        var time_101 =  UVM[0]
        var force_101 = [Math.cos(300*Math.PI/180.0)*time_101/100, Math.sin(300*Math.PI/180.0)*time_101/100]
        var time_100 = UVM[1]
        var force_100 = [1*time_100/100, 0*time_100/100]
        return [force_101[0]+force_100[0],-force_101[1]-force_100[1]]
      }
      return [0,0]
    },

    calcClarkFunction(Ia, Ib, Ic){
      // [I_alpha, I_beta]
      return [
        Ia - Ib*0.5 - Ic*0.5,
        Ib*0.86602540378 - Ic*0.86602540378
      ]
    },
    calcParkFunction(I_alpha, I_beta, ang){
      // [I_d, I_q]
      return [
        I_alpha * Math.cos(ang*Math.PI/180.0) + I_beta * Math.sin(ang*Math.PI/180.0),
        I_alpha * -Math.sin(ang*Math.PI/180.0) + I_beta * Math.cos(ang*Math.PI/180.0),
      ]
    },
    calc_R_ParkFunction(Id, Iq, angIN){
      return [
        Id * Math.cos(angIN*Math.PI/180.0) - Iq * Math.sin(angIN*Math.PI/180.0),
        Id * Math.sin(angIN*Math.PI/180.0) + Iq * Math.cos(angIN*Math.PI/180.0),
      ]
    },
    calc_R_ClarkFunction(U_alpha, U_Beta){
      // https://jianwei.fun/?p=2113

      var Va, Vb, Vc;
      Va = U_alpha/3*2;
      Vb = (-0.5*U_alpha+0.86602540378*U_Beta)/3*2
      Vc = (-0.5*U_alpha-0.86602540378*U_Beta)/3*2


      var U_a = U_alpha*Math.sqrt(3)
      var U_b = -U_Beta
      var X, Y, Z, Part
      var phA, phB, phC
      X = U_b
      Y = (U_b + U_a) / 2
      Z = (U_b - U_a) / 2
      
      if (Y<0) {
        if (Z<0){Part = 5;}
        else {
          if (X<=0) {Part = 4;}
          else {Part = 3;}
        }
      }
      else {
        if (Z>=0){Part = 2;}
        else if (X<=0){Part = 6;}
        else {Part = 1;}
      }             

      if (Part === 1 | Part === 4) {
      }




      return [Va, Vb, Vc, Part, N]
    },
  },
  mounted(){
    this.plotSixAreaBasePlot()
    this.plotSixAreaAngWay()
    this.poltPwmTimeBasePlot()
    this.plotPwmTimePaths()
    this.plotPwmLoopCycle()
    this.plotPwmLoopAng()
    this.plotCurrentLoopCycle()
    this.plotClarkLoopCycle()
    this.plotParkLoopCycle()
    this.plotReParkLoopCycle()
    this.plotReClarkLoopCycle()
    window.test = this
  },
}
</script>
<style lang="postcss">
/* https://github.com/csstools/postcss-plugins/tree/main/plugins/postcss-nesting */
</style>