### 请看下一节

一句话简介：将某奇艺自家的QSV格式的视频转码为FLV格式的视频。

这已经是2015年的科技创新训练的项目了，做这个完全为了自己追番的需要。2016年左右，随着版权意识的提高，大部分资源已经不能下载了，也就没有继续研究了。2018年开始打理这个GitHub账号，想起把项目发布出来。由于时间比较久了，算法的一些细节已经记不清了，也没有留下比较规范的文档。最近突然发现有这么多人关注，还有2个issues。所以下面整理了当时的一些资料，希望对开源社区的各位大佬有帮助。



### 转码思路 ###

QSV格式本质上就是FLV，只不过进行了分段，添加了头部、文件信息来记录索引、专辑等信息。根据QSV头部信息，提取每一段的FLV视频数据，重建FLV头部即可完成转码。为了保证拖动进度条时播放器能快速定位到关键帧，还需要MetaData信息。每一段开头的MetaData经过加密，这里的做法是直接自己重建MetaData信息，而没有尝试解密，逆向大佬除外。



### QSV格式分析

_由于可能的QSV版本更新、分析的QSV文件不具有代表性和本人比较菜等原因，格式分析仅供参考。_

格式分析参考了**星沉地动**大佬的[QSV文件格式简单分析](https://blog.csdn.net/qq446252221/article/details/7362348)。刚才搜了一下，最近QSV格式分析的文章挺多的，可以参考一下其他人的博客。

#### 头部

| 偏移      | 数据类型 | 解释                                              |
| --------- | -------- | ------------------------------------------------- |
| 0x00-0x09 | BYTE[10] | 标识符，"QIYI VIDEO"                              |
| 0x0A-0x0D | DWORD    | 版本号，这里是0x2，其他版本号请谨慎参考此格式分析 |
| 0x0E-0x1D | DWORD[4] | 视频ID                                            |
| 0x42-0x45 | DWORD    | 是否有音频，0x0或0x1                              |
| 0x46-0x49 | DWORD    | 是否有视频，0x0或0x1                              |
| 0x4A-0x51 | QWORD    | “文件信息”的绝对偏移                              |
| 0x52-0x55 | DWORD    | “文件信息”的长度，是4的倍数                       |
| 0x56-0x59 | DWORD    | 视频分段数量                                      |

其他没有提到的部分作用未知或作为reserved供以后的扩展。

#### 文件信息

文件信息包括视频时长、专辑 ID、主要演员、分集数等，经过异或加密。每4字节为一组，异或0x79706762，解密后得到XML字符串。<del>我记得当时解密出来似乎有点问题，比如乱码什么的。不过这一节对视频转码没有影响。</del>

| 偏移（相对于本节首地址） | 数据类型 | 解释                                  |
| ------------------------ | -------- | ------------------------------------- |
| 0x00-0x03                | DWORD    | 文件信息版本号，这里是0x1             |
| 0x04-0x0B                | QWORD    | 文件信息长度                          |
| 0x36-本节末尾            | BYTE[]   | XML字符串，最后几字节是乱码，原因未知 |

#### 视频数据

本节紧接“文件信息”节之后，是分段的FLV编码的视频数据，每段6分钟。每段开头为一个ScriptTag，前1024字节经加密处理，末尾的PreTagSize总是比该Tag的实际长度少0xD。ScriptTag后紧接着一个VideoTag和一个AudioTag，Timestamp总是为0。后面Tag的Timestamp从上一段最后一个Tag的Timestamp算起。

FLV格式请参考[Adobe官网文档](https://www.adobe.com/content/dam/acom/en/devnet/flv/video_file_format_spec_v10_1.pdf)。



### 已知的问题

1. 重建的FLV似乎有问题，在某度网盘上播放出错，在主流的播放器上没有问题。
2. 本来想加进度条的，后来一想反正是自己用，就偷懒了😂。
3. [@xcdiv](https://github.com/xcdiv)提出：没太看懂意思，大概是转码某个大文件时只转码了前几段。
4. [@rkuk](https://github.com/rkuk)提出：跳过加密的ScriptTag的算法有点问题，会导致死循环。

其他issues也欢迎提出，建议提供一下有问题的视频文件的下载链接或者标题、ID等（虽然暂时没时间改bug。



### 程序员何必为难程序员

目前有更新的打算，但由于时间和精力的问题，对这一计划不做时间担保🤔。逆向工程已经弃坑很久了，C#也很久没写过了(听说现在C#7.0都出来了)，重新捡起这个项目又是一次新的挑战。程序员何必为难程序员，考研党何必为难考研党。

