---
layout: post
title:  给BMP格式的8位灰度图 加个文件头 C语言实现
date: 2020-09-21
categories:
- c/c++
tags: [c]
---

8位灰度图是256色位图的一种，它有调色板的颜色表，表中都是灰度色。文件头的大小是（14文件头+40信息+1024调色盘=1078字节）。

在我的项目中，下位机传来的数据只有具体的图像数据，缺少文件头，下面这个函数就可以给"raw"数据加上8位灰度图的头。

```c
//AddBITMAPHEAD
//by pipawancui
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <malloc.h>
#include <math.h>

typedef unsigned char BYTE;
typedef unsigned long DWORD;
typedef unsigned short WORD;
typedef long LONG;

/* bits表示一个位，要转换成字节必须  bits/8   但是这样可能产生问题，
例如该字节是32个字节则占4位，那么33个字节呢，
按上面的办法把33--64个字节的图像字节，都认为它占8位。
所以如果不到32的整数   加一个31再除32乘8便可以解决零头化为4的倍数了。 */
#define WIDTHBYTES(bits) (((bits) + 31) / 32 * 4)

typedef struct _BMPFILEHEAD
{
    //WORD bfType;      //文件类型，必须为 "BM"(0x4D42)
    DWORD bfSize;     //文件大小
    WORD bfReserved1; //保留字，不考虑
    WORD bfReserved2; //保留字，同上
    DWORD bfOffBits;  //实际位图数据的偏移字节数，即前三个部分长度之和
} BMPFILEHEAD;
//上面定义的结构体长度刚好为12，4的倍数 bfType拿出来

typedef struct _BMPFILETYPE
{
    WORD bfType; //文件类型，必须为 "BM"(0x4D42)
} BMPFILETYPE;

typedef struct _BMPINFOHEAD
{
    DWORD biSize;         //指定此结构体的长度，为40
    LONG biWidth;         //位图宽
    LONG biHeight;        //位图高
    WORD biPlanes;        //平面数，为1
    WORD biBitCount;      //采用颜色位数，可以是1，2，4，8，16，24，新的可以是32
    DWORD biCompression;  //压缩方式，可以是0，1，2，其中0表示不压缩
    DWORD biSizeImage;    //实际位图数据占用的字节数
    LONG biXPelsPerMeter; //X方向分辨率
    LONG biYPelsPerMeter; //Y方向分辨率
    DWORD biClrUsed;      //使用的颜色数，如果为0，则表示默认值(2^颜色位数)
    DWORD biClrImportant; //重要颜色数，如果为0，则表示所有颜色都是重要的
} BMPINFOHEAD;

typedef struct _RGBQUAD
{
    //public:
    BYTE rgbBlue;     //蓝色分量
    BYTE rgbGreen;    //绿色分量
    BYTE rgbRed;      //红色分量
    BYTE rgbReserved; //保留值,必须为0
} RGBQUAD;

/* 
* 函数名: add8GreyBmpHead
* 功能: 给未加工的原始数据加上一个8位灰度图的头
* 参数:
* 参数名         参数类型      IN/OUT/INOUT      备注
* pixData        void *        IN               原始数据
* width          long          IN               图片高
* height         long          IN               图片宽
* desFile        const char*   OUT              输出的文件路径
* 返回值：
* 类型：int
* 取值：0:成功,其它:无
* 其它：by pipawancui
 */
int add8GreyBmpHead(void *pixData, long width, long height,char const *desFile)
{
    BYTE bitCount = 8, color = 128;

    FILE *out;
    BMPFILETYPE bft;
    BMPFILEHEAD bfh;
    BMPINFOHEAD bih;
    RGBQUAD rgbquad[256];
    DWORD LineByte, ImgSize;
    LineByte = WIDTHBYTES(width * bitCount); //计算位图一行的实际宽度并确保它为32的倍数
    ImgSize = height * LineByte;             //图片像素大小

    for (int p = 0; p < 256; p++)
    {
        rgbquad[p].rgbBlue = (BYTE)p;
        rgbquad[p].rgbGreen = (BYTE)p;
        rgbquad[p].rgbRed = (BYTE)p;
        rgbquad[p].rgbReserved = (BYTE)0;
    }
    //bmp file's type
    bft.bfType = 0x4d42;

    //bmp fileheader's info
    bfh.bfSize = (DWORD)(54 + 4 * 256 + ImgSize);
    bfh.bfReserved1 = 0;
    bfh.bfReserved2 = 0;
    bfh.bfOffBits = (DWORD)(54 + 4 * 256);

    //bmp file's infoheader
    bih.biSize = 40; //本位图信息头的长度，为40字节
    bih.biWidth = width;
    bih.biHeight = -height;//翻转显示
    bih.biPlanes = 1;
    bih.biBitCount = bitCount; //位图颜色位深
    bih.biCompression = 0;     //是否压缩:0不压缩
    bih.biSizeImage = ImgSize; //像素数据大小;
    bih.biXPelsPerMeter = 0;
    bih.biYPelsPerMeter = 0;
    bih.biClrUsed = 0;      //用到的颜色数,为0则是 2^颜色位深
    bih.biClrImportant = 0; //重要的颜色数,为0则全部都重要
    fopen_s(&out, desFile, "w+b");

    if (out)
    {
        //write bmp file's type
        fwrite(&bft, sizeof(BMPFILETYPE), 1, out);
        //write bmp file's header info
        fwrite(&bfh, sizeof(BMPFILEHEAD), 1, out);
        //write bmp file's infoheader
        fwrite(&bih, sizeof(BMPINFOHEAD), 1, out);
        //write bmp file's RGBQUAD data
        fwrite(&rgbquad, sizeof(RGBQUAD), sizeof(rgbquad) / sizeof(RGBQUAD), out);
        //writing pix data to file
        fwrite(pixData, sizeof(BYTE), ImgSize, out)；
    }
    if (out)
        fclose(out);
    //--------------------------------------
    return 0;
}

```

