https://mp.weixin.qq.com/s?__biz=MzU1NjEwMTY0Mw==&mid=2247586128&idx=2&sn=a95a7e687b19941b3945255eeee065c7&chksm=fbc9e834ccbe6122527496bb58cdef9a6faee06f0c3a7e602cdb8f34d8588c27e3fa71d62af1&scene=27
DIY相机（一）libcamera库
原创 bug404 古月居 2023-11-04 20:08 湖北
相机选型



DIY相机首先是要确定使用的相机型号。兼容树莓派，画质好一些的，目前主要有两款：一是Raspberry Pi Camera Module 3，二是Raspberry Pi HQ Camera。



下图是Raspberry Pi Camera Module 3的相关特性。支持自动对焦和HDR等特性，视场角为75度，1200万像素。



图片


Raspberry Pi Camera Module 3使用的是索尼imx708传感器。此款传感器曾搭载在如OPPO Find X2等旗舰手机上。



imx708传感器的参数如下表所示：



图片


其中说一下滚动快门。Rolling Shutter（滚动快门）是一种相机传感器工作方式，与其对立的是 Global Shutter（全局快门）。



在 Rolling Shutter 中，相机传感器会逐行地从顶部到底部逐个曝光，就像一个滚动的帘子一样。这意味着在一次曝光过程中，不同行的像素会有微小的时间差。



这种方式在拍摄运动速度较慢或相机固定时通常不会出现问题。然而，当拍摄快速移动的物体或相机本身在移动时，Rolling Shutter 可能会导致图像中出现奇怪的扭曲效果，这被称为“滚动快门效应”。



相比之下，Global Shutter 会在同一时间瞬间曝光整个图像传感器，而不是逐行进行。这意味着在拍摄运动物体或相机移动时，Global Shutter 不会导致扭曲效应，但相机可能会更昂贵，因为实现全局快门技术的传感器通常较复杂。



因此，在选择相机时，要考虑拍摄条件，如果你需要拍摄快速运动的物体，可能需要优先选择支持 Global Shutter 的相机。



第二款Raspberry Pi HQ Camera的优点是CMOS与镜头分离，可以根据需要搭载不同的镜头，比如长焦镜头、变焦镜头等，其使用imx477传感器。这款camera的详细信息等我们用时，再做详细说明。（主要是价格不菲，CMOS+镜头要小1k）。



除了考虑camera，还要考虑与树莓派的兼容性。下表是一个不同传感器与树莓派主板和驱动的对应关系。



图片


从表中可以看到，imx708传感器，只支持libcamera驱动，而不是之前的旧的raspicam驱动。



使用libcamera驱动，意味着我们在安装树莓派系统时，对应的Debian version要高于_Bullseye_，例如我用的就是Debian version: 12 (bookworm)系统。





拍照测试



使用bookworm系统的话，系统安装完成后，libcamera库是直接可以使用的。



树莓派也会自动连接camera。但是首先确认/boot/config.txt文件的内容如下面的文件内容所示，尤其是camera_auto_detect=1项。



对于树莓派camera module3来说，树莓派是可以自动检测的，不需要我们手动指定dtoverlay，后续我们用树莓派HQ camera的时候，其使用imx477传感器，这时候需要我们手动指定一下dtoverlay的型号。



# For more options and information see
# http://rptl.io/configtxt
# Some settings may impact device functionality. See link above for details

# Uncomment some or all of these to enable the optional hardware interfaces
#dtparam=i2c_arm=on
#dtparam=i2s=on
#dtparam=spi=on

# Enable audio (loads snd_bcm2835)
dtparam=audio=on

# Additional overlays and parameters are documented
# /boot/firmware/overlays/README

# Automatically load overlays for detected cameras
camera_auto_detect=1

# Automatically load overlays for detected DSI displays
display_auto_detect=1

# Automatically load initramfs files, if found
auto_initramfs=1

# Enable DRM VC4 V3D driver
dtoverlay=vc4-kms-v3d
max_framebuffers=2

# Don't have the firmware create an initial video= setting in cmdline.txt.
# Use the kernel's default instead.
disable_fw_kms_setup=1

