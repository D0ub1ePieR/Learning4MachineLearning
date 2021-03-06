* [BFM](#BFM)
* [PCA](#PCA)
* [线性插值](#interpolation)
* [空间变换网络](#STN)
* [L1、L2正则化以及L1、L2 loss](#l1l2)
  - [范数](#norm)
* [Bottleneck](#google-net)
* [lambertian](#lambertian反射)
* [膨胀与腐蚀](#dilation&erosion)
+ [other](#other)

***

## TODO
  * 图卷积
  * GAN
  * Link-cut tree
  * 交叉熵

***

## <p id='BFM'>BFM(Basel Face Model)</p>
* 模型拥有**53490**个顶点、<strong>160470</strong>个三角形、由**199**个主成分的线性组合而成
* 模型给出了
  * 平均形状`average shape`
  * 形状主成分`principle shape components`
  * 形状偏差`shape variance`
  * 网状拓扑结构`mesh topology`
  * 平均纹理`average texture`
  * 纹理主成分`principle texture components`
  * 纹理偏差`texture variance`

  | name | size |
  | :-: | :-: |
  | **shapeMU**形状平均值 | 159645*1 |
  | **shapePC**形状主成分 | 159645*199 |
  | **shapeEV**形状方差 | 199*1 |
  | **textMU**纹理平均值 | 159645*1 |
  | **textPC**纹理主成分 | 159645*199 |
  | **textEV**纹理方差 | 199*1 |
  | **tri** | 3*105840 |
  | **kpt_ind**68个特征点 | 1*68 |
  | **expMU**表情平均值 | 159645*1 |
  | **expPC**表情主成分 | 159645*29 |
  | **expEV**表情方差 | 29*1 |
  | **tri_mouth**嘴巴部分三角面片顶点 | 3*114 |
***
## <p id='PCA'>PCA(principle component analysis)</p>
  > 输入$x_i$默认为列向量

  * 维度降低，但是主要信息保留，*去噪声，去除不重要的数据*
  * 任意三点*中心化*后都是线性相关的，n维空间中的n个点一定能在n-1维空间中分析
    > e.g.  x,y,z轴的坐标轴进行平移后得到x',y',z',新的点对于z'轴有较小的抖动，可以认为是噪声的引入使得数据的相关性降低，则可以考虑点在x',y'的投影构成了数据的主成分

    * 通过计算数据矩阵的协方差矩阵，得到协方差矩阵的特征值特诊向量，选择特征值最大`方差最大`的k个特征所对应的特征向量组成的矩阵。得到协方差矩阵的*特征值特征向量*有两种实现方法:**特征值分解协方差矩阵**和**SVD分解协方差矩阵**
    + [特征值分解](#character-value)
    + [SVD分解](#svd)
    + [协方差和散度矩阵](#covariance)
  * 特征值分解协方差矩阵实现PCA
    > 输入 $X=\{x_1,x_2,x_3,\cdots,x_n\}$  
      需要降到k维

    1. 去平均值，每一维特征的平均值
    2. 计算协方差矩阵 $\frac{1}{n}XX^T$
        > 这里不除或不除n或n-1，对求特征向量没有影响

    3. 用特征值分解方法求2中矩阵的特征值与特征向量
    4. 对特征值从小到大排序，选择其中最大的k个，然后将其对应的k个特征向量分别作为行向量组成特征向量矩阵P
    5. 将数据转换到k个特征向量构建的新空间中，即$Y=PX$
  * SVD分解协方差矩阵实现PCA
    1. 求平均值
    2. 计算协方差矩阵
    3. 通过SVD计算协方差矩阵的特征值与特征向量
    4. 对特征值从小到大排序，选择最大的k个，然后将其对应的k个特征向量分别作为行向量组成特征向量矩阵
    5. 将数据转换到k个特征向量构建的新空间中
  > 这一步使用SVD分解有两个好处：  
    1. 有一些SVD实现算法不需要求出协方差矩阵也能求出右奇异矩阵V。在*样本量很大*的时候很有效  
    2. PCA仅仅使用到了左奇异矩阵。*右奇异矩阵*的用途:  
    &nbsp;&nbsp;&nbsp;&nbsp;假设矩阵X($m*n$)，我们通过SVD找到了$X^TX$最大的k个特征向量组成的$k*n$的矩阵$V^T$，则$X^{'}_{m*k}=X_{m*n}V^{T}_{n*k}$ 可以得到m*k的矩阵$X^{'}$，这个矩阵列数从n减少到了k，对列方向进行了压缩。也就是说**SVD分解协方差矩阵实现PCA可以得到两个方向的PCA降维**。
  ```python
  def mypca(x, k):
    m, n = x.shape
    mean = np.array([np.mean(x[:, i]) for i in range(n)])
    norm_x = x - mean
    scatter_x = np.dot(x.T, x)
    val, vec = np.linalg.eig(scatter_x)
    pair = [(np.abs(val[i]), vec[:, i]) for i in range(n)]
    pair.sort(reverse=True)
    feature = np.array([ele[1] for ele in pair[:k]])
    return np.dot(norm_x, feature.T)


  def sk_pca(x):
    pca_x = PCA(n_components=1)
    pca_x.fit(x)
    return pca_x.transform(x)
  ```
  运行会发现两种结果正好取反，由于sklearn中的PCA是通过$svd\_flip$实现的，并对奇异值分解的结果进行了一个处理，因为$u_i*\sigma_i*v_i=(-u_i)*\sigma_i*v_i$，所以导致了PCA降维得到了*不一样*的结果。但结果都是*正确*的。

### PCA还原
  Y为原始数据降维后的结果 **$Y=PX$**，反推X则有：  
  $Y=P*X \ \rightarrow \ P^{-1}*Y=P^{-1}*P*X \ \rightarrow \ P^{-1}*Y=X$  
  随后需要对于得到的X加上均值
  ```python
  def sk_pca(x):
    pca_x = PCA(n_components=1)
    pca_x.fit(x)
    pc = pca_x.transform(x)
    re = pca_x.inverse_transform(pc)
    return pc, re


  def pca_reduction(pc, mean, feature):
    red = np.dot(pc, feature) + mean
    return red
  ```

***

### <p id='interpolation'></p>

* 单线性插值

  已知数据(x0,y0)与(x1,y1)，要在[x0,x1]区间内某一位置x在直线上的y值。则
  $$y=\frac{x_1-x}{x_1-x_0}y_0+\frac{x-x_0}{x_1-x_0}y_1$$
  双线性插值本质上就是在*两个方向*做线性插值

* 双线性插值

  核心思想是在**两个方向分别进行一次线性插值**。  
  假设需求点P=(x,y)，其周围四个点为$Q_{11}=(x_1,y_1),Q_{12}=(x_1,y_2),Q_{21}=(x_2,y_1),Q_{22}=(x_2,y_2)$
  首先在x方向上进行线性插值，则可以得到
  $$f(R_1)\approx\frac{x_2-x}{x_2-x_1}f(Q_{11})+\frac{x-x_1}{x_2-x_1}f(Q_{21}) \ \ R_1=(x,y_1)$$
  $$f(R_2)\approx\frac{x_2-x}{x_2-x_1}f(Q_{12})+\frac{x-x_1}{x_2-x_1}f(Q_{22}) \ \ R_2=(x,y_2)$$
  随后在y方向上进行线性插值，则可以得到
  $$f(P)\approx\frac{f(Q_{11})}{(x_2-x_1)(y_2-y_1)}(x_2-x)(y_2-y)+\frac{f(Q_{21})}{(x_2-x_1)(y_2-y_1)}(x-x_1)(y_2-y)$$

  综合可以得到双线性插值的结果
  $$f(x,y)\approx\frac{f(Q_{11})}{(x_2-x_1)(y_2-y_1)}(x_2-x)(y_2-y)+\frac{f(Q_{21})}{(x_2-x_1)(y_2-y_1)}(x-x_1)(y_2-y)$$
  $$+\frac{f(Q_{12})}{(x_2-x_1)(y_2-y_1)}(x_2-x)(y-y_1)+\frac{f(Q_{22})}{(x_2-x_1)(y_2-y_1)}(x-x_1)(y-y_1)$$
  > 由于图像坐标系以左上角为原点(0,0)，需要将源图像和目标图像做中心对齐，以达到更好的效果。  
    另外，在图像仿射变换中还有其他常见的插值方法如:最邻近插值，双三次插值，兰索思插值等

  在由源图像向目标图像变换时很容易出现非整数数值，所以由目标图像的整数坐标反向变换至源图像，得到f(i+u,j+v)`其中i、j为整数部分，u、v为小数部分`。则该点像素值可由(i,j)、(i+1,j)、(i,j+1)、(i+1,j+1)得到。
  $$f(i+u,j+v)=(1-u)(1-v)f(i,j)+(1-u)vf(i,j+1)+u(1-v)f(i+1,j)+uvf(i+1,j+1)$$

  > 如果直接进行四舍五入不仅会导致各个坐标点的值不准确，还会在梯度下降时造成困难。

***

### <p id='STN'></p>

  * 为了使模型对任务具有*尺度不变性、平移不变性、旋转不变性*。

  * STN可以作为一个单独的模块，输入不仅可以为一幅图像也可以是一个网络层输出的feature map。

  * 实现

      <center><img src='./imgs/STN-net.png'/></center>

      + 一个localisation net，输入$U\in R^{H*W*C}$，输出 $\Theta={a,b,c,d,e,f}$ 在仿射变换中的变量,即旋转以及平移参数。

      + 随后数一个网格生成器，由目标图坐标为自变量，$\Theta$为参数得到输入图中的坐标点

      <center><img src='./imgs/STN-net2.png'/></center>

      + 最后进入一个采样器，对扭曲后的图像进行填充。

***

<p id='other'></p>

### <p id='character-value'>特征值分解</p>
  * 特征值与特征向量<br/>
    矩阵A的特征值 $\lambda$，对应特征向量$v$，可以得到：**$Av=\lambda v$**
  * 分解<br/>
    对于矩阵A，有一组特征向量$v$，将它们*正交化单位化*，可以得到一组**正交单位向量$Q$**，于是矩阵A可以分解为：$A=Q\sum Q^{-1}。\sum$为一个对角矩阵，对角线上元素为特征值。
### <p id='svd'>SVD分解</p>
  * 对于任意矩阵的分解，$A=U\sum V^T$
  * A`m*n`得到U`m*m`，U中的正交向量称为*左奇异向量*；$\sum$`m*n`除了对角元素均为0，对角线上元素称为*奇异值*；$V^T$`n*n`，$V^T$中的正交向量称为*右奇异向量*。一般来说 $\sum$上的元素会从小到大排列
  * SVD步骤
    1. 求出$AA^T$的特征值及特征向量，用单位化的特征向量构成U
    2. 求出$A^TA$的特征值及特征向量，用单位化的特征向量构成V
    3. 将$AA^T或A^TA$的特征值求平方根，构成 $\sum$
### <p id='covariance'>协方差与散度矩阵<br/>
  * 协方差<br/>
    **样本均值**: $\overline{x}=\frac{1}{n}\displaystyle\sum^{N}_{i=1}x_i$<br/>
    **样本方差**：$S^2=\frac{1}{n-1}\displaystyle\sum^{n}_{i=1}(x_i-\overline{x})^2$<br/>
    **样本X和样本Y的协方差**：$Cov(X,Y)=E[(X-E(X))(Y-E(Y))]=\frac{1}{n-1}\displaystyle\sum^{n}_{i=1}(x_i-\overline{x})(y_i-\overline{y})$<br/>
    > 协方差为正时，XY为正相关；协方差为负时，XY为负相关；协方差为0时，XY相互独立。<br/>
    Cov(X,X)为X的方差，方差是协方差的一种特殊情况。

    n维数据的协方差可以构成一个**协方差矩阵**，对于一个*3维数据*(x,y,z)，可以计算它的协方差为：$Cov(X,Y,Z)=\left[\begin{matrix}Cov(x,x)&Cov(x,y)&Cov(x,z)\\Cov(y,x)&Cov(y,y)&Cov(y,z)\\Cov(z,x)&Cov(z,y)&Cov(z,z)\end{matrix} \right]$

  * 散度矩阵<br/>
    即协方差矩阵乘上一个n-1的系数
***

### <p id='hdf5'>HDF5(Hierarchical Data Format)</p>
> 层次数据结构第五代版本，用于存储科学数据的一种文件格式和库文件。  
  在内存占用、压缩、访问速度方面有非常优秀的特性

* H5将文件简化成两个主要的对象类型
  + 数据集
  + 组(*一种容器结构，可以包含数据集和其他组*)
* 内部资源使用类似**POSIX**的语法进行访问(*/path/to/source*)
* 使用B-tree来索引表格对象，非常适合时间序列的数据
* 类似python字典，有data和label部分

***
### <p id='lambertian'>lambertian反射</p>

  **理想散射**，在一个固定的照明分布下从所有的视场方向上观测都具有相同的亮度，不吸收任何*入射光*，也可以成为**散光反射**。不管照明分布如何，lambertian表面在所有的表面方向上接受并散发所有的入射照明。

  $$surfacecolor \ = \ Emissive \ + \ Ambient \ + \ Diffuse \ + \ Specular$$
  $$最终表面 \ = \ 放射光 \ + \ 环境光 \ + \ 漫反射 \ + \ 镜面反射$$

***

### <p id='google-net'>Bottleneck</p>

  > Going Deeper with Convolutions 中google提出了第一个inception结构

  受NIN`MLPconv`的启发,为了减少每一层的特征过滤器的数目，从而减少运算量。使用了1*1的卷积块减少特征数量。一般被称为**瓶颈层**。*inception模块保证了网络深度和宽度增加的同时减少了参数*，增加了网络对尺度的适应性。

  随后谷歌团队还提出v1-v4版本的inception模块
  * v2  -  Batch Normalization: Accelerating Deep Network Training by Reducing Internal Covariate Shift

    **batch-normalized inception** 被引入

  * v3  -  Rethinking the Inception Architecture for Computer Vision

    平衡了深度和宽度，当深度增加时，在前往下一层之前**增加特征的结合**。只是用3\*3的卷积，5\*5和7\*7的卷积核能分成多个3*3的卷积核

  * v4

    将**inception模块**和**resnet模块**相结合

***

### <p id='l1l2'>L1、L2正则化以及L1、L2 loss</p>

  * L1正则化可以使参数稀疏化，得到的参数是一个*稀疏矩阵*，可用于特征选择

  $$\Omega(\theta)=||w||_ 1=\sum_i|w_i|\tag{l1-norm}$$
  $$J=J_0+\lambda\sum_w|w|\tag{l1-loss}$$  

  即为原始损失加上正则化项, 并由 $\lambda$ 系数控制。根据公式可以得到L1正则的等值线是一个方形的，空间中J与等值线相交的点**大概率会交于顶点**即坐标轴上，因此 $w_i=0$的概率会很大，这也使得L1正则具有稀疏性。

  * L2正则化*非稀疏输出*，有解析解，抗扰动能力强。也被称作"权重衰减"，较L1而言更能防止过拟合。

  $$\Omega(\theta)=||w||_ 2^2=\sum_i|w_i|_ 2^2\tag{l2-norm}$$
  $$J=J_0+\lambda\sum_w|w|_ 2^2\tag{l2-loss}$$  

  同样可以画出L2正则的等值线，是一个更为光滑的圆形，J与等值线相交时 $w_i=0$的概率就会小了许多。

  * 正则对于偏导的影响

    L1相比L2，L1减少的是一个**常量**，L2减少的是**权重的固定比例**。  

    收敛速度取决于权重本身，权重大时L2会收敛更快。并且就计算效率上来讲L2能使计算更加高效。

  * L1、L2范数做为损失函数  

    + L1范数损失函数，**最小绝对值偏差(LAD),最小绝对值误差(LAE)**
    $$S=\sum^n_{i=1}|Y_i-f(x_i)|$$

    + L2范数损失函数，**最小平方误差(LSE)**
    $$S=\sum^n_{i=1}(Y_i-f(x_i))^2$$

    + | L2 | L1 |
      | :-: | :-: |
      | 不是非常鲁棒 | 鲁棒 |
      | 稳定解 | 不稳定解 |
      | 总是一个解 | 可能有多个解 |
  * <text id='norm'></text>范数
    - 矩阵的1、2范数

      $$||A||_ 1=max_j\sum^m_{i=1}a_{i,j}\tag{1-范数}$$
      即所有矩阵列向量绝对值之和的最大值
      $$||A||_ 2=\sqrt{\lambda_1}\tag{2-范数}$$
      即$A'A$矩阵的最大特征值的开方,$\lambda_1=A^TA$的最大特征值

    - Frobenius norm

      即矩阵元素绝对值的平方和再开方
      $$||A||_ F=(\sum^m_{i=1}\sum^n_{j=1}|a_{i j}|^2)^{\frac{1}{2}}=(tr(A^TA))^{\frac{1}{2}}$$

    - nuclear norm

      指矩阵奇异值的和
      $$||A||_ * =\sum_j\sigma_j(A)$$

      约束低秩，能凸近似rank

      应用: 1)矩阵填充, 2)鲁棒PCA,核范数+1范数, 3)背景建模,将前景后景分离, 4)变换不变低秩纹理,将旋转图片转正

***
### <p id='dilation&erosion'>膨胀与腐蚀</p>
***
### <p id='other'>other</p>
* vec操作

  $$A=\begin{bmatrix}a_{11}&\cdots&a_{1m}\\\vdots&\ddots&\vdots\\a_{n1}&\cdots&a_{nm}\end{bmatrix}$$

  $$vec(A)=[a_{11},\cdots,a_{1m},a_{21},\cdots,a_{2m},\cdots,a_{n1},\cdots,a_{nm}]^T$$
