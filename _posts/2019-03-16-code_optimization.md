---
layout: post
title: 代码优化--使用矩阵运算代替循环
mathjax: true
date: 2019-03-16
categories: algorithm
tag: 矩阵，优化
---   

## 背景说明

最近接手了一个全景图片拼接的项目。由于整体视场较大，无法通过相机在一张图片中展现，因此使用同一个相机拍多张图片，这样可以覆盖整个视场。

大概是这个意思（图1）：

<div align="center">
<img src="/public/code_optimization/1.png" height="80%" width="80%">
</div>
<center> 图1 </center>
<br/>

<!-- more -->

图1中黄色边框、淡蓝色背景为所需的整个视场，相机依次拍摄了A、B、C、D、E、F六张小图来覆盖整个区域。  
项目中，为了覆盖整个视场区域，一共拍摄了96张小图，每张小图的高为$h=912$，宽为$w=1386$。

整个项目共两个需求：
1. 将所有的小图拼接为一份大图，来还原完整的视场图片
2. 对大图上的点与小图上的点一一匹配起来

接手项目之前，代码中存在大量的`for`循环，由于图片较大且数量较多，导致程序运行时间较长。因此我尝试将所有的`for`循环修改为矩阵运算。

在修改代码过程中，遇到了较多的坑，记录于此。

> PS：本文不会介绍拼接方法，只涉及代码优化部分。

## 算法优化

### 需求一：小图拼接以还原完整视场

现在我们需要把所有的小图拼接成一张大图，来还原一个完成的视场区域。

在进行拼接时，主要有两种思路：  
1. 寻找图像上的特征点，根据特征点进行匹配，实现图像拼接
2. 利用图像的GPS信息（经纬度、姿态角、高度等）进行拼接。这种方法对GPS信息的精度要求较高

本文主要采用第二种方法。

首先在96张小图中任选一张作为基准图片（base_img），其他95张小图（each_img）根据GPS信息向base_img进行旋转和平移，即在base_img坐标系中，依次放置95张each_img。

当把each_img放置在base_img坐标系中时，each_img中各个像素点的坐标也会发生改变。如图2所示，在小图B (each_img) 中，左下角点的坐标为$P(u1=1367, v1=911)$，起始下标为0。而在小图A (base_img) 坐标系下，$P$的坐标会变为$P'(u2=2100, v2=1500)$.

