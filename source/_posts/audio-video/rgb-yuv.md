---
title: RGB,YUV
date: 2017-08-12 10:55:32
categories:
reward: false
tags:
     - blog
     - 音视频
---

### 一、RGB
计算机彩色显示器显示色彩的原理与彩色电视机一样，都是采用R（Red）、G（Green）、B（Blue）相加混色的原理：通过发射出三种不同强度的电子束，使屏幕内侧覆盖的红、绿、蓝磷光材料发光而产生色彩。这种色彩的表示方法称为RGB色彩空间表示（它也是多媒体计算机技术中用得最多的一种色彩空间表示方法）。

根据三基色原理，任意一种色光F都可以用不同分量的R、G、B三色相加混合而成。
F = r [ R ] + g [ G ] + b [ B ]

<!--more-->

其中，r、g、b分别为三基色参与混合的系数。当三基色分量都为0（最弱）时混合为黑色光；而当三基色分量都为k（最强）时混合为白色光。调整r、g、b三个系数的值，可以混合出介于黑色光和白色光之间的各种各样的色光。

那么YUV又从何而来呢？在现代彩色电视系统中，通常采用三管彩色摄像机或彩色CCD摄像机进行摄像，然后把摄得的彩色图像信号经分色、分别放大校正后得到RGB，再经过矩阵变换电路得到亮度信号Y和两个色差信号R－Y（即U）、B－Y（即V），最后发送端将亮度和色差三个信号分别进行编码，用同一信道发送出去。这种色彩的表示方法就是所谓的YUV色彩空间表示。

采用YUV色彩空间的重要性是它的亮度信号Y和色度信号U、V是分离的。如果只有Y信号分量而没有U、V分量，那么这样表示的图像就是黑白灰度图像。彩色电视采用YUV空间正是为了用亮度信号Y解决彩色电视机与黑白电视机的兼容问题，使黑白电视机也能接收彩色电视信号。

#### 互相转换
YUV与RGB相互转换的公式如下（RGB取值范围均为0-255）：

Y = 0.299R + 0.587G + 0.114B
U = -0.147R - 0.289G + 0.436B
V = 0.615R - 0.515G - 0.100B

R = Y + 1.14V
G = Y - 0.39U - 0.58V
B = Y + 2.03U

#### 常见的RGB和YUV格式
在DirectShow中，常见的RGB格式有RGB1、RGB4、RGB8、RGB565、RGB555、RGB24、RGB32、ARGB32等；常见的YUV格式有YUY2、YUYV、YVYU、UYVY、AYUV、Y41P、Y411、Y211、IF09、IYUV、YV12、YVU9、 YUV411、YUV420等。

| GUID    |      格式描述|
| :-------- | --------:|
| MEDIASUBTYPE_RGB1   | 2色，每个像素用1位表示，需要调色板
| MEDIASUBTYPE_RGB4   | 16色，每个像素用4位表示，需要调色板
| MEDIASUBTYPE_RGB8   | 256色，每个像素用8位表示，需要调色板
| MEDIASUBTYPE_RGB565 | 每个像素用16位表示，RGB分量分别使用5位、6位、5位
| MEDIASUBTYPE_RGB555 | 每个像素用16位表示，RGB分量都使用5位（剩下的1位不用）
| MEDIASUBTYPE_RGB24  | 每个像素用24位表示，RGB分量各使用8位
| MEDIASUBTYPE_RGB32  | 每个像素用32位表示，RGB分量各使用8位（剩下的8位不用）
| MEDIASUBTYPE_ARGB32 | 每个像素用32位表示，RGB分量各使用8位（剩下的8位用于表示Alpha通道值）
| MEDIASUBTYPE_YUY2   | YUY2格式，以4:2:2方式打包
| MEDIASUBTYPE_YUYV   | YUYV格式（实际格式与YUY2相同）
| MEDIASUBTYPE_YVYU   | YVYU格式，以4:2:2方式打包
| MEDIASUBTYPE_UYVY   | UYVY格式，以4:2:2方式打包
| MEDIASUBTYPE_AYUV   | 带Alpha通道的4:4:4 YUV格式
| MEDIASUBTYPE_Y41P   | Y41P格式，以4:1:1方式打包
| MEDIASUBTYPE_Y411   | Y411格式（实际格式与Y41P相同）
| MEDIASUBTYPE_Y211   | Y211格式
| MEDIASUBTYPE_IF09   | IF09格式
| MEDIASUBTYPE_IYUV   | IYUV格式
| MEDIASUBTYPE_YV12   | YV12格式
| MEDIASUBTYPE_YVU9   | YVU9格式