# Run in 64-bit mode
arm_64bit=1

# Disable compensation for displays with overscan
disable_overscan=1

# Run as fast as firmware / board allows
arm_boost=1

[cm4]
# Enable host mode on the 2711 built-in XHCI USB controller.
# This line should be removed if the legacy DWC2 controller is required
# (e.g. for USB device mode) or if USB support is not required.
otg_mode=1

[all]


然后我们更新下系统



sudo apt update
sudo apt upgrade




预览camera流



直接使用libcamera-hello程序打开摄像头预览



sudo libcamera-hello -t 0


-t表示camera流持续多长时间，0表示一直持续。





拍摄照片



可以使用libcamera-jpeg命令拍摄图片。



libcamera-jpeg -o test.jpg


这个拍摄指令会显示一个5秒左右的预览串口，然后拍摄一张全像素的JPEG图像，保存为test.jpg。



另外，树莓派的libcamera驱动会针对不同的摄像头模块调用一个调谐文件，调谐文件中提供了各种参数，调用摄像头的时候，libcamera会调用调谐文件中的参数，结合算法对图像进行处理最终输出成预览画面。



由于libcamera驱动只能自动感光芯片信号，但是摄像头的最终显示效果还会受整个模块的影响，调谐文件的使用就是为了可以灵活处理不同模块的摄像头，调整提高图像质量。



在这里，我们调用系统中提供的针对imx708的调谐文件。



libcamera-jpeg -o test1.jpg --tuning-file /usr/share/libcamera/ipa/rpi/vc4/imx708.json


但是通过使用faststone软件对比，两张图片的成像并没有什么差距。调谐文件同样适用于其他的libcamera指令。



还可以指定出图的分辨率以及预览窗口的预览时间。-t 2000指2s的预览时间。



libcamera-jpeg -o test.jpg -t 2000 --width 640 --height 480




曝光补偿和曝光增益



树莓派的AEC/AGX算法允许程序指定曝光补偿，也就是通过设置EV数值来调整图像的亮度。比如：



libcamera-jpeg --ev -0.5 -o darker.jpg
libcamera-jpeg --ev 0 -o normal.jpg
libcamera-jpeg --ev 0.5 -o brighter.jpg


我们对如下的名词进行解释：



AEC（Automatic Exposure Control）和 AGX（Automatic Gain Control）是图像处理中常用的两种自动控制算法，它们通常用于相机或摄像机系统中，以调整图像的曝光和增益，以确保在不同光照条件下获得合适的图像质量。



AEC（自动曝光控制）：AEC 算法通过调整快门速度（曝光时间）来控制图像的亮度。在低光条件下，它会增加曝光时间以提高亮度，而在高光条件下，它会减少曝光时间以避免图像过曝。



AGX（自动增益控制）：AGX 算法通过调整图像的放大倍数来控制亮度。它可以在光线较暗的情况下增加图像信号的放大程度，以提高亮度。



EV代表曝光值（Exposure Value），它是一个相机设置，用于调整照片的曝光水平。EV值的变化会影响相机的快门速度、光圈和ISO等参数，从而改变照片的亮度和细节。



EV值是一个以对数形式表示的指标，通常以“EV +/- 数值”表示，例如+1 EV 或 -2 EV。EV值的变化一般以1/3或1/2 EV为单位进行调整。



正的EV值表示相机会增加曝光，使图像变亮。负的EV值表示相机会减少曝光，使图像变暗。



曝光补偿常用于情况复杂、光照条件变化或拍摄者希望在相机自动设置之外进行调整的情况。例如，在拍摄对比度很高的场景时，你可能会使用曝光补偿来避免过曝或欠曝的情况。



我们上面3张照片通过调整EV值来实现不同的曝光等级。下表是上面3张图片的exif信息。



图片


通过这3张照片的快门和ISO数据，可以看到我们通过调整EV值获得的不同曝光程度的照片，是通过调整快门和ISO数据得到的。



我们再来看一下libcamera通过控制增益来实现不同亮度的图片的。



libcamera-jpeg -o normal.jpg -t 2000 --shutter 25000 --gain 0
libcamera-jpeg -o brighter.jpg -t 2000 --shutter 25000 --gain -1
libcamera-jpeg -o darker.jpg -t 2000 --shutter 25000 --gain 1


