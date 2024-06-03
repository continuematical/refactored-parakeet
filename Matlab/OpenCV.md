# 图像的基本变换
## 放大和缩小
```python
resize(src, dsize, fx, fy, interpolation)
```
`src`：目标图片；
`dsize` ：目标大小；
`fx, fy`：宽度和高度的缩放比，与`numpy` 不一样；
`interpolation`：插值算法；
主要有
```python
INTER_NEAREST: int  
INTER_LINEAR: int  
INTER_CUBIC: int  
INTER_AREA: int
```
1. 邻近插值，速度快，效果差；
2. 双线性插值，使用原图的四个点插值，默认；
3. 三次插值，使用原图的16个点插值；
4. 区域插值，效果好，计算时间最长。
示例
```python
def tran():  
    # resize  
    img = cv2.imread('img1.png')  
    cv_show('new_img', cv2.resize(img, (800, 400), interpolation=cv2.INTER_NEAREST))
```
## 翻转和旋转
### 翻转
```python
flip(src, flipcode)
```
`flipcode`：
1. `flipcode=0`上下翻转
2. `flipcode<0`上下左右翻转
3. `flipcode>0` 左右翻转
```python
def tran():  
    # resize  
    img = cv2.imread('img1.png')  
    cv_show('new_img', cv2.flip(img, flipCode=0))
```
### 旋转
```python
rotate(src, rotateCode)
```
`rotateCode`：
```python
ROTATE_90_CLOCKWISE: int  
ROTATE_180: int  
ROTATE_90_COUNTERCLOCKWISE: int
```
```python
def tran():  
    img = cv2.imread('img1.png')  
    cv_show('new_img', cv2.rotate(img, rotateCode=cv2.ROTATE_180))
```
## 仿射变换
图像旋转、缩放和平移的总称，具体的做法是通过一个矩阵和原图片坐标进行运算，得到新的坐标，完成变换，关键就是这个矩阵。
### 平移
```python
warpAffine(src, M, dsize, flag, mode, value)
```
`M`：变换矩阵，应该是`float32` 位；
`dsize`：输出图片大小；
`flag`：插值算法标志；
`mode`：边界外推法标志；
`value`：填充边界值；
如果沿着`(x,y)` 方向移动，移动的距离为$(t_x, t_y)$ ，构建的移动矩阵为：
$$
M=
\begin{bmatrix}
1 & 0 & t_x\\
0 & 1 & t_y
\end{bmatrix}
$$
```python
def tran():
	h, w, s = img.shape  
	# 水平右移200单位
	M = np.float32([[1, 0, 200], [0, 1, 0]])  
	new_img = cv2.warpAffine(img, M, (w, h), flags=cv2.INTER_NEAREST)  
	cv_show('new_img', new_img)
```
### 获取变换矩阵
进行旋转操作时，获取变换矩阵。
```python
getRotationMatrix2D(center, angle, scale)
```
`center`：旋转中心点
`angle`：旋转角度
`scale`：缩放比例
```python
def rotate_getM():  
    img = cv2.imread('img1.png')  
    h, w, s = img.shape  
    M = cv2.getRotationMatrix2D((10, 10), 15, 1)  
    new_img = cv2.warpAffine(img, M, (w, h), flags=cv2.INTER_NEAREST)  
    cv_show('new_img', new_img)
```
利用三个点旋转前后的位置确定变换矩阵。
```python
getAffineTransform(src, des)
```
`src`：三个点的初始位置；
`des`：三个点旋转后的位置。
```python
def rotate_getM():  
    img = cv2.imread('img1.png')  
    h, w, s = img.shape  
    M = np.float32([[200, 100], [300, 100], [200, 300]])  
    N = np.float32([[100, 150], [360, 200], [280, 120]])  
    res = cv2.getAffineTransform(M, N)  
    cv_show('new_img', cv2.warpAffine(img, res, (w, h), flags=cv2.INTER_NEAREST))
```
### 透视变换
# 滤波器
### 卷积
#### 基本概念
##### 图像卷积
又称为核操作，就是卷积核按步长对图像局部像素块进行加权求和的操作。
##### 卷积核大小
通常情况下，卷积核大小为奇数，用于设置卷积核中心作为锚点。
二维离散卷积公式如下：
$$
h[x,y]=\sum_{n_1=-\infty}^{+\infty}\sum_{n_2=-\infty}^{+\infty} g(x-n_1,y-n_2)\cdot f(n_1,n_2)
$$
其中$g(x,y)$ 是卷积核。
可以使用公式求出需要填充的零的圈数：
1. 输入体积大小 $W_1\times H_1\times D_1$ ；
2. 滤波器数量$K$ ，大小$F$ ，步长$S$ ，零填充大小$S$ ；
3. 输出体积大小$W_2\times H_2\times D_2$ ：
$$
H_2=(H_1-F+2P)/S+1\quad W_2=(W_1-F+2P)/S+1
$$
#### 算法实现
```python
def filter2D(src, ddepth, kernel, dst=None, anchor=None, delta=None, borderType=None)

# src：输入图像
# dst：相同大小和通道的结果图像
# ddepth：结果图像深度，看Depth combinations
# kernel：卷积核，一个单通道浮点矩阵
# anchor：卷积核锚点位置，默认值（-1,-1），表示在核中心
# delta：额外值
# borderType：边界填充像素方式，参考BorderTypes
```
```python
def convolve():  
    img = cv2.imread('img1.png')  
    # 卷积核必须是float类型  
    kernel = np.ones((5, 5), np.float32) / 25  
    
    # 突出边界，弱化细节  
kernel = np.array([[-1, -1, -1], [-1, 8, -1], [-1, -1, -1]])

    # 卷积  
    # 结果图像深度如果和原图一样的话，直接-1  
    des = cv2.filter2D(img, -1, kernel)  
    cv_show('convolve', des)
```
### 方框滤波
#### 原理
用一个内核图像进行卷积：
$$
K=\alpha
\begin{bmatrix}
1 & 1 & 1\\
1 & 1 & 1\\
1 & 1 & 1
\end{bmatrix}
$$
其中：
$$
\alpha=
\begin{cases}
\frac {1} {ksize.width\cdot ksize.height}\quad when\ nolmalize\ is\ true\\
1\quad otherwise
\end{cases}
$$
`ksize` 是滤波器的大小。
一般情况下我们使用`nolmalize=true` 的情况下，这是等同于均值滤波。
#### 实现
```python
void boxFilter( InputArray src, OutputArray dst, int ddepth,
                Size ksize, Point anchor = Point(-1,-1),
                bool normalize = true,
                int borderType = BORDER_DEFAULT );
```
```python
# 不需要手动创建卷积核，只需要传入卷积核的大小
def box_filter():  
    img = cv2.imread('img1.png')  
    des = cv2.boxFilter(img, -1, (5, 5), normalize=True)  
    cv_show('des', des)
```
### 均值滤波
```python
def blur_filter():  
    img = cv2.imread('img1.png')  
    des = cv2.blur(img, (5, 5))  
    cv_show('des', des)
```
### 高斯滤波
卷积核中的权重符合高斯分布。
二维高斯分布计算：
$$
G(x,y)=\frac 1 {2\pi\sigma^2}e^{-\frac {x^2+y^2} {2\sigma^2}}
$$
计算时需要制定$\sigma$ 的值。
```python
GaussianBlur(src, ksize, sigmaX, sigmaY)
```
`sigmaX`：x轴的标准差；
`sigmaY`：y轴的标准差，默认为0，此时`sigmaX=sigmaY` ；
`sigmaX`控制图像的平滑效果，其越大，平滑效果越明显。
### 中值滤波
取中位数作为卷积核的值。中值滤波对于胡椒噪音（椒盐噪音）效果明显。
```python
medianBlur(src, ksize)
```
`ksize` ：只是一个整数；
```python
def Median_filter():  
    img = cv2.imread('img1.png')  
    # 注意大小只能是一个整数
    des = cv2.medianBlur(img, 5)  
    cv_show('median', des)
```
### 双边滤波
双边滤波本质上是高斯滤波，不同的是，双边滤波既利用了位置信息又利用了像素信息来确定滤波窗口的权重，高斯滤波只利用了位置信息。
1. 对于高斯滤波，仅用空间距离的权重信息与图像卷积，确定中心点的灰度值。即认为，离中心点越近的值，权重越大；
2. 双边滤波加入了对灰度信息的权重，即在领域内，灰度值越距离中心权重更大，灰度值相差大的点权重值小。权重大小由高斯函数确定；两者权重信息相乘，就得到最终的卷积模板。双边滤波的速度比一般的滤波速度慢很多，而且计算量增长速度为核大小的平方；
双边滤波可以比较好的保留边界信息，同时对边界内的区域进行平滑处理。
#### 原理
将像素值权重表示为$G_r$ ，空间距离权重表示为$G_s$ 。
$$
G_s=exp(-\frac{||p-q||^2} {2\sigma_s^2})
$$
$$
G_r=exp(-\frac{||I_p-I_q||^2}{2\sigma_r^2})
$$
整个滤波器可以表示为$BF$ ，那么滤波结果为：
$$
BF=\frac1{W_q} \sum_{p\in S} G_s(p)G_r(p)\times I_p\\
=\frac1{W_q} \sum_{p\in S}exp(-\frac{||p-q||^2} {2\sigma_s^2})exp(-\frac{||I_p-I_q||^2}{2\sigma_r^2})\times I_p
$$
其中$W_q$ 为滤波窗口内每个像素的权重和，用于权重的归一化。
$$
W_q=\sum_{p\in S}G_s(p)G_r(p)=\sum_{p\in S}exp(-\frac{||p-q||^2} {2\sigma_s^2})exp(-\frac{||I_p-I_q||^2}{2\sigma_r^2})
$$
平坦区域，滤波器每个像素点$G_r$ 接近，由$G_s$ 主导滤波效果；
边缘区域，边缘同侧的$G_r$ 值相近，且远大于边缘另一侧的$G_r$ 值，此时另一侧的像素点的权重对滤波结果几乎不影响，边缘信息得到保护。
#### 实现
```python
void bilateralFilter( InputArray src, OutputArray dst, int d,
                                   double sigmaColor, double sigmaSpace,
                                   int borderType = BORDER_DEFAULT );
```
`diameter`：滤波窗口的直径；
`sigmaColor`：计算像素信息使用的`sigma`；
`sigmaSpace`：计算空间信息使用的`sigma`；
```python
def Gaussian_filter():  
    img = cv2.imread('img1.png')  
    des = cv2.GaussianBlur(img, (5, 5), 1)  
    cv_show('des', des)
```
对椒盐噪声几乎没有任何效果。
## 边缘检测
### 基本概念
作用：识别数字图像中亮度变化明显的点；
分类：
>基于搜索
>通过寻找图像一阶导数中的最大值来检测边界，然后利用计算结果估计边缘的局部方向，通常采用梯度的方向，并利用此方向找到局部梯度模的最大值，代表有`Sobel`算子和`Scharr`算子。