#### RGB1、RGB4、RGB8
RGB1、RGB4、RGB8都是调色板类型的RGB格式，在描述这些媒体类型的格式细节时，通常会在BITMAPINFOHEADER数据结构后面跟着一个调色板（定义一系列颜色）。它们的图像数据并不是真正的颜色值，而是当前像素颜色值在调色板中的索引。以RGB1（2色位图）为例，比如它的调色板中定义的两种颜色值依次为0x000000（黑色）和0xFFFFFF（白色），那么图像数据001101010111…（每个像素用1位表示）表示对应各像素的颜色为：黑黑白白黑白黑白黑白白白…。

#### RGB565
 RGB565使用16位表示一个像素，这16位中的5位用于R，6位用于G，5位用于B。程序中通常使用一个字（WORD，一个字等于两个字节）来操作一个像素。当读出一个像素后，这个字的各个位意义如下：
| 高字节     |    低字节 | 
| :-------- | --------:|
| RRRRRGGG  |  GGGBBBBB|
可以组合使用屏蔽字和移位操作来得到RGB各分量的值：

``` cpp
#define RGB565_MASK_RED    0xF800
#define RGB565_MASK_GREEN  0x07E0
#define RGB565_MASK_BLUE   0x001F
R = (wPixel & RGB565_MASK_RED) >> 11;   // 取值范围0-31
G = (wPixel & RGB565_MASK_GREEN) >> 5;  // 取值范围0-63
B =  wPixel & RGB565_MASK_BLUE;         // 取值范围0-31
```


#### RGB555
RGB555是另一种16位的RGB格式，RGB分量都用5位表示（剩下的1位不用）。使用一个字读出一个像素后，这个字的各个位意义如下（X表示不用，可以忽略）：
| 高字节     |    低字节 | 
| :-------- | --------:|
| XRRRRRGG  |  GGGBBBBB|
可以组合使用屏蔽字和移位操作来得到RGB各分量的值：

``` cpp
#define RGB555_MASK_RED    0x7C00
#define RGB555_MASK_GREEN  0x03E0
#define RGB555_MASK_BLUE   0x001F
R = (wPixel & RGB555_MASK_RED) >> 10;   // 取值范围0-31
G = (wPixel & RGB555_MASK_GREEN) >> 5;  // 取值范围0-31
B =  wPixel & RGB555_MASK_BLUE;         // 取值范围0-31
```

#### RGB24
RGB24使用24位来表示一个像素，RGB分量都用8位表示，取值范围为0-255。注意在内存中RGB各分量的排列顺序为：BGR BGR BGR…。
通常可以使用RGBTRIPLE数据结构来操作一个像素，它的定义为：

``` cpp
typedef struct tagRGBTRIPLE { 
BYTE rgbtBlue;    // 蓝色分量
BYTE rgbtGreen;   // 绿色分量
BYTE rgbtRed;     // 红色分量
} RGBTRIPLE;
```


#### RGB32、ARGB32
使用32位来表示一个像素，RGB分量各用去8位，剩下的8位用作Alpha通道或者不用。（ARGB32就是带Alpha通道的 RGB32。）注意在内存中RGB各分量的排列顺序为：BGRA BGRABGRA…。
通常可以使用RGBQUAD数据结构来操作一个像素，它的定义为：