我们通过--gain参数控制曝光增益，使用--shutter将快门时间固定在25ms。



图片
图片
图片


树莓派的camera module 3设想模组，光圈大小是不可调节的，快门速度我们也固定为了25ms，目前只有ISO是可调节的。



可见--gain参数也是通过控制ISO的值来得到不同曝光水平的照片的。





libcamera-still，更多的出图控制策略



可以像libcamera-jpeg一样，使用



libcamera-still -o test.jpg


得到一张图片。与libcamera-jpeg得到的图片基本一致，图片占用的存储空间也一致。



libcamera-still可以通过-e参数指定不同的编码器，实现不同的格式保存。可以支持png和bmp编码，也支持直接不带编码或者任何图像格式地将RGB或者YUV像素的二进制转储保存成文件。



如果是直接保存RGB或者YUV数据，程序在读取此类文件的时候必须了解文件的像素排列方式。



libcamera-still -e png -o test.png
libcamera-still -e bmp -o test.bmp
libcamera-still -e rgb -o test.data
libcamera-still -e yuv420 -o test_yuv.data


png和jpg是两种常用的格式，这里介绍一下他们的区别：



PNG（Portable Network Graphics）和JPEG（Joint Photographic Experts Group）是两种常见的图像文件格式，它们在某些方面有着明显的区别。



1.压缩算法：



○PNG使用无损压缩算法，保留了所有图像细节，不会损失图像质量。这使得PNG格式适用于需要保留高质量细节的场景，比如图形设计、线条图像等。



○JPEG使用有损压缩算法，通过消除图像中的一些细节和色彩信息来减小文件大小，从而降低了图像质量。这使得JPEG适用于照片等需要较小文件大小的情况。



2.颜色深度：



○PNG 支持索引色、灰度、RGB和RGBA等多种颜色模式，同时支持16位和8位深度的颜色，适用于各种色彩丰富的图像。



○JPEG 主要用于保存照片，通常以8位RGB模式存储。



3.透明度：



○PNG 支持完全透明和半透明，能够在图像中创建复杂的透明效果，适用于图形设计等需要透明背景的场景。



○JPEG 不支持透明度，它只能显示实色背景。



4.文件大小：



○JPEG 文件通常比同样分辨率和质量的PNG文件小得多，因为它是有损压缩的，可以在一定程度上减小文件大小。

○PNG 文件相对较大，因为它是无损压缩的，会保留所有的图像细节。



5.适用场景：



○PNG 适用于需要保留高质量细节、有透明度要求或者需要无损压缩的图像，比如图形设计、图标、线条图像等。

○JPEG 适用于照片和其他需要较小文件大小的场景。



综上所述，选择使用PNG还是JPEG取决于图像的具体用途和要求。



如果需要保留高质量细节、支持透明度或者需要无损压缩，那么PNG是一个更好的选择。如果主要是保存照片或者需要较小的文件大小，那么JPEG可能更适合。



可以看到，jpg文件的大小是1.2M，但是png文件的大小就到了10.7M。但是从对比图看，png相比jpg，没有明显的画质优势。



图片


输出RAW图



原始图像（Raw image)就是直接图像传感器输出的图像， 没有经过任何ISP或者CPU处理。在摄影后期中，在摄影的后期中，使用后期尤其重要，有以下的原因：



使用RAW格式图像相比JPEG格式具有一些优点，尤其是在后期处理方面：



1.更高的动态范围：RAW图像包含了相机传感器捕捉到的原始数据，因此它们通常具有比JPEG更高的动态范围。这意味着在后期处理中你可以更好地调整高光和阴影部分，以保留更多细节。



2.更多的色彩深度：RAW图像通常以更高的色彩深度（比如14位或16位）存储，相比之下，JPEG通常是8位。这使得你在后期处理时可以更精细地调整颜色和色调。



3.更大的灵活性：由于RAW图像未经压缩或经过少量压缩，因此你可以在后期处理时进行更大范围的调整，例如白平衡、曝光、对比度等。