>基于零穿越
>通过寻找二阶导数零穿越来寻找边界，代表算法是`Laplace`算子。

### `Sobel`算子
#### 原理
图像是二维的，所以对宽度/高度两个方向求导。
1. 水平变换：将图像与奇数大小的模板进行卷积，结果为$G_x$ ，比如大小为3时，$G_x$ 为：
   $$
G_x=
\begin{bmatrix}
-1 & 0 & +1\\
-2 & 0 & +2\\
-1 & 0 & +1
\end{bmatrix}
\times I
$$
2. 垂直变换：将图像与奇数大小的模板进行卷积，结果为$G_y$ ，比如大小为3时，$G_y$ 为：
$$
G_y=
\begin{bmatrix}
-1 & -2 & -1\\
0 & 0 & 0\\
+1 & +2 & +1
\end{bmatrix}
\times I
$$
综合考虑两个方向的变化，得出：
$$
G=\sqrt{G_x^2+G_y^2}
$$
#### 实现
```python
void Sobel (InputArray src, OutputArray dst, int ddepth, int dx, int dy, int   ksize=3, 
            double scale=1, double delta=0, int borderType=BORDER_DEFAULT );  
```
`InputArray src` ：输入的原图像，Mat类型；
`OutputArray dst` ：输出的边缘检测结果图像，`Mat` 型，大小与原图像相同；
`int ddepth`：输出图像的深度，针对不同的输入图像，输出目标图像有不同的深度，具体组合如下：
1. 若src.depth() = CV_8U，取ddepth =-1/CV_16S/CV_32F/CV_64F；
2. 若src.depth() = CV_16U/CV_16S，取ddepth =-1/CV_32F/CV_64F；
3. 若src.depth() = CV_32F，取ddepth =-1/CV_32F/CV_64F；
4. 若src.depth() = CV_64F，取ddepth = -1/CV_64F；
注：ddepth =-1时，代表输出图像与输入图像相同的深度。
`int dx`：int类型dx，x 方向上的差分阶数，1或0；
`int dy`：int类型dy，y 方向上的差分阶数，1或0；