``` cpp
typedef struct tagRGBQUAD {
BYTE    rgbBlue;      // 蓝色分量
BYTE    rgbGreen;     // 绿色分量
BYTE    rgbRed;       // 红色分量
BYTE    rgbReserved;  // 保留字节（用作Alpha通道或忽略）
} RGBQUAD;
```

 
### 二、YUV
#### 关于YUV格式
摘自 [维基百科](https://zh.wikipedia.org/wiki/YUV)，YUV，是一种颜色编码方法。
YUV是编译true-color颜色空间（color space）的种类，Y'UV, YUV, YCbCr，YPbPr等专有名词都可以称为YUV，彼此有重叠。“Y”表示**明亮度**（Luminance、Luma），“U”和“V”则是**色度、浓度**（Chrominance、Chroma），Y'UV, YUV, YCbCr, YPbPr 常常有些混用的情况，其中 YUV 和 Y'UV 通常用来描述类比讯号，而相反的 YCbCr 与 YPbPr 则是用来描述数位的影像讯号，例如在一些压缩格式内 MPEG、JPEG 中，但在现今，YUV 通常已经在电脑系统上广泛使用。YUV Formats分成两个格式：

+ **紧缩格式（packed formats）**：将Y、U、V值储存成Macro Pixels阵列，和RGB的存放方式类似。
+ **平面格式（planar formats）**：将Y、U、V的三个份量分别存放在不同的矩阵中。

紧缩格式（packed format）中的YUV是混合在一起的，对于YUV4:4:4格式而言，用紧缩格式很合适的，因此就有了UYVY、YUYV等。平面格式（planar formats）是指每Y份量，U份量和V份量都是以独立的平面组织的，也就是说所有的U份量必须在Y份量后面，而V份量在所有的U份量后面，此一格式适用于采样（subsample）。平面格式（planar format）有I420（4:2:0）、YV12、IYUV等。像是一个三维平面一样。

#### 常见的YUV格式
为节省带宽起见，大多数YUV格式平均使用的每像素位数都少于24位元。主要的抽样（subsample）格式有YCbCr 4:2:0、YCbCr 4:2:2、YCbCr 4:1:1和YCbCr 4:4:4。YUV的表示法称为A:B:C表示法：
+ 4:4:4表示完全取样。表示色度值(UV)没有减少采样。即Y,U,V各占一个字节，加上Alpha通道一个字节，总共占4字节.这个格式其实就是24bpp的RGB格式了。
+ 4:2:2表示2:1的水平取样，垂直完全采样。表示UV分量采样减半,比如第一个像素采样Y,U,第二个像素采样Y,V,依次类推,这样每个点占用2个字节.二个像素组成一个宏像素。
+ 4:2:0表示2:1的水平取样，垂直2：1采样。  这种采样并不意味着只有Y，Cb而没有Cr分量，这里的0说的U，V分量隔行才采样一次。比如第一行采样 4:2:0 ,第二行采样 4:0:2 ,依次类推…在这种采样方式下，每一个像素占用16bits或10bits空间。
+ 4:1:1表示4:1的水平取样，垂直完全采样。可以参考4:2:2分量，是进一步压缩，每隔四个点才采一次U和V分量。一般是第0点采Y,U,第1点采Y,第3点采YV,第四点采Y,依次类推。

除了4:4:4采样，其余采样后信号重新还原显示后,会丢失部分UV数据，只能用相临的数据补齐，但人眼对UV不敏感，因此总体感觉损失不大。

用三个图来直观地表示采集的方式吧，以黑点表示采样该像素点的Y分量，以空心圆圈表示采用该像素点的UV分量。

![Alt text](http://ooid5jqhw.bkt.clouddn.com/1498793693858.png)

先记住下面这段话，以后提取每个像素的YUV分量会用到。
+ YUV 4:4:4采样，每一个Y对应一组UV分量。
+ YUV 4:2:2采样，每两个Y共用一组UV分量。 
+ YUV 4:2:0采样，每四个Y共用一组UV分量。 

最常用Y:UV记录的比重通常1:1或2:1，DVD-Video是以YUV 4:2:0的方式记录，也就是我们俗称的I420。至于其他常见的YUV格式有YUY2、YUYV、YVYU、UYVY、AYUV、Y41P、Y411、Y211、IF09、IYUV、YV12、YVU9、YUV411、YUV420等。

+ 4:2:2示例
   如果原始数据三个像素是 Y0 U0 V0 ,Y1 U1 V1,Y2 U2 V2,Y3 U3 V3
   经过4:2:2采样后，数据变成了 Y0 U0 ,Y1 V1 ,Y2 U2,Y3 V3
   如果还原后，因为某一些数据丢失就补成 Y0 U0 V1,Y1 U0 V1,Y2 U2 V3 ,Y3 U3 Y2
 
+ 4:1:1示例
   原来四个像素为: [Y0 U0 V0] [Y1 U1 V1] [Y2 U2 V2] [Y3 U3 V3]
   存放的码流为:    Y0 U0     ,Y1 , Y2  V2,   Y3
   还原出像素点为：[Y0 U0 V2] [Y1 U0 V2] [Y2 U0 V2] [Y3 U0 V2]
   
+ 4:2:0示例
   下面八个像素为：[Y0 U0 V0] [Y1 U1 V1] [Y2 U2 V2] [Y3 U3 V3] [Y5 U5 V5] [Y6 U6 V6] [Y7U7 V7] [Y8 U8 V8]
   存放的码流为：  Y0 U0 ,Y1, Y2 U2, Y3 ,Y5 V5, Y6, Y7 V7, Y8
   映射出的像素点为：[Y0 U0 V5] [Y1 U0 V5] [Y2 U2 V7] [Y3 U2 V7] [Y5 U0 V5] [Y6 U0 V5] [Y7U2 V7] [Y8 U2 V7]

#### YUY2、YUYV
YUY2（和YUYV）格式为每个像素保留Y分量，而UV分量在水平方向上每两个像素采样一次。一个宏像素为4个字节，实际表示2个像素。（4:2:2的意思为一个宏像素中有4个Y分量、2个U分量和2个V分量。）图像数据中YUV分量排列顺序如下：
Y0 U0 Y1 V0    Y2 U2 Y3 V2 …

#### YVYU
YVYU格式跟YUY2类似，只是图像数据中YUV分量的排列顺序有所不同：
Y0 V0 Y1 U0    Y2 V2 Y3 U2 …

#### UYVY
UYVY格式跟YUY2类似，只是图像数据中YUV分量的排列顺序有所不同：
U0 Y0 V0 Y1    U2 Y2 V2 Y3 …

#### AYUV
AYUV格式带有一个Alpha通道，并且为每个像素都提取YUV分量，图像数据格式如下：
A0 Y0 U0 V0    A1 Y1 U1 V1 …

#### Y41P、Y411
 Y41P（和Y411）格式为每个像素保留Y分量，而UV分量在水平方向上每4个像素采样一次。一个宏像素为12个字节，实际表示8个像素。图像数据中YUV分量排列顺序如下：
U0 Y0 V0 Y1    U4 Y2 V4 Y3    Y4 Y5 Y6 Y8 … 

#### Y211
Y211格式在水平方向上Y分量每2个像素采样一次，而UV分量每4个像素采样一次。一个宏像素为4个字节，实际表示4个像素。图像数据中YUV分量排列顺序如下：
Y0 U0 Y2 V0    Y4 U4 Y6 V4 …

#### YVU9
YVU9格式为每个像素都提取Y分量，而在UV分量的提取时，首先将图像分成若干个4 x 4的宏块，然后每个宏块提取一个U分量和一个V分量。图像数据存储时，首先是整幅图像的Y分量数组，然后就跟着U分量数组，以及V分量数组。IF09格式与YVU9类似。

#### IYUV
IYUV格式为每个像素都提取Y分量，而在UV分量的提取时，首先将图像分成若干个2 x 2的宏块，然后每个宏块提取一个U分量和一个V分量。YV12格式与IYUV类似。

#### YUV411、YUV420
YUV411、YUV420格式多见于DV数据中，前者用于NTSC制，后者用于PAL制。YUV411为每个像素都提取Y分量，而UV分量在水平方向上每4个像素采样一次。YUV420并非V分量采样为0，而是跟YUV411相比，在水平方向上提高一倍色差采样频率，在垂直方向上以U/V间隔的方式减小一半色差采样。

#### YUV转UYVY格式

``` cpp
void YUVtoUYVY(uint8_t *y_plane, uint8_t *u_plane, uint8_t *v_plane, 
			   int y_stride, int uv_stride,
			   OUT uint8_t *pDstBuf, int width, int height)
{
    for (int row = 0; row < height; row = row + 1) 
    {
        for (int col = 0; col < width; col=col + 2)
        {
            pDstBuf[0] = u_plane[row/2 * uv_stride + col/2];
            pDstBuf[1] = y_plane[row * y_stride + col];
            pDstBuf[2] = v_plane[row/2 * uv_stride + col/2];
            pDstBuf[3] = y_plane[row * y_stride + col + 1];
            pDstBuf += 4;
        }
    }
}
```


### 三、FOURCC
FourCC全称Four-Character Codes，代表四字符代码 (four character code), 它是一个32位的标示符，其实就是typedef unsigned int FOURCC;是一种独立标示视频数据流格式的四字符代码。

``` cpp
VC++转换方法：
DWORD fccYUY2 = MAKEFOURCC('Y','U','Y','2');
DWORD fccYUY2 = FCC('YUY2');
DWORD fccYUY2 = '2YUY';  // Declares the FOURCC 'YUY2'.

GUID：
FOURCCMap fccMap(FCC('YUY2'));
GUID g1 = (GUID)fccMap;

//Equivalent:
GUID g2 = (GUID)FOURCCMap(FCC('YUY2'));
```


#### FOURCC for YUV
[参考官网](http://www.fourcc.org/yuv.php)

**Packed YUV Formats**

![Alt text](http://ooid5jqhw.bkt.clouddn.com/1498808416643.png)

**Planar YUV Formats**

![Alt text](http://ooid5jqhw.bkt.clouddn.com/1498808497697.png)

### 四、YUV packed 
#### 1、YUYV、YUY2（属于YUV422）
相邻的2个Y共用其相邻的Cb、Cr分析，对于像素点Y'00、Y'01而言，其Cb、Cr的值均为Cb00、Cr00，其他的像素点的YUV取值依次类推。Y0 U0 Y1 V0  Y2 U2 Y3 V2

![Alt text](http://ooid5jqhw.bkt.clouddn.com/1498792179197.png)

#### 2、UYVY（属于YUV422）
UYVY格式也是YUV422采样的存储格式中的一种，只不过与YUYV不同的是UV的排列顺序不一样而已，还原其每个像素点的YUV值的方法与上面一样。 U0 Y0 V0 Y1 U2  Y2 V2 Y3

![Alt text](http://ooid5jqhw.bkt.clouddn.com/1498792314107.png)

### 五、YUV planar 

#### 1、YUV422P（属于YUV422）
这里，Y U V数据是分开存放的，每两个水平Y采样点，有一个Cb和一个Cr采样点，如下图：

![Alt text](http://ooid5jqhw.bkt.clouddn.com/1498792824367.png)

#### 2、YUV422 Semi-Planar
Semi 是’半‘的意思 我的理解这个半平面模式，这个格式的数据量跟YUV422 Planar的一样，但是U、V是交叉存放的，如下图：

![Alt text](http://ooid5jqhw.bkt.clouddn.com/1498786542233.png)

#### 3、YUV420P（I420、IYUV）
这个格式跟YUV422 Planar 类似，但对于Cb和Cr的采样在水平和垂直方向都减少为2:1，如下图：

![Alt text](http://ooid5jqhw.bkt.clouddn.com/1498793286690.png)

#### 4、YV12、YU12（属于YUV420）
YU12和YV12属于YUV420格式，也是一种Plane模式，将Y、U、V分量分别打包，依次存储。其每一个像素点的YUV数据提取遵循YUV420格式的提取方式，即4个Y分量共享一组UV。

#### 5、NV12、NV21（属于YUV420）
NV12和NV21属于YUV420格式，是一种two-plane模式，即Y和UV分为两个Plane，但是UV（CbCr）为交错存储，而不是分为三个plane。其提取方式与上一种类似，即Y'00、Y'01、Y'10、Y'11共用Cr00、Cb00
+ NV12

![Alt text](http://ooid5jqhw.bkt.clouddn.com/1498786766086.png)

