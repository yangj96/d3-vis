# d3-vis
## 1. FD-Layout

FD-Layout(force-directed layout)布局算法的思想基于物理学的引力和斥力，图中的节点看作点电荷，任意节点之间存在静电斥力，用库仑定律计算；以边相连的节点间存在引力，弹簧的胡克定律计算，两种力合成转化成位移，导致节点位置的更新，反复迭代直到整个系统稳定下来，达到平衡。实际上以点与点之间的完全平衡作为理想效果，由于存在计算误差，结果可能会出现图的振荡，因此实现时通常采用将所有点的能量总和小于某一个阈值作为结束条件。本例使用D3.js实现了一个基于FD-Layout的图的可视化，数据使用某公司人物部门的关系网数据，在布局实现时主要有两个注意点：

1. 为保证图的布局不超过屏幕，实现中使用窗口大小作为svg的大小和力作用的范围。

2. 为保证点与点之间始终不相交，尤其是节点数非常多或人工拖动节点的情况，实现中包括基于四叉树实现的碰撞检测方法以防止节点覆盖，具体代码如下：

   ```javascript
   var padding = 1, 
       radius=10;
   
   function collide(alpha) {
     var quadtree = d3.geom.quadtree(graph.nodes);
     return function(d) {
       var rb = 2*radius + padding,
           nx1 = d.x - rb,
           nx2 = d.x + rb,
           ny1 = d.y - rb,
           ny2 = d.y + rb;
       
       quadtree.visit(function(quad, x1, y1, x2, y2) {
         if (quad.point && (quad.point !== d)) {
           var x = d.x - quad.point.x,
               y = d.y - quad.point.y,
               l = Math.sqrt(x * x + y * y);
             if (l < rb) {
             l = (l - rb) / l * alpha;
             d.x -= x *= l;
             d.y -= y *= l;
             quad.point.x += x;
             quad.point.y += y;
           }
         }
         return x1 > nx2 || x2 < nx1 || y1 > ny2 || y2 < ny1;
       });
     };
   }
   ```

除基础布局外，代码中实现三类交互操作：
1) 拖拽响应 
2) 双击单个节点高亮其相连节点，隐藏其他无关节点（双击恢复） 
3) 鼠标在节点上停留悬浮显示团队名和具体的员工姓名，其实鼠标悬浮标签的交互借助d3-tip来实现，核心代码如下：

```javascript
// 拖拽和高亮的相邻节点查找
var highlight = 0;
var linkedByIndex = {};
for (i = 0; i < graph.nodes.length; i++) {
    linkedByIndex[i + "," + i] = 1;
};

graph.links.forEach(function (d) {
    linkedByIndex[d.source.index + "," + d.target.index] = 1;
});

function neighboring(a, b) {
    return linkedByIndex[a.index + "," + b.index];
}
function connectedNodes() {
    if (highlight == 0) {
        d = d3.select(this).node().__data__;
        node.style("opacity", function (o) {
            return neighboring(d, o) | neighboring(o, d) ? 1 : 0.1;
        });

        link.style("opacity", function (o) {
            return d.index==o.source.index | d.index==o.target.index ? 1 : 0.1;
        });
        highlight = 1;
    } else {
        node.style("opacity", 1);
        link.style("opacity", 1);
        highlight = 0;
    }
}
// 悬浮标签
var tip = d3.tip()
  .attr('class', 'd3-tip')
  .offset([-10, 0])
  .html(function (d) {
    return  d.name + "</span>";
  })
svg.call(tip);
```
## 2. Edge Bundling

D3.js库中现有的Bundle Layout主要基于Danny Holten的[hierarchical edge bundling](http://citeseerx.ist.psu.edu/viewdoc/download?doi=10.1.1.220.8113&rep=rep1&type=pdf) 算法实现，但这一算法需要数据图能够创建合适的层次结构以便应用HEB进行bundling，
而本例使用d3.forceEdgeBundling的算法基于Holten于2009发表的[Force‐directed edge bundling for graph visualization](https://onlinelibrary.wiley.com/doi/abs/10.1111/j.1467-8659.2009.01450.x)一文，类似力导向布局的思想实现自组织方法的Edge Bundling，无需创建图的层次结构。

算法的基本思想是将节点之间的边缘建模为弹簧，如果满足初始化设置的几何兼容参数阈值则相互吸引使其自组织捆绑。为了改变节点之间的连线形状，算法将相连节点边的边缘细分为段，在同一条边的连续划分点之间存在弹簧的吸引力，使用胡可定律计算并且系数K可以设置控制边变形的僵硬柔软程度。在几何兼容参数范围内的不同邻边的划分段间存在电荷的引力，使得在几何兼容范围的边趋向融合。综合得到每个分段$p_i$上的合力$F_{p_i}$：

$$F_{p_i} =  k_P ·(||p_{i−1} − p_i||+ ||p_i − p_{i+1}||) + ∑\limits_{Q∈E} \frac{1}{p_i −q_i}$$

其中，E为除P以外在几何兼容范围内的所有相近边

作用在每个划分端上的合力使其在力的方向上移动一定的步长，模拟重复一定的迭代次数，在迭代循环结束之后，所得到的图形边缘再次以较小的分区划分，再次重复该过程直到结束循环。上述过程并未给出几何兼容参数的明确定义，考虑几何兼容参数需要考虑上述分割捆绑时的一些细节，例如垂直的边不应该bundling，长度相差较大的边不应该bundling，距离太远的边不应该bundling以及重叠区域很少的边，例如下图的情况的边不应该bundling。

![重叠长度较短情况](https://upload-images.jianshu.io/upload_images/2764802-23985f39f0dc5571.jpeg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

相应地，算法分别增加了$C_{\alpha}$,$C_{s}$,$C_p$和$C_v$对上述条件进行约束：

$$C_{\alpha} = |\cos(\frac{P*Q}{ |P||Q| })|$$,

$$C_{s} = \frac{2}{l_{avg}*min(|P|,|Q|) + max(|P|,|Q|)/l_{avg}}$$,

$$C_p = l_{avg} / (l_{avg} + ||P_m -Q_m||)$$,

$$V(P, Q) = max(1- \frac{2||P_m-I_m||}{||I_0-I_1||}, 0) \; C_v = min(V(P, Q) ,V( Q, P) )$$

综上，几何兼容参数$C_e = C_{\alpha}*C_{s}*C_p*C_v  \in [0,1]$, 高于该分数的值被视为可兼容的边，因此该参数值设置接近0，edge bundling效果越显著。另一个可以在调用时进行设置的参数是步长，由于在实现时我们固定力相互作用的迭代次数为60次，划分段模拟迭代的周期数为6，因此步长越小，图形会越类似node-link的原始图形，而步长太高会导致边缘过度扭曲。算法复杂度$O(N·M^2·K)$，运行时间较长。

## 3. 运行
项目路径下执行`python -m http.server`开启HTTP服务，浏览器输入对应地址即可运行。