其中，dx=1，dy=0，表示计算X方向的导数，检测出的是垂直方向上的边缘；dx=0，dy=1，表示计算Y方向的导数，检测出的是水平方向上的边缘。
`int ksize`：为进行边缘检测时的模板大小为$ksize\times ksize$ ，取值为1、3、5和7，其中默认值为3。特殊情况：ksize=1时，采用的模板为3\*1或1\*3。
```python
def sobel_test():  
    img = cv2.imread('img1.png')  
    # 两个方向的梯度一定要分开计算，不然效果很差
    # 计算x轴方向的梯度  
    dx = cv2.Sobel(img, -1, dx=1, dy=0, ksize=3)  
    # 计算y轴方向的梯度  
    dy = cv2.Sobel(img, -1, dx=0, dy=1, ksize=3)  
    dt = cv2.add(dx, dy)  
    # dt = cv2.addWeighted(dx, dy)  
    cv_show('new', np.hstack((img, dt)))
```
### `Scharr` 算子
和`Sobel` 算子的思想一样，提取边界也更加灵敏，能提取到更细小的边界。但是该函数仅作用于内核大小为`3`的内核，只不过使用不同的`Kernel` 值。
$$
G_x=
\begin{bmatrix}
-3 & 0 & +3\\
-10 & 0 & +10\\
-3 & 0 & +3
\end{bmatrix}
\times I
$$
$$
G_y=
\begin{bmatrix}
-3 & -10 & -3\\
0 & 0 & 0\\
+3 & +10 & +3
\end{bmatrix}
\times I
$$
```python
Scharr(src: cv2.typing.MatLike, ddepth: int, dx: int, dy: int, dst: cv2.typing.MatLike | None = ..., scale: float = ..., delta: float = ..., borderType: int = ...)
```
- `src`：待提取边缘的图像；
- `ddepth`：输出图像的数据类型（深度），根据输入图像的数据类型不同拥有不同的取值范围，当赋值为-1时，输出图像的数据类型自动选择。
- `dx`：X方向的差分阶数
- `dy`：Y方向的差分阶数
- `scale`：对导数计算结果进行缩放的缩放因子，默认系数为1，不进行缩放。
- `delta`：偏值，在计算结果中加上偏值。
- `borderType`：像素外推法选择标志，默认参数为BORDER_DEFAULT，表示不包含边界值倒序填充。
### `Laplace` 算子

