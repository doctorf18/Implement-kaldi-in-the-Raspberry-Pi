# 在树莓派上搭建kaldi离线语音识别系统（交叉编译）


# 一、系统功能和环境概述
首先感谢博主[遇逆境处之泰然](https://blog.csdn.net/cj1989111)撰写的系列文章，本文内容参照该博主文章完成：

>参考：
>作者: [遇逆境处之泰然](https://blog.csdn.net/cj1989111)
>系列文章: [kaldi嵌入式平台的移植及实现](https://blog.csdn.net/cj1989111/article/details/84320162) 

## 1.1、实现功能
最近买了个树莓派，想做一个语音识别智能控制的功能，但是使用各大语音识别厂商提供的在线语音识别接口没有成就感，而且可能还要花钱=_=，干脆就使用Dan大神的开源语音识别框架kaldi来实现；在网上没有发现用树莓派去跑kaldi的相关博客和实例，只能自己来了。本来想着直接将kaldi源码在树莓派中直接编译，但毕竟是嵌入式的平台，很多依赖包和编译文件中会出现各式各样的问题，搞得人头大，所以选择使用交叉编译工具，速度快，还省心省力。

最终实现了将交叉编译后kaldi语音识别工具箱移植到树莓派开发板，并进行实时的离线语音识别，可以实现麦克风实时读入和大量连续词语音识别，识别率良好。

这是最后实现的功能：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191124144858728.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2NzA3Njk1,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191124144930625.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2NzA3Njk1,size_16,color_FFFFFF,t_70)

>这里使用的SHELL脚本是我随便写的小demo，只是为了说明kaldi离线识别的效果；

## 1.2、开发环境
- 树莓派4B开发板 4GB内存版本（某宝购入）+ 官网 raspberry（debian）Linux 32位系统
- 树莓派3.5寸电阻屏（买板子送的）+ 128G的TF卡一张 + 某宝购入的10块钱麦克风
- 树莓派官方交叉编译工具 arm-linux-gnueabihf
- Ubuntu_18.04 64位系统（Macbook Pro的Parallels Desktop虚拟机上跑的，这里其实也可以直接在MacOS里进行交叉编译，但也是会出现乱七八糟的问题=_=||，所以还是选择Ubuntu，省心靠谱，实现方便）

# 二、kaldi语音识别工具箱
关于kaldi语音识别工具箱，目前最流行和性能最好的开源语音识别框架，平民老百姓的首选，也适合学习语音识别的研究人员，再次感谢Dan Povey大神的贡献_(:з」∠)_。关于kaldi的安装和配置，不多介绍，网上有无数的教程和博客文章。

此处默认读者已可以在x86的Linux系统上成功编译并运行kaldi，简单了解kaldi文件结构和主要程序、脚本的功能。

kaldi源码下载：[kaldi源码下载](https://github.com/kaldi-asr/kaldi)

# 三、树莓派的相关配置
本文使用的树莓派是4B、4GB版本，当然也可以使用1GB、2GB的版本，但估计低内存的版本跑不了体积较大的连续词识别模型，一般一个可用的、效果尚可的大量连续词识别模型的大小在500M到1.5GB之间，所以建议使用内存大的版本。我曾在16GB内存笔记本上跑一个8GB的CVTE的模型，结果内存炸了。当然，如果是想实现孤立词识别，那256MB的内存就绰绰有余了。树莓派的内存卡为128的闪迪class10卡，其实用不到那么大的空间，一个32GB的tf卡就够了，16GB的够呛，因为整个kaldi源码编译下来就15个G了。

树莓派系统为官方raspberry32位桌面版系统[下载链接](https://www.raspberrypi.org/downloads/raspbian/)，建议不要使用其他系统，包括官网提供的ubuntu MATE，这些系统都会有乱七八糟的奇怪问题出现。。出现了问题后没有充分活跃的社区和足够的博客来参考并解决问题。

在移植kaldi之前，默认已经对树莓派完成基本配置，包括烧录系统、配置无线网、配置ssh、配置远程桌面或者3.5寸屏驱动、配置文件传输服务vsftpd、配置免驱麦克风。这些配置工作，网上有大量的教程和博客可以参考。

# 四、kaldi交叉编译过程
以下过程在Ubuntu系统中进行：

## 4.1、配置Ubuntu中的交叉编译环境

- 下载[树莓派官方工具箱（含交叉编译工具）](https://github.com/raspberrypi/tools)：

或者使用git工具克隆到本地：(克隆到本地的路径就是终端命令行所在路径)

	sudo apt-get install git
	git clone git://github.com/raspberrypi/tools.git
	
网上的所有教程都是这么写的。。但是！！！这个工具包！！！特别大！！！有时候github的渣网速根本下不动！！！这里提供三个解决办法：

> 1. 百度 "github下载提速教程" 自行找办法解决；
> 2. 如果懒得做第一条，那可以使用一个叫码云的网站（类似中国的代码托管平台，可以直接将github的存储库拉取到码云的库，再从码云的库下载，速度爽到飞起）；
> 3. 最后一条，我把这个工具箱中交叉编译工具单独上传到CSDN了(￣▽￣)~*，可以从我这里下载；

下载链接：

如果你的ubuntu虚拟机为32位：[树莓派的交叉编译工具链arm-linux-gnueabihf（32位linux）](https://download.csdn.net/download/qq_36707695/11994807)
如果你的ubuntu虚拟机为64位：[树莓派的交叉编译工具链arm-linux-gnueabihf（64位linux）](https://download.csdn.net/download/qq_36707695/11994794)

- 交叉编译环境配置：

如果是直接下载的整个工具箱，在根目录下找到arm-bcm2708文件夹并进入，找到gcc-linaro-arm-linux-gnueabihf-raspbian-x64或gcc-linaro-arm-linux-gnueabihf-raspbian这两个文件夹（前者为64位ubuntu使用的文件，后者为32位ubuntu使用的文件），将其移动到自己电脑中的某个路径下，如/home/tools/raspi/下。

将交叉编译命令加入系统环境变量，打开/.bashrc文件并添加路径：

	sudo gedit ~/.bashrc

*** 若是32位ubuntu系统，添加如下代码：

	export PATH=$PATH:$HOME/tools/raspi/gcc-linaro-arm-linux-gnueabihf-raspbian/bin

*** 若是64位ubuntu系统，添加如下代码：

	export PATH=$PATH:$HOME/tools/raspi/gcc-linaro-arm-linux-gnueabihf-raspbian-x64/bin
	
保存并退出文件，接着执行以下指令以便立即更新当前控制台所包含的环境变量。

	source .bashrc

测试：在命令行中输入：

	arm-linux-gnueabihf-gcc -v

如显示
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191124162711269.png)
则代表树莓派交叉编译环境配置完毕；

## 4.2、kaldi相关依赖工具的交叉编译
>这部分内容也可以参考博主[遇逆境处之泰然](https://blog.csdn.net/cj1989111)的系列文章[kaldi嵌入式平台的移植及实现](https://blog.csdn.net/cj1989111/article/details/84320162) 。不同之处在于该博主使用的为国内mips类型的嵌入式开发板和mips的交叉编译工具，部分步骤和语句有所不同。

该文章以下内容中用到的所有需要下载的各类依赖包，我都上传到了这里：[kaldi交叉编译各类依赖包](https://download.csdn.net/download/qq_36707695/11994860)，可以直接从这里进行下载，就不用后续在官网上挨个下载了。该文件中包含：

>openfst-1.6.7.tar.gz     			// openFST
>OpenBLAS-master.zip          // OpenBLAS
>clapack-3.1.1.1.tgz    			// Clapack
>alsa-lib-1.1.7.tar.bz2			 	// Alsa
>pa_stable_v190600_20161030.tgz     // portaudio

### 4.2.1 openFST的交叉编译过程
>openFST基本上是kaldi中最重要的依赖工具了，kaldi训练出的识别模型就是HCLG.fst文件（有限状态机），基于这个“搜索图”模型文件进行最佳路径的搜索就是语音识别过程。

下载openfst-1.6.7.tar.gz：[下载链接](http://www.openfst.org/twiki/pub/FST/FstDownload/openfst-1.6.7.tar.gz)

下载后拷贝openfst-1.6.7.tar.gz至kaldi源码路径下的tools/

> 这里需要说明，在./kaldi/tools/源码目录中本就存在openfst-1.6.7.tar.gz文件，但该文件中有损坏的概率很大，我在之前测试kaldi时，其自带的openfst工具包经常出问题，原因不清楚，所以建议先将tools文件夹下的所有名字中带有openfst字样的文件或文件夹删除，然后将新下载的openfst文件拷贝至tools下。

1、在tools目录下执行解压命令：

	tar -zxvf openfst-1.6.7.tar.gz 

2、进入文件：

	cd openfst-1.6.7/

3、执行命令：
	
	CXX=arm-linux-gnueabihf-g++ ./configure --prefix=`pwd` --enable-static --enable-shared --enable-ngram-fsts --host=arm-linux-gnueabihf LIBS="-ldl"

4、执行命令：

	make
	make install
	
5、创建软链接：

	ln -s openfst-1.6.7 openfst

### 4.2.2 OpenBlas的交叉编译过程
>OpenBlas是kaldi可以使用的一种基础线性代数子程序库，类似于Python里面的numpy，在C++中就是OpenBlas了。kaldi可以使用不同的矩阵运算库，此处以OpenBlas为主。

从github上下载：[OpenBlas下载地址](https://github.com/xianyi/OpenBLAS)

或直接git克隆：

	git clone https://github.com/xianyi/OpenBLAS.git

下载后的文件OpenBLAS-master.zip不必放在kaldi源码目录中，因为我们只需要交叉编译之后的静态库.a文件。

1、手动解压并进入根目录：

	cd ./OpenBLAS

2、执行编译指令：

	make TARGET=ARMV7 HOSTCC=gcc BINARY=32 CC=arm-linux-gnueabihf-gcc FC=arm-linux-gnueabihf-gfortran

3、输出中有显示"OpenBLAS build complete.(CBLAS)"则证明编译成功，然后执行指令：

	make PREFIX=./install/ install

然后可以发现生成的库在目录OpenBLAS/install/下。

### 4.2.3 clapack的交叉编译过程
>CLAPACK是LAPACK的C语言接口。LAPACK的全称是Linear Algebra PACKage，是非常著名的线性代数库。LAPACK是用Fortran写的，为了方便C/C++程序的使用，就有了LAPACK的C接口库CLAPACK。

编译clapack的过程比较麻烦，要分别编译生成四个静态库，首先下载[clapack](http://www.netlib.org/clapack/clapack-3.1.1.1.tgz)，下载文件clapack-3.1.1.1.tgz不必放入kaldi源文件目录。

1、解压clapack-3.1.1.1.tgz：

	tar xvf clapack-3.1.1.1.tgz

2、进入文件夹：

	cd CLAPACK-3.1.1.1/

3、安装gfortran：

	sudo apt-get install gfortran

4、根据例程创建一个make.inc文件：

	cp make.inc.example make.inc

5、打开make.inc并修改以下几条语句，然后保存退出：

    CC = arm-linux-gnueabihf-gcc
    LOADER    = arm-linux-gnueabihf-gcc
    ARCH     = arm-linux-gnueabihf-ar
    RANLIB   = arm-linux-gnueabihf-gcc-ranlib
    BLASLIB      = ../../libblas.a
    LAPACKLIB    = libclapack.a

6、编译生成libblas.a静态库：

在clapack根目录下，进入./BLAS/SRC文件夹：

	cd ./BLAS/SRC/

编译：

	make

命令行最后出现arm-linux-gnueabihf-gcc-ranlib ../../libblas.a 代表编译成功，此时在CLAPACK-3.1.1.1的根目录下，就可以看到生成的libblas.a。

7、编译生成liblapack.a、libclapack.a静态库:

在clapack根目录下，进入./SRC文件夹：

	cd ./SRC/

编译：

	make

命令行出现arm-linux-gnueabihf-ranlib ../libclapack.a 代表编译成功，此时在CLAPACK-3.1.1.1目录下，可以看到生成的libclapack.a静态库。

然后回到clapack根目录下，执行：

	cp libclapack.a liblapack.a

8、编译生成libf2c.a静态库：

在clapack根目录下，进入./F2CLIBS/libf2c/文件夹：

	cd ./F2CLIBS/libf2c/

修改文件夹下的makefile文件：

	注释该两行：
	#       ld -r -x -o $*.xxx $*.o
	#       mv $*.xxx $*.o

	注释或修改如下几行：
	libf2c.a: $(OFILES)
	#		ar r libf2c.a $?
	#		-ranlib libf2c.a
			arm-linux-gnueabihf-ar r libf2c.a $?
			arm-linux-gnueabihf-ranlib libf2c.a

	注释如下部分中的两行：
	arith.h: arithchk.c
        $(CC) $(CFLAGS) -DNO_FPINIT arithchk.c -lm ||\
         $(CC) -DNO_LONG_LONG $(CFLAGS) -DNO_FPINIT arithchk.c -lm
        #./a.out >arith.h
        #rm -f a.out arithchk.o

	另外：删除所有关于uninit的部分，比如uninit.o uninit.c等等。

修改完保存退出，执行：

	make

执行后会输出如下：

	arm-linux-gnueabihf-ar: creating libf2c.a
	arm-linux-gnueabihf-ranlib libf2c.a
	mv libf2c.a ..

然后进入到上一级的./F2CLIBS下，会发现生成了静态库libf2c.a，将其复制到clapack根目录下，四个静态库编译完成。

### 4.2.4 Alsa的交叉编译过程
>ALSA是Advanced Linux Sound Architecture的缩写，高级Linux声音架构的简称,它在Linux操作系统上提供了音频和MIDI（Musical Instrument Digital Interface，音乐设备数字化接口）的支持，ALSA也为kaldi的portaudio库提供了必要依赖。

先下载[Alsa](ftp://ftp.alsa-project.org/pub/lib/alsa-lib-1.1.7.tar.bz2)，下载文件alsa-lib-1.1.7.tar.bz2不必放入kaldi源文件目录。

1、解压：

	tar -jxvf alsa-lib-1.1.7.tar.bz2

2、进入根目录：

	cd alsa-lib-1.1.7/

3、配置交叉编译器：

**注意：此处的--prefix=/home/parallels/Desktop/alsa-lib-1.1.7/install/应改为自己电脑中alsa-lib-1.1.7/install/的路径(install文件夹不存在，通过此命令创建）**

	CC=arm-linux-gnueabihf-gcc  ./configure --enable-shared --prefix=/home/parallels/Desktop/alsa-lib-1.1.7/install/ --host=arm-linux-gnueabihf
	
4、编译:

	make
	sudo make install

编译完后生成的库文件在./install/文件夹下。

### 4.2.5 Portaudio的交叉编译过程
> PortAudio是一个免费的、跨平台的、开放源码的音频I/O库，kaldi在实时麦克风解码识别的部分会用到此工具库，如果需要使用麦克风输入并识别，可以编译此库。但是该库在kaldi中的安装过程bug非常多，导致我已经有了阴影=_=。虽然我不打算使用这个portaudio库（打算使用树莓派免驱麦克风 + 自己编写录音识别脚本），但还是把相关编译过程放在这以供参考。

下载[portaudio库](http://www.portaudio.com/archives/pa_stable_v190600_20161030.tgz)，下载文件pa_stable_v190600_20161030.tgz不必放入kaldi源文件目录。

1、解压：

	tar -zxvf pa_stable_v190600_20161030.tgz

2、进入源码目录

	cd portaudio/

3、配置交叉编译器：

	CC=arm-linux-gnueabihf-gcc  ./configure --enable-static --host=arm-linux-gnueabihf

4、如果第3步命令行输出结果中有"ALSA........no"，则做以下操作，打开Makefile文件，仔细对比并修改为以下代码：

	CFLAGS = -g -O2 -DPA_LITTLE_ENDIAN -I$(top_srcdir)/include -I$(top_srcdir)/src/common -I$(top_srcdir)/src/os/unix -pthread -DPACKAGE_NAME=\"\" -DPACKAGE_TARNAME=\"\" -DPACKAGE_VERSION=\"\" -DPACKAGE_STRING=\"\" -DPACKAGE_BUGREPORT=\"\" -DPACKAGE_URL=\"\" -DSTDC_HEADERS=1 -DHAVE_SYS_TYPES_H=1 -DHAVE_SYS_STAT_H=1 -DHAVE_STDLIB_H=1 -DHAVE_STRING_H=1 -DHAVE_MEMORY_H=1 -DHAVE_STRINGS_H=1 -DHAVE_INTTYPES_H=1 -DHAVE_STDINT_H=1 -DHAVE_UNISTD_H=1 -DHAVE_DLFCN_H=1 -DLT_OBJDIR=\".libs/\" -DHAVE_SYS_SOUNDCARD_H=1 -DHAVE_LINUX_SOUNDCARD_H=1 -DSIZEOF_SHORT=2 -DSIZEOF_INT=4 -DSIZEOF_LONG=8 -DHAVE_CLOCK_GETTIME=1 -DHAVE_NANOSLEEP=1 -DPA_USE_ALSA=1 -DPA_USE_OSS=1
 
	OTHER_OBJS = src/hostapi/alsa/pa_linux_alsa.o src/hostapi/oss/pa_unix_oss.o src/os/unix/pa_unix_hostapis.o src/os/unix/pa_unix_util.o src/common/pa_ringbuffer.o
	INCLUDES = portaudio.h pa_linux_alsa.h
 
	for include in $(INCLUDES); do \
    $(INSTALL_DATA) -m 644 $(top_srcdir)/include/$$include $(DESTDIR)$(includedir)/$$include; \
	done
	$(INSTALL_DATA) -m 644 $(top_srcdir)/src/common/pa_ringbuffer.h $(DESTDIR)$(includedir)/$$include;
	$(INSTALL_DATA) -m 644 $(top_srcdir)/src/common/pa_memorybarrier.h $(DESTDIR)$(includedir)/$$include;
	$(INSTALL) -d $(DESTDIR)$(libdir)/pkgconfig

	CXX = arm-linux-gnueabihf-g++
	AR = arm-linux-gnueabihf-ar

	CFLAGS = -g -O2 -DPA_LITTLE_ENDIAN -I$(top_srcdir)/include -I$(top_srcdir)/src/common -I$(top_srcdir)/src/os/unix -pthread -DPACKAGE_NAME=\"\" -DPACKAGE_TARNAME=\"\" -DPACKAGE_VERSION=\"\" -DPACKAGE_STRING=\"\" -DPACKAGE_BUGREPORT=\"\" -DPACKAGE_URL=\"\" -DSTDC_HEADERS=1 -DHAVE_SYS_TYPES_H=1 -DHAVE_SYS_STAT_H=1 -DHAVE_STDLIB_H=1 -DHAVE_STRING_H=1 -DHAVE_MEMORY_H=1 -DHAVE_STRINGS_H=1 -DHAVE_INTTYPES_H=1 -DHAVE_STDINT_H=1 -DHAVE_UNISTD_H=1 -DHAVE_DLFCN_H=1 -DLT_OBJDIR=\".libs/\" -DHAVE_SYS_SOUNDCARD_H=1 -DHAVE_LINUX_SOUNDCARD_H=1 -DSIZEOF_SHORT=2 -DSIZEOF_INT=4 -DSIZEOF_LONG=4 -DHAVE_CLOCK_GETTIME=1 -DHAVE_NANOSLEEP=1 -DPA_USE_ALSA=1 -DPA_USE_OSS=1 -I /home/parallels/Desktop/alsa-lib-1.1.7/install/include
	LDFLAGS = -L/home/parallels/Desktop/alsa-lib-1.1.7/install/lib/
	LIBS = -lm -lpthread /home/parallels/Desktop/alsa-lib-1.1.7/install/lib/libasound.so

上述代码最后三行中关于/home/parallels/Desktop/alsa-lib-1.1.7/install/的路径，请自行修改为自己的alsa库的路径。

5、修改完后保存退出，并执行：

	make
	make install PREFIX=`pwd`/install

编译完成后就可以在./install/目录下发现生成的库文件。

至此，kaldi相关依赖包的交叉编译已经完成，编译后的各类文件会在接下来进行移植。

## 4.3、kaldi的交叉编译
> kaldi的交叉编译首先要将上述过程生成的各类依赖库移动到kaldi源码中，所以要确保上述各类依赖包的交叉编译顺利完成。

1、进入kaldi源码根目录，确认openFST库的存放路径为：

> kaldi/tools/openfst/include/ 和 kaldi/tools/openfst/lib/

2、将之前在clapack根目录下的生成的以下三个静态库复制到OpenBLAS/install/lib文件夹下：

>./CLAPACK-3.1.1.1/libblas.a 
>./CLAPACK-3.1.1.1/libclapack.a 
>./CLAPACK-3.1.1.1/libf2c.a

3、进入./kaldi/src/文件夹下，打开configure文件，注释掉以下内容：

	# If more than one library specified, or a conflict has been recorded above
	# already, then add all deduced libraries as conflicting options (not all may
	# be conflicting sensu stricto, but let the user deal with it).
	#if [[ ${#mathlibs[@]} -gt 1 || $incompat ]]; then
	#  for libpfx in "${mathlibs[@]}"; do
	#    # Handle --mkl-libdir out of common pattern.
	#    [[ $libpfx == mkl && $MKLLIBDIR ]] && incompat+=(--mkl-libdir=)
	#    # All other switches follow the pattern --$libpfx-root.
	#    incompat+=(--$(lcase $libpfx)-root=)
	#  done
	#  failure "Incompatible configuration switches: ${incompat[@]}"
	#fi

4、配置交叉编译器：

注意该语句中的相关依赖路径要改为自己电脑的对应路径。

	CXX=arm-linux-gnueabihf-g++ AR=arm-linux-gnueabihf-ar AS=arm-linux-gnueabihf-as RANLIB=arm-linux-gnueabihf-ranlib ./configure --static --use-cuda=no --openblas-root=/home/parallels/Desktop/OpenBLAS/install --clapack-root=/home/parallels/Desktop/OpenBLAS/install

执行完该语句后会报错，其中错误为：

	mips-linux-gnu-g++: error: unrecognized command line option '-msse'
	mips-linux-gnu-g++: error: unrecognized command line option '-msse2'

5、打开./kaldi/src/kaldi.mk文件，添加如下内容：

注意将相关路经改为自己的对应路径。

	修改前两行内容并添加后三行内容：
	OPENBLASINC = /home/parallels/Desktop/OpenBLAS/install/include
	OPENBLASLIBS = -L/home/parallels/Desktop/OpenBLAS/install/lib -l:libopenblas.a -lgfortran
	OPENBLASLIBS += /home/parallels/Desktop/OpenBLAS/install/lib/libclapack.a 
	OPENBLASLIBS += /home/parallels/Desktop/OpenBLAS/install/lib/libblas.a
	OPENBLASLIBS += /home/parallels/Desktop/OpenBLAS/install/lib/libf2c.a

删除如下参数：

	-msse 
	-msse2 

6、在OpenBLAS/install/include文件夹中，添加OpenBLAS/lapack-netlib/LAPACKE/include/路径下的四个文件：

注意此处文件为我电脑的路径。

	/home/parallels/Desktop/OpenBLAS/lapack-netlib/LAPACKE/include/lapacke.h
	/home/parallels/Desktop/OpenBLAS/lapack-netlib/LAPACKE/include/lapacke_config.h
	/home/parallels/Desktop/OpenBLAS/lapack-netlib/LAPACKE/include/lapacke_mangling.h
	/home/parallels/Desktop/OpenBLAS/lapack-netlib/LAPACKE/include/lapacke_utils.h

7、编译kaldi

	make depend -j4
	make 

此处编译时间较长，大概20-40分钟；

8、编译onlinebin解码器

>kaldi现在的makefile里默认不编译已经停止维护的online和onlinebin解码器文件夹，但是其网络上大部分在线解码识别教程中都是用到这两个文件夹来进行识别的，所以可以编译一下。

进入./kaldi/src/文件夹，修改onlinebin中的makefile文件，修改并确保所有和portaudio有关依赖的路径都设置正确（注意改为对应路径）：

	EXTRA_CXXFLAGS += -Wno-sign-compare -I/home/parallels/Desktop/portaudio/install/include
	
	ifneq "$(wildcard /home/parallels/Desktop/portaudio/install/lib/libportaudio.a)" ""
    	EXTRA_LDLIBS = /home/parallels/Desktop/portaudio/install/lib/libportaudio.a
	else
    	EXTRA_LDLIBS = /home/parallels/Desktop/portaudio/install/lib/libportaudio.a
	endif

在onlinebin文件夹下的makefile中引入alsa库的相关路径：

注意改为自己的对应路径。

	UNAME=$(shell uname)
	ifeq ($(UNAME), Linux)
	  ifneq ($(wildcard ../../tools/portaudio/install/include/pa_linux_alsa.h),)
   		EXTRA_LDLIBS += /home/parallels/Desktop/alsa-lib-1.1.7/lib/libasound.so -lrt
  	  else
    	EXTRA_LDLIBS += -lrt
  	  endif
	endif

进入online文件夹，修改Makefile：

	EXTRA_CXXFLAGS += -Wno-sign-compare -I/home/parallels/Desktop/portaudio/install/include

并将所有的 ../../tools/portaudio/lib/libportaudio.a 修改为 /home/parallels/Desktop/portaudio/install/lib/libportaudio.a

全部修改完后，在onlinebin文件夹下执行：

	make

编译后会生成解码器的两个可执行文件：online-gmm-decode-faster 和 online-wav-gmm-decode-faster。

至此，Ubuntu中的kaldi交叉编译工作就完成了。该编译后的kaldi源码可以移植到任意树莓派设备中并运行；

# 五、kaldi的移植和使用
## 5.1、kaldi移植到树莓派
>将编译好的kaldi源码文件直接拷贝到树莓派文件系统中即可。由于源码工程体积很大，所以使用FileZilla软件传输很慢，建议使用USB3.0的U盘直接进行拷贝。

- 运行kaldi前需要先在树莓派中安装gfortran：

此处安装时需要使用树莓派默认软件源！（树莓派换源了的话需要先换回来）

	sudo apt-get install libgfortran3 gfortran

>关于kaldi源码从电脑中拷贝到树莓派，会出现一个大问题，就是源码中所有快捷方式的复制都会失效，所以如果要运行kaldi/egs/中的例程，需要提前检查例程文件下所有快捷方式，删除并将其替换成同名实体文件才行。这个问题我暂时没找到好办法解决T_T

移植完成后的操作就是kaldi的训练、识别过程了，网上的教程很多，这里不多介绍。只要注意一下失效的快捷方式和相关路径配置问题就好了。

- 如果要在kaldi中实现孤立词识别，可以自行训练模型和利用online-gmm-decode-faster进行解码就可以了，树莓派的性能可以完美支持孤立词识别任务。因为我太懒，所以还没有训练和测试孤立词识别系统_(:з」∠)_，以后有时间可能会尝试。

- 如果要进行大量连续词语音识别，建议使用不超过1GB大小的模型文件，太大的模型会导致内存不足或者识别速度过慢。

关于大量连续词语音识别的模型，建议不要自己训练Thchs30之类的数据集，亲测花了大量时间和精力去训练30个小时的语音，得到的模型还是不能识别我说的话（识别结果可以说是狗屁不通），连续词识别太吃数据量和设备了。所以如果只是想实现比较有准确的大量连续词识别，建议使用[数据堂公司发布的chain模型](http://kaldi-asr.org/models/m10)，700M，亲测是kaldi官网上发布的识别效果最好的中文模型。其他几个中文模型我也测试过，都算不上理想，但其实CVTE发布的模型效果应该更好，但是它太大了，我16GB的电脑都带不动。。。所以目前能用的最好的就是数据堂的这个了。[卡尔迪ASR模型列表](http://kaldi-asr.org/models.html)

## 5.2、在树莓派中测试数据堂chain模型
模型的下载和相关使用方法就不多介绍了，相关说明在模型目录下的tutorial.md文件中有详细说明。测试结果如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/2019112420383061.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2NzA3Njk1,size_16,color_FFFFFF,t_70)
测试音频为我自己录的几段wav和两段thchs30数据集中的语音，可以看到识别的结果还算准确，速度也还可以接受。

由于树莓派嵌入式性能的限制，在载入大模型和寻优的计算上肯定会慢一些，每句话的识别大概会有几秒的延迟。如果想要更快的识别过程，可以考虑对kaldi对应的解码模块进行优化，比如剪枝数目等，或者使用更精炼、更小的模型。当然延迟只是大量连续词识别中的问题，如果只是孤立词，那妥妥的零错误率秒识别。

最终利用树莓派免驱麦克风和编写的脚本（没有用到portaudio）实现kaldi嵌入式平台的大量连续词语音识别效果如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191124144956975.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM2NzA3Njk1,size_16,color_FFFFFF,t_70)
之后会考虑利用该系统去实现语音助手、智能控制、对话机器人等各类功能。

感谢您的阅读！如有错误请您指正！

-2019.11.24




参考文献：

 [1]  [遇逆境处之泰然](https://blog.csdn.net/cj1989111) : [kaldi嵌入式平台的移植及实现](https://blog.csdn.net/cj1989111/article/details/84320162) , CSDN