4.无损压缩：RAW图像通常使用无损压缩，这意味着图像不会失去任何细节或质量，即使你进行了多次保存。



5.避免了摄影机的内部处理：JPEG图像通常会经过相机内部的处理，包括锐化、噪声降低等。使用RAW图像可以避免这些内部处理，使你在后期处理时能够更好地控制图像的外观。



6.更好的白平衡控制：RAW图像允许你在后期处理时更精确地调整白平衡，而JPEG通常会锁定一种白平衡。



然而，使用RAW格式也有一些缺点：



1.文件大小：RAW文件通常比JPEG文件大，因为它们包含了未经压缩的原始数据。



2.需要后期处理：与JPEG相比，RAW图像通常需要更多的后期处理工作，因为它们没有经过相机内部的自动处理。



3.需要专门的软件：通常需要专门的软件（如Adobe Camera Raw、Lightroom等）来处理RAW格式图像。



4.更大的存储需求：由于RAW文件通常较大，你可能需要更大的存储空间来保存这些文件。



综上所述，如果你在摄影中注重后期处理的灵活性和控制能力，使用RAW格式可能是个不错的选择。然而，如果你只是想快速拍摄和分享照片，JPEG格式可能更为便捷。



对于彩色相机传感器，一般来说原始图像的输出格式是Bayer. 注意原始图和我们之前说的位编码的RGB和YUV图像不同，RGB和YUV也是经过ISP处理后的图像的。



拍摄一张原始图像的指令：



libcamera-still -r -o test.jpg


原始图像一般是以DNG （Adobe digital Negative) 格式保存的，DNG格式可以兼容大部分标准程序， 比如dcraw、RawTherapee或者HoneyView. 原始图像会被保存为.dng后缀的相同名字的文件,比如如果运行上面的指令，为被另存为test.dng， 并同时生成一张jpeg文件。



DNG文件中包含了已图像获取有关的元数据， 比如白平衡数据，ISP颜色矩阵等， 下面是用exiftool工具显示的元数据编码信息：



---- ExifTool ----
ExifTool Version Number         : 12.69
---- File ----
File Name                       : test.dng
Directory                       : .
File Size                       : 24 MB
File Modification Date/Time     : 2023:10:29 17:18:03+08:00
File Access Date/Time           : 2023:10:29 17:18:03+08:00
File Creation Date/Time         : 2023:10:29 17:18:03+08:00
File Permissions                : -rw-rw-rw-
File Type                       : DNG
File Type Extension             : dng
MIME Type                       : image/x-adobe-dng
Exif Byte Order                 : Little-endian (Intel, II)
---- EXIF ----
Subfile Type                    : Reduced-resolution image
Image Width                     : 288
Image Height                    : 162
Bits Per Sample                 : 8 8 8
Compression                     : Uncompressed
Photometric Interpretation      : RGB
Make                            : Raspberry Pi
Camera Model Name               : imx708
Strip Offsets                   : 8
Orientation                     : Horizontal (normal)
Samples Per Pixel               : 3
Strip Byte Counts               : 139968
Planar Configuration            : Chunky
Software                        : libcamera-still
Subfile Type                    : Full-resolution image
Image Width                     : 4608
Image Height                    : 2592
Bits Per Sample                 : 16
Compression                     : Uncompressed
Photometric Interpretation      : Color Filter Array
Strip Offsets                   : 140624
Samples Per Pixel               : 1
Strip Byte Counts               : 23887872
Planar Configuration            : Chunky
CFA Repeat Pattern Dim          : 2 2
CFA Pattern 2                   : 2 1 1 0
Black Level Repeat Dim          : 2 2
Black Level                     : 64 64 64 64
White Level                     : 1023
Compression                     : Uncompressed
Strip Offsets                   : 0
Strip Byte Counts               : 0
Exposure Time                   : 1/16
ISO                             : 400
Date/Time Original              : 2023:10:29 17:18:03
Subject Distance                : 0.8847590685 m
DNG Version                     : 1.1.0.0
DNG Backward Version            : 1.0.0.0
Unique Camera Model             : Raspberry Pi imx708
Color Matrix 1                  : 0.9413505197 -0.32803303 -0.09434454143 -0.2564420402 1.072353721 0.1573984027 -0.02698263898 0.1290469915 0.4360769093
As Shot Neutral                 : 0.4639694691 1 0.5782194734
Calibration Illuminant 1        : D65
---- Composite ----
CFA Pattern                     : [Blue,Green][Green,Red]
Image Size                      : 4608x2592
Megapixels                      : 11.9
Shutter Speed                   : 1/16