类似于二阶`Sobel` 算子。
### `Canny` 边缘检测
[Canny边缘检测算法 ](https://zhuanlan.zhihu.com/p/99959996)
最优边缘检测的三个主要评价标准是：
1. 低错误率：标识出尽可能多的实际边缘，尽可能减少噪声产生的误报；
2. 高定位性：标识出的边缘要与实际边缘尽可能接近；
3. 最小响应：边缘只能标识一次；
#### 步骤
##### 去噪
一般使用高斯滤波去噪。
##### 计算梯度
采用`Sobel` 算子计算梯度：
$$
G=\sqrt {G_x^2+G_y^2}
$$
$$
\theta=arctan(\frac {G_y}{G_x})
$$
梯度的方向总是与边缘垂直的，通常就近取值为水平（左、右）、垂直（上、下）、对角线（右上、左上、左下、右下）等 8 个不同的方向。
梯度由幅度和方向表示。
##### 非极大值抑制(NMS)
遍历图像中的像素点，去除所有非边缘的点。在具体实现时，逐一遍历像素点，判断当前像素点是否是周围像素点中具有相同梯度方向的最大值，并根据判断结果决定是否抑制该点。通过以上描述可知，该步骤是边缘细化的过程。针对每一个像素点：  
1. 如果该点是正/负梯度方向上的局部最大值，则保留该点；  
2. 如果不是，则抑制该点（归零）。
##### 滞后阈值
完成上述步骤后，图像内的强边缘已经在当前获取的边缘图像内。但是，一些虚边缘可能也在边缘图像内。这些虚边缘可能是真实图像产生的，也可能是由于噪声所产生的。对于后者，必须将其剔除。
设置两个阈值，其中一个为高阈值`maxVal`，另一个为低阈值`minVal`。根据当前边缘像素的梯度值与这两个阈值之间的关系，判断边缘的属性。
具体步骤为：
1. 如果当前边缘像素的梯度值 $I\ge maxVal$ ，则将当前边缘像素标记为强边缘；
2. 如果当前边缘像素的梯度值 $minVal\lt I\lt maxVal$ 
   与强边缘连接，则将该边缘处理为边缘；
   与强边缘无连接，则抑制该边缘；
3. 如果当前边缘像素的梯度值 $I\le minVal$ ，则抑制当前边缘像素。
#### 函数及使用
```python
edges = cv.Canny( image, threshold1, threshold2[, apertureSize[, L2gradient]])
```
1. edges 为计算得到的边缘图像；
2. image 为 8 位输入图像；
3. threshold1 表示处理过程中的第一个阈值；
4. threshold2 表示处理过程中的第二个阈值；
5. apertureSize 表示 Sobel 算子的孔径大小；
6. L2gradient 为计算图像梯度幅度的标识。其默认值为 `False` 。如果为 `True`，则使用更精确的 L2 范数进行计算（即两个方向的导数的平方和再开方），否则使用 L1 范数（直接将两个方向导数的绝对值相加）。
```python
def canny_test():  
    img = cv2.imread('img1.png')  
    des = cv2.Canny(img, threshold1=64, threshold2=128)  
    cv_show('des1', des)
```
# 图像形态学
## 基本概念
形态学图像处理（简称形态学）是指一系列处理图像**形状特征**的图像处理技术。
形态学的基本思想是利用一种特殊的**结构元**来测量或提取输入图像中相应的形状或特征，以便进一步进行图像分析和目标识别。
基本是对二进制图像进行处理，即黑白图像。
常用基本操作：
1. 膨胀和腐蚀；
2. 开运算和闭运算；
4. 顶帽；
5. 黑帽。
## 二值化
### 图像全局二值化
图像二值化就是把让图像的像素点只有`0`和`1`（只有黑白两各种颜色，黑是背景，白是前景），关键点是寻找一个阈值`T`，使图像中小于阈值`T`的像素点变为`0`，大于`T`的像素点变为`255`。
```python
retval = cv2.threshold(src, des, thresh, maxval, type)
```
1. `retval`：返回的阈值，double类型；
2. `des`：阈值分割结果图像(也可以写到函数参数里面)；
3. `src`：原图像， 最好是灰度图；
4. `thresh`：要设定的阈值；
5. `maxval`：当type类型是`THRESH_BINARY` 时，需要设定的最大值；
6. `type`：阈值分割的方式。
```python
def threshold():  
    img = cv2.imread('img2.png')  
    des, number = cv2.threshold(img, 0, 255, cv2.THRESH_BINARY)  
    cv_show('des', des)
```
### 自适应阈值二值化
前面的部分我们采取的是全局阈值，但是当一幅图像上有不同的亮度时，采取自适应阈值。
```python
dst = cv2.adaptiveThreshold(src, maxValue, adaptiveMethod, thresholdType, blockSize, C)
```
1. `src`：源图像，8位的灰度图；
2. `maxValue`：用于指定满足条件的像素设定的灰度值；
3. `adaptiveMethod`：使用的自适应阈值算法，有2种类型`ADAPTIVE_THRESH_MEAN_C`算法（局部邻域块均值）或`ADAPTIVE_THRESH_GAUSSIAN_C`（局部邻域块高斯加权和）。
`ADAPTIVE_THRESH_MEAN_C`的计算方法是计算出邻域的平均值再减去第六个参数C的值；
`ADAPTIVE_THRESH_GAUSSIAN_C`的计算方法是计算出邻域的高斯均匀值再减去第六个参数C的值。
处理边界时使用BORDER_REPLICATE | BORDER_ISOLATED模式。
4. `thresholdType`：阈值类型，只能是`THRESH_BINARY`或`THRESH_BINARY_INV`二者之一；
5. `blockSize`：表示邻域块大小，用来计算区域阈值，一般选择3、5、7等；
6. `C`：表示常数，它是一个从均匀或加权均值提取的常数，通常为正数，但也可以是负数或零；
注：只有一个返回值。
## 腐蚀和膨胀
### 腐蚀操作
腐蚀操作是将物体的边缘加以腐蚀。
用结构元素的中心点，从左到右从上到下，依次扫描灰度图的像素点，图片上该像素点的值取为结构元素所覆盖区域中像素点的**最小值**，扫描一遍后会得到一张新图，就是原图的腐蚀图。
大部分时候卷积操作使用的是全为`1`的卷积核。
```python
dst = cv2.erode(src, kernel, iterations)
```
`iterations`表示迭代次数。次数越多，腐蚀效果越明显。
```python
import cv2
import numpy as np
 
#读取图片
image = cv2.imread("E:/pythonProject/02.jpg",cv2.IMREAD_UNCHANGED)
 
#设置卷积核
kernel = np.ones((10, 10), np.uint8)
 
#图像腐蚀处理
erosion = cv2.erode(image, kernel)
 
#显示窗口
cv2.imshow("erosion", erosion)
cv2.imshow("image", image)
 
#窗口等待
cv2.waitKey(0)
cv2.destroyAllWindows()
```
#### 获取形态学卷积核
```python
kernel = cv2.getStructuringElement(shape, ksize, anchor)
```
其中，`shape`设定卷积核的形状，`ksize`设定卷积核的大小，`anchor`表示描点的位置，一般 `anchor = 1`，表示描点位于中心。
卷积核形状：
1. MORPH_RECT(函数返回矩形卷积核)  ；
2. MORPH_CROSS(函数返回十字形卷积核)；  
3. MORPH_ELLIPSE(函数返回椭圆形卷积核)。
```python
kernel = cv2.getStructuringElement(cv2.MORPH_ELLIPSE, (9, 9))  
print(kernel)
```
### 膨胀操作
```python
dst = cv2.dilate(src, kernel, iterations)
```
## 开运算和闭运算
### 开运算
先腐蚀，再膨胀。
```python
cv2.morphologyEx(src, cv2.MORPH_OPEN, kernel)
```
### 闭运算
先膨胀，再腐蚀。它经常被用来填充前景物体中的小洞，或者前景物体上的小黑点。
```python
cv2.morphologyEx(src, cv2.MORPH_CLOSE, kernel)
```
### 形态学梯度
梯度计算主要显示的是边缘信息。计算的方法：

> 膨胀的图像 - 腐蚀的图像。

```python
cv2.morphologyEx(img, cv2.MORPH_GRADIENT, kernel)
```
