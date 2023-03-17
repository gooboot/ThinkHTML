### 一、前言

> flex布局也叫弹性布局；相对于浮动布局、定位布局来说，flex布局更加灵活易用，无论是在思想上还是在布局使用上都更胜一筹。尤其是实现元素的居中对齐非常方便

### 二、关键概念

- flex容器：设置了flex属性的容器元素就叫flex容器
- flex子项：在flex容器内的所有子元素叫flex子项，即**flex item**
- 主轴：用来确定flex子项的排列方向，是水平方向还是垂直方向。
- 侧轴：侧轴也叫**交叉轴**，侧轴垂直于主轴，特别注意：**只有先确定了主轴方向之后，才知道侧轴的方向，因为侧轴永远垂直于主轴**
- 剩余空间：flex子项在flex容器中排列后剩余的空间大小，这类似于Android中LinearLayout的*layout_weight*属性

### 三、flex布局的使用

- justify-content：用于主轴方向的对齐方式，属性值如下：

``` css
justify-content: flex-start;/*以主轴的start端对齐*/
justify-content: flex-end;/*以主轴的end端对齐*/
justify-content: center;/*主轴方向居中对齐*/
justify-content: space-around; /*子元素平均分配剩余空间*/
justify-content: space-between;/*两端的元素先贴边，再平均分配剩余空间*/
justify-content: stretch;/*朝主轴方向拉伸*/
```

- align-content：用于侧轴方向的对齐方式，属性值如下：

```css
align-content: flex-start; /* 以侧轴的start端对齐 */
align-content: flex-end;   /* 以侧轴的end端对齐 */
align-content: space-around;  /* 子元素平均分配剩余空间 */
align-content: space-between;/*两端的元素先贴边，再平均分配剩余空间*/
align-content: stretch;/*朝主轴方向拉伸*/
```

- align-items：用于flex子项在侧轴方向上的对齐方式，只适用于单行布局，属性值如下：

```css
align-items: flex-start; /* 对齐于侧轴的start端 */ 
align-items: flex-end; /* 对齐于侧轴的end端 */ 
align-items: center; /* 侧轴方向居中对齐 */ 
```

- align-self：用于子项单独在侧轴方向的对齐方式，属性值如下：

```css
align-self: flex-start; /* 以侧轴的start端对齐 */ 
align-self: flex-end; /* 对齐于侧轴的end端  */ 
align-self: center; /* 侧轴方向居中对齐 */ 
```