这是关于一个名为 test.dng 的图像文件的 Exif 信息的提取结果。Exif 信息包含了许多有关照片的详细元数据。



以下是对一些关键信息的解释：



1.ExifTool版本信息：

ExifTool 版本号：12.69



2.文件信息：

文件名：test.dng

文件大小：24 MB

文件修改日期：2023年10月29日 17:18:03 (UTC+8)

文件访问日期：2023年10月29日 17:18:03 (UTC+8)

文件创建日期：2023年10月29日 17:18:03 (UTC+8)

文件权限：-rw-rw-rw-

文件类型：DNG (数字负片格式)

MIME类型：image/x-adobe-dng

Exif字节顺序：小端序 (Intel, II)



3.EXIF信息

Full-resolution image (全分辨率图像)：



图像宽度：4608 像素

图像高度：2592 像素

每样本位数：16 位

压缩：未压缩

光度解释：彩色滤波阵列 (Color Filter Array)

曝光时间：1/16 秒

ISO：400

拍摄时间：2023年10月29日 17:18:03

主体距离：0.8847590685 米

DNG版本：1.1.0.0

相机型号：Raspberry Pi imx708



Reduced-resolution image (降低分辨率图像)：

图像宽度：288 像素

图像高度：162 像素

每样本位数：8 位

压缩：未压缩

光度解释：RGB

相机制造商：Raspberry Pi

相机型号：imx708

…

（还有许多其他的Exif信息，比如色彩校正矩阵、曝光设置等等）



4.Composite信息：

CFA模式（Color Filter Array）：[蓝色，绿色][绿色，红色]

图像尺寸：4608x2592 像素

百万像素：11.9 MP (百万像素)



这份Exif信息提供了关于该图像的详尽信息，包括拍摄设备、拍摄条件、图像分辨率等等。





超长曝光



超长曝光一般用在我们拍摄流光快门的场景，通过长曝光，将一段时间内的景象变化都记录下来。



但是在我们使用的树莓派camera module3中，超长曝光是没太有用处的。



包括在没有可变光圈的手机专业模式中，设置超长曝光也不会有流光快门的效果。



因为想要得到流光快门的效果，不仅要快门设置的够长，光圈还要设置的特别小，甚至还要加上ND1000、ND64这样的减光镜，以及ISO调到最小，才能实现一张流光快门的效果。



这里简单说一下曝光三要素：



曝光三要素是指在摄影中控制照片曝光的三个关键因素，它们分别是：



1.光圈（Aperture）：

光圈是指相机镜头的光圈大小。它用一个数字表示，称为光圈值（f）。

光圈值越小，光圈越大，相机能够接收到更多光线，照片会更亮。

光圈值越大，光圈越小，相机接收到的光线更少，照片会更暗。

光圈还会影响景深，较大的光圈（小光圈值）会产生较大的背景虚化效果。



2.快门速度（Shutter Speed）：

快门速度是指相机传感器或胶片暴露于光线的时间长短。

快门速度越快，相机接收到的光线时间越短，照片会变得暗淡，但可以冻结运动。

快门速度越慢，相机接收到的光线时间越长，照片会更亮，但可能会导致运动模糊。



3.ISO感光度：

ISO值代表了相机传感器对光的敏感程度。较高的ISO值意味着相机在相同光线条件下可以使用更快的快门速度，但也可能会引入图像噪点。



较低的ISO值会减少噪点，但可能需要更慢的快门速度或更大的光圈来保持曝光正常。



这三个要素相互影响，称为”曝光三角”。调整其中一个参数会影响到其他两个。例如，如果你增加了光圈值（缩小光圈），你可以选择更快的快门速度来保持相同的曝光，或者降低ISO以减少噪点。



