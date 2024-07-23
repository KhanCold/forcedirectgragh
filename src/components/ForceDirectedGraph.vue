<template>
  <div class="canvas-container" ref="canvasContainer">
    <canvas ref="canvas" width="800" height="600"></canvas>
  </div>
</template>

<script>
import data from "@/data.json";
//new branch

export default {
  name: "ForceDirectedGraph",
  data() {
    return {
      // 初始化节点，设置随机位置和速度，固定位置为null
      nodes: data.nodes.map(node => ({//展开节点数据，添加随机位置和速度
        ...node,
        x: Math.random() * 800, // 随机位置
        y: Math.random() * 600,
        vx: 0,  // 速度
        vy: 0,
        fx: null,// 是否固定位置
        fy: null
      })),
      // 初始化连接
      links: data.links,
      width: 800,
      height: 600,
      draggingNode: null,  // 当前拖动的节点
      transform: {// 画布平移和缩放
        x: 0,
        y: 0,
        k: 1
      },
      isPanning: false,  // 是否处于拖动画布状态
      lastMousePos: { x: 0, y: 0 }  // 最后一次鼠标位置
    };
  },
  mounted() {
    this.createForceDirectedGraph();
  },
  methods: {
    createForceDirectedGraph() { // 创建力导向图
      const canvas = this.$refs.canvas;
      const ctx = canvas.getContext("2d");//获取 2D 绘图上下文，以便在 canvas 上进行 2D 绘图操作
      const alpha = 0.1; // 阻尼系数

      // 为画布添加鼠标事件监听器
      canvas.addEventListener("mousedown", this.onMouseDown);
      canvas.addEventListener("mousemove", this.onMouseMove);
      canvas.addEventListener("mouseup", this.onMouseUp);
      canvas.addEventListener("wheel", this.onMouseWheel);

      // 模拟步骤函数
      //原始数据格式node:{id:xx,group:xx}
      //links:{source:xx,target:xx,value:xx}
      const simulationStep = () => {
        // 计算连线的力
        this.links.forEach(link => {
          const source = this.nodes.find(n => n.id === link.source);
          const target = this.nodes.find(n => n.id === link.target);
          const dx = target.x - source.x;
          const dy = target.y - source.y;
          const distance = Math.sqrt(dx * dx + dy * dy);//用勾股定理计算两点间距离
          const force = (distance - 100) / distance * 0.1; // 理想距离是100 这里的力是吸引力，如果距离大于100，力为正，表示吸引；如果距离小于100，力为负，表示排斥。

          const fx = force * dx;
          const fy = force * dy;

          // 如果节点没有固定位置，更新速度 
          // 源节点的速度增加力的分量，目标节点的速度减少力的分量，以模拟节点之间的相互作用。
          if (!source.fx) {
            source.vx += fx;
            source.vy += fy;
          }
          if (!target.fx) {
            target.vx -= fx;
            target.vy -= fy;
          }
        });

        // 防止节点重叠逻辑
        this.nodes.forEach((nodeA, i) => {
          this.nodes.forEach((nodeB, j) => {
            if (i >= j) return; // 避免重复计算
            const dx = nodeB.x - nodeA.x;
            const dy = nodeB.y - nodeA.y;
            const distance = Math.sqrt(dx * dx + dy * dy);
            if (distance < 20) { // 最小距离20 如果节点之间的距离小于 20（最小距离），则需要调整节点位置以防止重叠。
              // 计算节点间的角度 angle，使用 Math.atan2(dy, dx)。
              const angle = Math.atan2(dy, dx);
              // 计算需要移动的距离 moveDistance，即 (20 - distance) / 2，表示每个节点需要移动的距离。
              const moveDistance = (20 - distance) / 2;
              // 计算每个节点在 x 轴和 y 轴上需要移动的距离 moveX 和 moveY。
              const moveX = Math.cos(angle) * moveDistance;
              const moveY = Math.sin(angle) * moveDistance;
              
              //如果节点 nodeA 没有被固定（fx 为 null），则更新其位置，使其向远离 nodeB 的方向移动。
              // 如果节点 nodeB 没有被固定（fx 为 null），则更新其位置，使其向远离 nodeA 的方向移动。
              if (!nodeA.fx) {
                nodeA.x -= moveX;
                nodeA.y -= moveY;
              }
              if (!nodeB.fx) {
                nodeB.x += moveX;
                nodeB.y += moveY;
              }
            }
          });
        });

        // 更新节点位置和速度
        this.nodes.forEach(node => {
          if (!node.fx) {
            node.vx *= alpha;//乘上阻尼系数，用于减缓节点的速度，使其趋于稳定
            node.vy *= alpha;

            node.x += node.vx; // 更新节点位置
            node.y += node.vy;

            // 防止节点超出画布边界
            node.x = Math.max(0, Math.min(this.width, node.x));
            node.y = Math.max(0, Math.min(this.height, node.y));
          } else {
            node.x = node.fx;
            node.y = node.fy;
          }
        });

        this.render(ctx);  // 渲染画布

        requestAnimationFrame(simulationStep);  // 请求下一帧动画
      };

      simulationStep();
    },
    render(ctx) { // 渲染画布
      ctx.clearRect(0, 0, this.width, this.height);  // 清除画布
      ctx.save();//保存当前画布状态 确保在应用变换和绘制图形后，画布状态能够恢复到变换之前的状态，从而避免后续的绘图操作受到不必要的影响
      ctx.translate(this.transform.x, this.transform.y);  // 应用平移变换
      ctx.scale(this.transform.k, this.transform.k);  // 应用缩放变换

      // 绘制连接
      this.links.forEach(link => {
        const source = this.nodes.find(n => n.id === link.source);
        const target = this.nodes.find(n => n.id === link.target);

        ctx.beginPath();
        ctx.moveTo(source.x, source.y);
        ctx.lineTo(target.x, target.y);
        ctx.strokeStyle = `rgba(0,0,0,.3)`;
        ctx.lineWidth = 2;  // 固定边的粗细值
        ctx.stroke();
      });

      // 绘制节点
      this.nodes.forEach(node => {
        ctx.beginPath();
        ctx.arc(node.x, node.y, 5, 0, 2 * Math.PI, false);
        ctx.fillStyle = this.getColor(node.group);  // 根据节点组获取颜色
        ctx.fill();
        ctx.strokeStyle = '#000';
        ctx.stroke();
      });

      ctx.restore();
    },
    getColor(group) { // 根据节点组获取颜色
      const colors = ["#1f77b4", "#ff7f0e", "#2ca02c", "#d62728", "#9467bd", "#8c564b", "#e377c2", "#7f7f7f", "#bcbd22", "#17becf"];
      return colors[group % colors.length];  // 根据组返回颜色
    },
    onMouseDown(event) { // 鼠标按下事件
      const mousePos = this.getMousePos(event);
      this.draggingNode = this.getNodeAtPos(mousePos);  // 检查鼠标点击位置是否在节点上

      if (this.draggingNode) {
        // 固定节点位置
        this.draggingNode.fx = this.draggingNode.x;
        this.draggingNode.fy = this.draggingNode.y;
      } else {
        // 开始拖动画布
        this.isPanning = true;
        this.lastMousePos = mousePos;
      }
    },
    onMouseMove(event) { // 鼠标移动事件
      const mousePos = this.getMousePos(event);

      if (this.draggingNode) {
        // 更新节点固定位置
        this.draggingNode.fx = mousePos.x;
        this.draggingNode.fy = mousePos.y;
      } else if (this.isPanning) {
        // 更新画布平移
        const dx = mousePos.x - this.lastMousePos.x;
        const dy = mousePos.y - this.lastMousePos.y;

        this.transform.x += dx;
        this.transform.y += dy;

        this.lastMousePos = mousePos;
      }
    },
    onMouseUp() { // 鼠标松开事件
      if (this.draggingNode) {
        // 释放节点
        this.draggingNode.fx = null;
        this.draggingNode.fy = null;
        this.draggingNode = null;
      } else if (this.isPanning) {
        this.isPanning = false;  // 停止拖动画布
      }
    },
    onMouseWheel(event) { // 鼠标滚轮事件
      const mousePos = this.getMousePos(event);
      const zoom = event.deltaY > 0 ? 0.9 : 1.1;

      // 更新缩放比例和画布平移
      this.transform.k *= zoom;
      this.transform.x = mousePos.x - (mousePos.x - this.transform.x) * zoom;
      this.transform.y = mousePos.y - (mousePos.y - this.transform.y) * zoom;

      event.preventDefault();  // 阻止默认滚轮事件
    },
    getMousePos(event) { // 获取鼠标位置
      const rect = this.$refs.canvas.getBoundingClientRect();
      return {
        x: (event.clientX - rect.left - this.transform.x) / this.transform.k,
        y: (event.clientY - rect.top - this.transform.y) / this.transform.k
      };
    },
    getNodeAtPos(pos) { // 获取鼠标点击位置的节点
      return this.nodes.find(node => {
        const dx = node.x - pos.x;
        const dy = node.y - pos.y;
        return Math.sqrt(dx * dx + dy * dy) < 5;  // 检查鼠标是否在节点范围内
      });
    }
  }
};
</script>

<style scoped>
.canvas-container {
  display: flex;
  justify-content: center;
  align-items: center;
  width: 100%;
  height: 100vh;
}
canvas {
  border: 1px solid #ccc;
}
</style>