> $(u1, v1, 1)$是二维坐标的[齐次形式](https://baike.baidu.com/item/%E9%BD%90%E6%AC%A1%E5%9D%90%E6%A0%87/511284?fr=aladdin)，主要是为了进行矩阵运算。


<div align="center">
<img src="/public/code_optimization/2.png" height="100%" width="100%">
</div>
<center> 图2 </center>
<br/>

$P$点坐标转换为$P'$点坐标，由以下公式给出（参考[相机标定原理](https://www.cnblogs.com/Jessica-jie/p/6596450.html)，[根据经纬度计算距离](https://www.cnblogs.com/softfair/p/distance_of_two_latitude_and_longitude_points.html)和[姿态角](https://baike.baidu.com/item/%E5%81%8F%E8%88%AA%E8%A7%92/4783835?fr=aladdin)）：

$$\left[
\begin{matrix}
u_2 \\
v_2 \\
1 \\
\end{matrix}\right]= KRK^{-1}\cdot\left[
\begin{matrix}
u_1 \\
v_1 \\
1 \\
\end{matrix}\right]+\frac{Kt}h \tag{1}$$

其中$K$为相机内参，$R$为旋转向量，$t$为平移向量，$h$为拍摄时相机高度。  

假设以A为base_img，拼接后的大图应该为：

<div align="center">
<img src="/public/code_optimization/3.png" height="80%" width="80%">
</div>
<center> 图3 </center>
<br/>

**坐标转换**

在进行坐标转换时，最朴素的思想是两层循环：

```python
%%time
# 图片的高度与宽度
h, w = 912, 1368

# 图2中只对base_img右侧和下侧进行了扩展，这里对图片的top, bottom, left, right均进行了扩展
# 实际项目中，随着每次拼接的each_img数量的增多，会不断的在base_img的基础上对canvas进行扩展
canvas = cv2.copyMakeBorder(base_img, h, h, w, w, cv2.BORDER_CONSTANT, value = (0, 0, 0))
# for loop
# 对each_img中的每一个坐标进行转换
for u1 in range(w):  
    for v1 in range(h):
        # 写成齐次坐标的形式
        uv1 = np.array([[u1], [v1], [1]])
        # step 1: 根据公式对坐标进行转换
        uv2 = np.dot(np.dot(np.dot(K, R), np.linalg.inv(K)), uv1) + np.dot(K, t) / height
        # 取整
        uv2 = np.round(uv2)
        # step 2: 根据转换后的坐标，将each_img放置在canvas中
        canvas[int(uv2[1, 0]) + h, int(uv2[0, 0]) + w, :] = each_img[v1, u1, :]
```

    Wall time: 53.1 s
    

以上代码简洁明了，但是也很耗时。对一张图片进行坐标转换大约需要1min，一共95张图片，大约需要1.5h，这是一个非常耗时的工作，非常不利于debug。

仔细观察step 1和step 2，其实都可以转换为矩阵运算。

先看**step 1**的转换。

公式(1)给出了一个坐标点的情形下的转换方式，当有多个点存在时：

$$\left[
\begin{matrix}
u^{(1)}_2 & u^{(2)}_2 & ...  & u^{(i)}_2 & ... \\
v^{(1)}_2 & v^{(2)}_2 & ...  & v^{(i)}_2 & ... \\
\ 1 &  1 & ... & 1 & ... \\
\end{matrix}\right] = KRK^{-1} \cdot \left[
\begin{matrix}
u^{(1)}_1 & u^{(2)}_1 & ...  & u^{(i)}_1 & ... \\
v^{(1)}_1 & v^{(2)}_1 & ...  & v^{(i)}_1 & ... \\
\ 1 &  1 & ... & 1 & ... \\
\end{matrix}\right]+\frac{Kt}h \tag{2}$$

公式(2)中，$i$代表小图中不同的像素点，取值范围为：$1 \rightarrow  1247616(=912 × 1368)$

根据公式(2)，step 1可以转换成如下矩阵计算形式：

```python
%%time
# 生成坐标矩阵，uv1的shape为(3, 1247616)
# uv1的第一行为列坐标，对应u方向；第二行为行坐标，对应v方向；第三列均为1
grid = np.meshgrid(np.arange(w), np.arange(h))
uv1 = np.vstack((grid[0].T.reshape(1,-1), grid[1].T.reshape(1,-1), [1] * h * w))

# 计算a和b
a = np.dot(np.dot(K, R), np.linalg.inv(K))
b = np.dot(K, t) / height
# 转换后的坐标
uv_convert = np.around(np.dot(a, uv1) + b).astype(int)
# 由于canvas在上方和左侧进行了扩展，因此需要加上扩展的行数和列数才是最终的坐标值（uv坐标的原点在左上角，画个图感受一下就非常明了）
uv2 = np.around(np.dot(a, uv1) + b).astype(int) + np.array([[w], [h],[0]])
```

    Wall time: 309 ms
    
用时为0.309s。

再看**step 2**。

step 2的功能是将当前图片根据转换后的坐标放置于`canvas`中。

乍看起来非常困难，其实numpy已经提供了此功能，且看例子：

```python
# x为4行4列的0矩阵
# 类似于canvas的像素矩阵
# 请注意区别  像素矩阵  和  坐标矩阵
x = np.zeros(16).reshape(4, 4).astype(int)
print("x: ")
print(x)

# i表示行号
i = np.array([
    [0, 3],
    [1, 2]
])
# j表示列号
j = np.array([
    [0, 2],
    [2, 3]
])

# 这里的s类似于each_img的像素值
s = np.array([
    [-1, -2], 
    [-4, -5]
])

# 将x中坐标为[0, 0], [3, 2], [1, 2], [2, 3]（注意对应关系）的值替换为s
x[i, j] = s
print("new x:")
print(x)
```

    x: 
    [[0 0 0 0]
     [0 0 0 0]
     [0 0 0 0]
     [0 0 0 0]]
    new x:
    [[-1  0  0  0]
     [ 0  0 -4  0]
     [ 0  0  0 -5]
     [ 0  0 -2  0]]
    

了解上述功能后，step 2的转换就迎刃而解：

```python
canvas[uv2[1].reshape(w, h).T, uv2[0].reshape(w, h).T, :] = each_img
```

为了将step 2转成矩阵运算，我整整花了2天，  
一是，不知道numpy已经提供了此功能，搜索了很久；  
二是，uv2的坐标需要先reshape再转置（详因不细表）。   

当我写下这句话时，已经泪流满面。

我们看下转换为矩阵后，转换单张小图所需的总时间：

```python
%%time
h, w = 912, 1368
canvas = cv2.copyMakeBorder(base_img, h, h, w, w, cv2.BORDER_CONSTANT, value = (0, 0, 0))

# 生成坐标矩阵，uv1的shape为(3, 1247616)
# uv1的第一行为列坐标，对应u方向；第二行为行坐标，对应v方向；第三列均为1
grid = np.meshgrid(np.arange(w), np.arange(h))
uv1 = np.vstack((grid[0].T.reshape(1,-1), grid[1].T.reshape(1,-1), [1] * h * w))

# 计算a和b
a = np.dot(np.dot(K, R), np.linalg.inv(K))
b = np.dot(K, t) / height
# 转换后的坐标
uv_convert = np.around(np.dot(a, uv1) + b).astype(int)
# 由于canvas在上方和左侧进行了扩展，因此需要加上扩展的行数和列数才是最终的坐标值（uv坐标的原点在左上角，画个图感受一下就非常明了）
uv2 = np.around(np.dot(a, uv1) + b).astype(int) + np.array([[w], [h],[0]])
canvas[uv2[1].reshape(w, h).T, uv2[0].reshape(w, h).T, :] = each_img
```

    Wall time: 276 ms
    

一共不到0.3s，速度提升了几百倍。非常可观。

不得不感叹矩阵运算的优美。

### 需求二：像素点匹配

得到全景大图后，又接到一个新需求：随机在大图上进行点击，需要给出该点的来源图，以及来源点的坐标。

例如，大图上有一点$M$，坐标为$(u,v)$，我们需要找出来源图为B和E，以及来源点坐标为B图像坐标系下的$(u_b,v_b)$，以及E图像坐标系下的$E(u_e,v_e)$（如图4）。


<div align="center">
<img src="/public/code_optimization/4.png" height="100%" width="100%">
</div>
<center> 图4 </center>
<br/>

接到新需求后，我会在拼接大图时，保存所有小图转换后的坐标，即：

```python
indices = {  
    "image_name_1": [ uv_convert_1],   
    "image_name_2": [ uv_convert_2],   
    ...  
    "image_name_i": [ uv_convert_i],
    ...
}
```

之后，根据给定点M的坐标，在`indices`中进行循环查找即可。

```python
%%timeit
h, w = 912, 1368
# 给定点M（2347，1301）
(pt_u, pt_v) = (1301, 2400)  # 行和列

pt_res = []

# 循环查找
for name in indices.keys():
    # 将uv_convert转换成真实的坐标
    # 最终的大图中，canvas在base_img的基础上，在top方向上扩展了4个h，left方向扩展了2个w
    ind_true = indices[name] + np.array([[2 * w], [4 * h]])
    # 根据ind_true与pt进行匹配
    flag = ((ind_true[0] == pt_u) & (ind_true[1] == pt_v))
    pos = np.argwhere(flag == True)
    if pos.size > 0:
        # 一张小图中可能会有多个点匹配成功
        for i in pos:
            ind = int(i)  
            pt_res.append({name: (uv1[0][ind], uv1[1][ind])})

print(pt_res)
```

    [{'100_0170_0072.JPG': (4, 261)}, {'100_0170_0073.JPG': (9, 478)}, {'100_0170_0074.JPG': (9, 714)}, {'100_0170_0075.JPG': (6, 897)}, {'100_0170_0088.JPG': (976, 34)}, {'100_0170_0089.JPG': (978, 253)}, {'100_0170_0090.JPG': (982, 467)}, {'100_0170_0091.JPG': (983, 666)}, {'100_0170_0092.JPG': (984, 872)}]
Wall time: 2.83 s
    

从结果可以看出，查找一个点需要2.83s，非常影响客户体验。

因此我们同样尝试将循环转换为矩阵运算以提高运行效率。

**一张小图**

舍弃循环，我们首先理解一下上述代码的逻辑。

假设有一张$2 × 3$的小图，转换前后其坐标矩阵为：

```python
# 矩阵第一行为u方向，第二行为v方向，第三行省略（下同）
uv1 = np.array([
    [0, 1, 2, 0, 1, 2],
    [0, 0, 0 ,1, 1, 1],
])

# 转换之后，每个点在大图中的坐标为：
uv2 = np.array([
    [4, 3, 2, 4, 3, 2],
    [5, 4, 3 ,2, 1, 0],
])
```

我们要找的点为$M(4, 2)$：

```python
flag = uv2 == np.array([[4], [2]])
print(flag2)
```

    [[ True False False True False False]
     [False False False True False False]]
    

从结果可以看出，我们需要找的点在第3列（下标从0开始），全为`true`。  
那么如何通过矩阵运算来定位到`flag`矩阵的第3列呢？

`flag`中每一列有两个值，只有当这两个值都为`True`时，才是我们想找的点。

在python中，`True`即为1，因此`flag`相当于：

```python
flag = np.array([
    [1, 0, 0, 1, 0, 0],
    [0, 0, 0, 1, 0, 0]
])
```

自然的，我们对矩阵元素进行按列相加，那么结果为2的列，就是我们要找的列。

```python
# 矩阵列相加有两种方法：
# 方法1：
np.sum(flag, axis=0)
# 方法2：
m_m = np.array([[1], [1]])  # 构造媒介矩阵
np.dot(flag.T, m_m).T

```
    array([[1, 0, 0, 2, 0, 0]])

上述两种方法对应两种处理思路：
- 如果使用第一种方法，还是需要保留循环
- 第二种方法，可以舍却循环


**两张小图的情况**

假设现在有2张$2 × 3$的小图，转换前后的坐标矩阵为：

```python
# 转换前的坐标矩阵相同
uv1 = np.array([
    [0, 1, 2, 0, 1, 2],
    [0, 0, 0 ,1, 1, 1],
])

# 转换之后在大图中的坐标分别为：
img1_uv2 = np.array([
    [4, 3, 2, 4, 3, 2],
    [5, 4, 3 ,2, 1, 0],
])

img2_uv2 = np.array([
    [6, 5, 4, 6, 4, 4],
    [7, 6, 5 ,4, 2, 2],
])

# 我们将img1_uv2和img2_uv2进行合并
uv2 = np.vstack((img1_uv2, img2_uv2))
uv2
```

    array([[4, 3, 2, 4, 3, 2],
           [5, 4, 3, 2, 1, 0],
           [6, 5, 4, 6, 4, 4],
           [7, 6, 5, 4, 2, 2]])

由于对两个小图的坐标进行了堆叠，因此我们对要找的点的坐标$M(4, 2)$也进行堆叠，从而一一对应：

```python
flag = uv2 == np.tile(np.array([[4], [2]]), (2, 1))
flag
```
    array([[ True, False, False,  True, False, False],
           [False, False, False,  True, False, False],
           [False, False,  True, False,  True,  True],
           [False, False, False, False,  True,  True]])

使用方法2，首先构造媒介矩阵，然后对`flag`按列求和：

```python
m_m = np.array([
    [1, 0], 
    [1, 0], 
    [0, 1], 
    [0, 1]
])

pos = np.dot(flag.T, m_m).T
pos
```

    array([[1, 0, 0, 2, 0, 0],
           [0, 0, 1, 0, 2, 2]])

根据`flag`结果找出和为2的位置：

```python
ind = np.argwhere(pos == 2)
ind
```
    array([[0, 3],
           [1, 4],
           [1, 5]], dtype=int64)

结果矩阵中，第一列表示哪一张小图，第二列表示匹配成功的点所在的列，即：匹配成功的有三个点，其中第一个点在第0张图坐标矩阵的第3列，第2和第3个点在第1张小图的第4列和第5列。

**代码转换**

知道转换逻辑后，我们就可以将上述代码转换成矩阵形式。

首先我们需要做一些预处理：

```python
# 1. 首先，使用矩阵形式时，我们需要改变indices的形式：
# 其中image_names中存放所有图片的名称
# indices中按列堆叠每张小图转换后的矩阵，堆叠顺序与image_names中的小图的顺序相同
%%time
image_names = []
n = 0
for img_n, uv in indices.items():
    image_names.append(img_n)
    if n == 0:
        img_ind = uv
    else:
        img_ind = np.vstack((img_ind, uv))
    n += 1

# 真实坐标
ind_true = img_ind + np.tile(np.array([[2 * w], [4 * h]]), (95, 1)) 

# 2. 构造中间矩阵：
# 小图个数
img_num = 95 
m_m = np.zeros(img_num * 2 * img_num)

for i in range(img_num): 
    m_m[(2 * img_num + 1) * i] = 1 
    m_m[(2 * img_num + 1) * i + img_num] = 1 

m_m = m_m.reshape((img_num * 2, img_num)) 
```

之后再查找点的位置：

```python
%%time
# 查找点 
(pt_u, pt_v) = (1301, 2400)
%time flag = ind_true == np.tile(np.array([[1301], [2400]]), (95, 1)) 
%time pos = np.dot(flag.T, m_m).T
%time ind = np.argwhere(pos == 2)

for i in ind:
    print("img {}: ({},{})".format(image_names[i[0]], uv1[0][i[1]], uv1[1][i[1]]))
```

    Wall time: 256 ms
    Wall time: 2.69 s
    Wall time: 1.17 s
    img 0072.JPG: (4,261)
    img 0073.JPG: (9,478)
    img 0074.JPG: (9,714)
    img 0075.JPG: (6,897)
    img 0088.JPG: (976,34)
    img 0089.JPG: (978,253)
    img 0090.JPG: (982,467)
    img 0091.JPG: (983,666)
    img 0092.JPG: (984,872)
    Wall time: 4.12 s

结果表明，转换成矩阵后，时间反而变长了。主要时间消耗在`np.dot()`和`np.argwhere()`语句，分别耗时2.69s和1.17s。猜想其原因，应该是由于矩阵较大，其中`ind_true`的维数为$(190, 1247616)$，最终导致矩阵运算耗时较多。

> 点匹配时间效率问题的最终解决：  
> 上述代码运行结果，我们一共匹配了9张小图的9个点。而再次和客户确认需求后得知，客户只需要找出其中一张小图即可。在这样的需求下，我们发现使用循环效率更高。  
> 最终点匹配的时间在0.8s左右。

## 总结

由于我是临时接手，主要负责代码优化部分，从头至尾，总工花了约1周的时间。但收获颇丰，一些心得与大家共勉：
1. **确认真实需求**，这是项目开始第一步；
2. **正确对待**循环与矩阵运算。适当的矩阵运算可以提高算法效率；但当矩阵过大时，矩阵运算的时间开销，可能会远高于其带来的效率提高；
3. 将循环修改为矩阵运算时，首先应该搞清楚单次循环中的代码逻辑，再着手修改代码。