理解和掌握这些曝光三要素，可以让你更好地控制照片的曝光水平，从而拍摄出更令人满意的照片。



我们在使用微单或单反拍照时，有经验的摄影师也都会使用M档（手动挡）进行拍摄，这时候就需要根据场景去采用合适的光圈、快门和ISO大小。



比如在拍摄球类运动时，由于球的运动速度很快，为了保证排出的照片不模糊，需要用较高的快门速度，例如1/400s的快门速度，使用最大光圈，保证给足够的进光量，例如使用F1.8的大光圈。



这时候为了保证曝光水平的正常，可以让机器来决定ISO的数值，也可以自己指定ISO的数值。



如果使用微单或单反拍摄流光快门的效果，常用的参数配置是：快门30s，光圈F16或镜头支持的最小光圈，ISO机器支持的最低，添加ND1000或ND64减光镜。



使用手机拍摄流光快门效果时，其实不是像单反那样纯光学成像的效果，而是算法计算的结果。



因此，等同于手机，使用树莓派camera module 3时，由于光圈大小不可变，就算开启了超长曝光，也是实现不了流光快门的，因为光圈大小太大的话，曝光时间太长，就会导致进光量过大，照片会过曝，因此需要后期使用算法实现流光快门的效果。



我们在这里先介绍下libcamera长曝光的API。



如果要拍摄一张超长曝光的图片，我们需要禁用AEC/AGC和白平衡，否则这些算法会导致图片在收敛的时候多等待很多帧数据。



禁用这些算法需要另设置显式值，另外, 用户可以通过—immediate设置来跳过预览过程。



libcamera-still -o long_exposure.jpg --shutter 100000000 --gain 1 --awbgains 1,1 --immediate


备注：几款官方摄像头的最长曝光时间参考表格.



图片




总结



本篇主要说明了我们使用树莓派diy一个相机时，涉及到的相机选型和底层库libcamera的相关基础使用。



libcamera提供的功能远不止此，我们使用单反/微单的M档拍摄时，可以设置的全部参数，都可以通过libcamera库去设置。



但是libcamera只是在终端中使用的指令，我们制作一个相机，需要有自己的相机app和图库app等，这就需要我们在上层语言上进行调用。下一节我们会介绍对libcamera进行封装的picamera2库。

.. SPDX-License-Identifier: CC-BY-SA-4.0

.. section-begin-libcamera

===========
 libcamera
===========

**A complex camera support library for Linux, Android, and ChromeOS**

Cameras are complex devices that need heavy hardware image processing
operations. Control of the processing is based on advanced algorithms that must
run on a programmable processor. This has traditionally been implemented in a
dedicated MCU in the camera, but in embedded devices algorithms have been moved
to the main CPU to save cost. Blurring the boundary between camera devices and
Linux often left the user with no other option than a vendor-specific
closed-source solution.

To address this problem the Linux media community has very recently started
collaboration with the industry to develop a camera stack that will be
open-source-friendly while still protecting vendor core IP. libcamera was born
out of that collaboration and will offer modern camera support to Linux-based
systems, including traditional Linux distributions, ChromeOS and Android.

.. section-end-libcamera
.. section-begin-getting-started

Getting Started
---------------

To fetch the sources, build and install:

.. code::

  git clone https://git.libcamera.org/libcamera/libcamera.git
  cd libcamera
  meson setup build
  ninja -C build install

Dependencies
~~~~~~~~~~~~

The following Debian/Ubuntu packages are required for building libcamera.
Other distributions may have differing package names:

A C++ toolchain: [required]
        Either {g++, clang}

Meson Build system: [required]
        meson (>= 0.60) ninja-build pkg-config

for the libcamera core: [required]
        libyaml-dev python3-yaml python3-ply python3-jinja2

for IPA module signing: [recommended]
        Either libgnutls28-dev or libssl-dev, openssl

        Without IPA module signing, all IPA modules will be isolated in a
        separate process. This adds an unnecessary extra overhead at runtime.

for improved debugging: [optional]
        libdw-dev libunwind-dev

        libdw and libunwind provide backtraces to help debugging assertion
        failures. Their functions overlap, libdw provides the most detailed
        information, and libunwind is not needed if both libdw and the glibc
        backtrace() function are available.

for device hotplug enumeration: [optional]
        libudev-dev

for documentation: [optional]
        python3-sphinx doxygen graphviz texlive-latex-extra

for gstreamer: [optional]
        libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev

for Python bindings: [optional]
        libpython3-dev pybind11-dev

for cam: [optional]
        libevent-dev is required to support cam, however the following
        optional dependencies bring more functionality to the cam test
        tool:

        - libdrm-dev: Enables the KMS sink
        - libjpeg-dev: Enables MJPEG on the SDL sink
        - libsdl2-dev: Enables the SDL sink

for qcam: [optional]
        libtiff-dev qtbase5-dev qttools5-dev-tools

for tracing with lttng: [optional]
        liblttng-ust-dev python3-jinja2 lttng-tools

for android: [optional]
        libexif-dev libjpeg-dev

for Python bindings: [optional]
        pybind11-dev

for lc-compliance: [optional]
        libevent-dev libgtest-dev

for abi-compat.sh: [optional]
        abi-compliance-checker

Basic testing with cam utility
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``cam`` utility can be used for basic testing. You can list the cameras
detected on the system with ``cam -l``, and capture ten frames from the first
camera and save them to disk with ``cam -c 1 --capture=10 --file``. See
``cam -h`` for more information about the ``cam`` tool.

In case of problems, a detailed debug log can be obtained from libcamera by
setting the ``LIBCAMERA_LOG_LEVELS`` environment variable:

.. code::

    :~$ LIBCAMERA_LOG_LEVELS=*:DEBUG cam -l

Using GStreamer plugin
~~~~~~~~~~~~~~~~~~~~~~

To use the GStreamer plugin from the source tree, use the meson ``devenv``
command.  This will create a new shell instance with the ``GST_PLUGIN_PATH``
environment set accordingly.

.. code::

  meson devenv -C build

The debugging tool ``gst-launch-1.0`` can be used to construct a pipeline and
test it. The following pipeline will stream from the camera named "Camera 1"
onto the OpenGL accelerated display element on your system.

.. code::

  gst-launch-1.0 libcamerasrc camera-name="Camera 1" ! queue ! glimagesink

To show the first camera found you can omit the camera-name property, or you
can list the cameras and their capabilities using:

.. code::

  gst-device-monitor-1.0 Video

This will also show the supported stream sizes which can be manually selected
if desired with a pipeline such as:

.. code::

  gst-launch-1.0 libcamerasrc ! 'video/x-raw,width=1280,height=720' ! \
       queue ! glimagesink

The libcamerasrc element has two log categories, named libcamera-provider (for
the video device provider) and libcamerasrc (for the operation of the camera).
All corresponding debug messages can be enabled by setting the ``GST_DEBUG``
environment variable to ``libcamera*:7``.

Presently, to prevent element negotiation failures it is required to specify
the colorimetry and framerate as part of your pipeline construction. For
instance, to capture and encode as a JPEG stream and receive on another device
the following example could be used as a starting point:

.. code::

   gst-launch-1.0 libcamerasrc ! \
        video/x-raw,colorimetry=bt709,format=NV12,width=1280,height=720,framerate=30/1 ! \
        queue ! jpegenc ! multipartmux ! \
        tcpserversink host=0.0.0.0 port=5000

Which can be received on another device over the network with:

.. code::

   gst-launch-1.0 tcpclientsrc host=$DEVICE_IP port=5000 ! \
        multipartdemux ! jpegdec ! autovideosink

.. section-end-getting-started

Troubleshooting
~~~~~~~~~~~~~~~

Several users have reported issues with meson installation, crux of the issue
is a potential version mismatch between the version that root uses, and the
version that the normal user uses. On calling `ninja -C build`, it can't find
the build.ninja module. This is a snippet of the error message.

::

  ninja: Entering directory `build'
  ninja: error: loading 'build.ninja': No such file or directory

This can be solved in two ways:

1. Don't install meson again if it is already installed system-wide.

2. If a version of meson which is different from the system-wide version is
   already installed, uninstall that meson using pip3, and install again without
   the --user argument.
