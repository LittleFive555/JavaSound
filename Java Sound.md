# Java Sound

摘自：[The Java™ Tutorials](https://docs.oracle.com/javase/tutorial/sound/sampled-overview.html)，翻译为机翻+少量修正

## （一）Sampled包概述

[`javax.sound.sampled`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/package-summary.html)包从根本上涉及音频传输——换句话说，Java Sound API专注于回放和捕获。Java Sound API解决的中心任务是如何将格式化音频数据的字节输入和输出系统。此任务涉及打开音频输入和输出设备，以及管理装有实时声音数据的缓冲区。它还可能涉及将多个音频流混合到一个流中（无论是输入还是输出）。当用户请求启动，暂停，恢复或停止声音流时，必须正确处理声音进出系统的问题。

为了支持对基本音频输入和输出的关注，Java Sound API提供了在各种音频数据格式之间进行转换以及读取和写入常见类型的声音文件的方法。但是，它并不试图成为一个全面的声音文件工具箱。Java Sound API的特定实现不需要支持大量的文件类型或数据格式转换。第三方服务提供商可以提供“插入”现有实现的模块，以支持其他文件类型和转换。

Java Sound API可以以<流式、缓冲方式>和<内存中、非缓冲方式>处理音频传输。在一般意义上，“流”是指音频字节的实时处理。它不涉及<以某种格式通过Internet发送音频>的这种特定的、众所周知的情况。换句话说，音频流只是一组连续的音频字节，它们到达的速度或多或少与处理（播放，录制等）它们时的速度相同。对字节的操作在所有数据到达之前就会开始。在流模型中，尤其是在音频输入而不是音频输出的情况下，您不必事先知道声音多长时间以及它何时到达。您只需一次处理一个音频数据缓冲区，直到操作停止。对于音频输出（播放），如果要播放的声音数据太大而无法一次全部存储在内存中，则还需要缓冲数据。换句话说，您将音频字节以块的形式传递给声音引擎，并且它会在正确的时间播放每个样本。提供的机制使您很容易知道每个块中要传送多少数据。

Java Sound API还只允许在回放的情况下进行无缓冲传输，前提是您已经准备好所有音频数据，并且它不会太大而无法容纳在内存中。在这种情况下，无需应用程序来缓冲音频，虽然需要时仍然可以使用缓冲，实时的方法。相反，可以将整个声音立即预加载到内存中以供后续播放。由于所有声音数据都是预先加载的，因此可以立即开始播放——例如，当用户单击“开始”按钮时。与缓冲模型相比，这是一个优势，在缓冲模型中，回放必须等待第一个缓冲区填满。此外，内存中无缓冲模型使声音可以轻松循环（循环）或设置为数据中的任意位置。

要使用Java Sound API播放或捕获声音，您至少需要三件事：格式化的音频数据，混音器和一行。以下概述了这些概念。

### 什么是格式化音频数据？

格式化的音频数据是指多种标准格式中的任何一种声音。Java Sound API区分*数据格式*和*文件格式*。

#### 数据（Data）格式

数据格式告诉您如何解释一系列字节的“原始”采样音频数据，例如已经从声音文件中读取的采样，或已从麦克风输入捕获的采样。例如，您可能需要知道多少个位构成一个采样（声音的最短瞬间的表示），类似地，您可能需要知道声音的采样率（采样应遵循的速度有多快）。设置播放或捕获时，可以指定要捕获或播放的声音的数据格式。

在Java Sound API中，数据格式由[`AudioFormat`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/AudioFormat.html)对象表示，该 对象包括以下属性：

- 编码技术，通常是脉冲编码调制（PCM）

- 声道数（1个单声道，2个立体声等）

- 采样率（每个通道每秒的采样数）

- 每个样本的位数（每个通道）

- 帧率

- 帧大小（以字节为单位）

- 字节顺序（big-endian or little-endian，大端或小端）

  a) Little-Endian就是**低位字节排放在内存的低地址端，高位字节排放在内存的高地址端**。
  b) Big-Endian就是**高位字节排放在内存的低地址端，低位字节排放在内存的高地址端**。
  c) 网络字节序：**TCP/IP各层协议将字节序定义为Big-Endian，因此TCP/IP协议中使用的字节序通常称之为网络字节序。**

PCM是声音波形的一种编码。Java Sound API包括两种PCM编码，它们使用振幅的线性量化以及有符号或无符号整数值。线性量化意味着存储在每个样本中的数字与该瞬间的原始声压成正比（除任何失真外），并且与该时刻随声音振动的扬声器或鼓膜的位移成正比。例如，光盘使用线性PCM编码的声音。Mu-law编码和a-law编码是常见的非线性编码，可提供音频数据的更多压缩版本。这些编码通常用于电话或语音记录。非线性编码使用非线性函数将原始声音的幅度映射到存储的值，

帧包含某一特定时间的所有通道的数据。对于PCM编码的数据，该帧只是给定时间点内所有通道中同时采样的集合，而没有任何其他信息。在这种情况下，帧速率等于采样率，以字节为单位的帧大小是通道数乘以以位为单位的采样大小，再除以字节中的位数。

对于其他类型的编码，一帧可能除了采样以外还包含其他信息，并且帧速率可能与采样速率完全不同。例如，考虑MP3（MPEG-1音频第3层）编码，在Java Sound API的当前版本中未明确提及，但Java Sound API的实现或第三方可以支持该编码。服务提供者。在MP3中，每个帧都包含一束压缩数据，用于一系列采样，而不仅仅是每个通道一个采样。因为每个帧封装了整个样本序列，所以帧速率比采样速率慢。该框架还包含一个标题。尽管有标头，以字节为单位的帧大小仍小于相等数量的PCM帧的以字节为单位的大小。（毕竟，MP3的目的是比PCM数据更紧凑。

#### 文件（File）格式

文件格式指定声音文件的结构，不仅包括文件中原始音频数据的格式，还包括可以存储在文件中的其他信息。声音文件有各种标准品种，例如WAVE（也称为WAV，通常与PC关联），AIFF（通常与Macintoshes关联）和AU（通常与UNIX系统关联）。不同类型的声音文件具有不同的结构。例如，它们在文件的“标题”中可能具有不同的数据排列方式。标头包含通常在文件实际音频样本之前的描述性信息，尽管某些文件格式允许描述性和音频数据的连续“块”。标头包含用于将音频存储在声音文件中的数据格式的规范。

在Java Sound API中，文件格式由[`AudioFileFormat`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/AudioFileFormat.html)对象表示 ，其中包含：

- 文件类型（WAVE，AIFF等）
- 文件的长度（以字节为单位）
- 文件中包含的音频数据的长度（以帧为单位）
- 一个AudioFormat对象，它指定文件中包含的音频数据的数据格式

[`AudioSystem`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/AudioSystem.html)类提供用于在不同的文件格式中读写声音的方法，以及在不同的数据格式之间转换的方法。某些方法可让您通过称为[`AudioInputStream`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/AudioInputStream.html)的流访问文件的内容 。`AudioInputStream`类是[`InputStream`](https://docs.oracle.com/javase/8/docs/api/java/io/InputStream.html)类的子类，它封装了可以顺序读取的一系列字节。在其超类中，`AudioInputStream`类添加了有关字节的音频数据格式（由一个`AudioFormat`对象表示）的知识。通过将声音文件读取为`AudioInputStream`，您可以立即访问样本，而不必担心声音文件的结构（其标头，块等）。单个方法调用为您提供了所需的有关数据格式和文件类型的所有信息。

### 什么是Mixer（混音器）？

声音的许多应用程序编程接口（API）都使用音频*设备*的概念。设备通常是物理输入/输出设备的软件接口。例如，声音输入设备可能代表了声卡的输入功能，包括麦克风输入，线路电平模拟输入以及数字音频输入。

在Java Sound API中，设备由[`Mixer`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/Mixer.html)对象表示 。混合器的目的是处理一个或多个音频输入流和一个或多个音频输出流。在典型情况下，它实际上将多个传入流混合在一起成为一个传出流。一个`Mixer`对象可以代表诸如声卡之类的物理设备的混音功能，可能需要混合从各种输入端进入计算机的声音，或从应用程序进入的声音，来进行输出。

可替代地，一个`Mixer`对象可以代表完全在软件中实现的混音功能，而没有任何物理设备的固有接口。

在Java Sound API中，诸如声卡上的麦克风输入之类的组件本身并不被视为设备（即混音器），而是被视为混音器的*端口*。端口通常会向混音器中或从混音器中提供单个音频流（尽管该流可以是多声道的，例如立体声）。混合器可能有几个这样的端口。例如，代表声卡输出功能的混音器可能将几个音频流混合在一起，然后将混合后的信号发送到各种连接到混音器的输出端口中的某一个或者全部端口。这些输出端口可以是（例如）耳机插孔，内置扬声器或线路电平输出。

为了了解Java Sound API中混音器的概念，它有助于可视化物理混音控制台，例如在现场音乐会和录音室中使用的那些。



![以下上下文描述了此图。](https://docs.oracle.com/javase/tutorial/figures/sound/chapter2.anc1.gif)

​														**物理调音台**

物理混频器具有“条”（也称为“切片”），每个条代表一条路径，单个音频信号通过该路径进入混频器进行处理。条带具有旋钮和其他控件，通过它们可以控制该条带中的信号的音量和声相（在立体声图像中的相位）。而且，混音器可能具有单独的总线来实现混响等效果，并且该总线可以连接到内部或外部混响单元。每个条带都有一个电位计，用于控制该条带中有多少信号进入混响混音。然后将混响的（“湿”）混合物与来自条的“干”信号混合。物理混音器将此最终混合物发送到输出总线，该输出总线通常到达磁带录音机（或基于磁盘的录音系统）和/或扬声器。

想象一下以立体声录制的现场音乐会。来自舞台上许多麦克风和电子乐器的电缆（或无线连接）插入到调音台的输入中。如图所示，每个输入都进入混频器的单独条带。声音工程师决定增益，声相和混响控件的设置。所有条带的输出和混响单元被混合在一起成为两个通道。这两个通道到达混音器上的两个输出，电缆被插入其中，并连接到立体声磁带录音机的输入。根据音乐类型和大厅的大小，这两个通道也可能通过放大器发送到大厅的扬声器。

现在想象一下一个录音室，其中每个乐器或歌手都被录制到多轨录音机的单独轨道上。乐器和歌手全部录制完毕后，录音工程师执行“混音”操作，将所有录音带组合成一个两通道（立体声）录音，然后将其分发到光盘上。在这种情况下，每个混音器条的输入不是麦克风，而是多轨录音的一个轨。再一次，工程师可以使用控制条上的控件来确定每个音轨的音量，声像和混响量。调音台的输出再次转到立体声录音机和立体声扬声器，如现场音乐会的示例所示。

这两个示例说明了混音器的两种不同用法：捕获多个输入通道，将它们组合成更少的轨道，然后保存混合，或者播放多个轨道，同时将它们混合成更少的轨道。

在Java Sound API中，混合器可以类似地用于输入（捕获音频）或输出（播放音频）。对于输入，混音器从中获取音频进行混音的*源*是一个或多个输入端口。混合器将捕获的和混合的音频流发送到其*目标*，该*目标*是带有缓冲区的对象，应用程序可以从该对象检索此混合音频数据。在音频输出的情况下，情况相反。调音台的音频源是一个或多个包含缓冲区的对象，一个或多个应用程序在其中写入其声音数据。调音台的目标是一个或多个输出端口。

### 什么是Line（线路）？

物理调音台的比喻，有益于了解Java Sound API的 *线路（line）* 的概念。

线路（line）是数字音频“管道”的元素，即用于将音频移入或移出系统的路径。通常，线路（line）是进入或流出混合器的路径（尽管从技术上讲，混合器本身也是一种线路）。

音频的输入和输出端口是线路（line）。这些类似于连接到物理调音台的麦克风和扬声器。另一类线路是数据路径，应用程序可通过该数据路径从混音器获取输入音频或将输出音频发送到混音器。这些数据路径类似于连接到物理调音台的多轨录音机的轨道。

Java Sound API中的线与物理混音器中的线之间的区别是，流过Java Sound API中的线的音频数据可以是单声道或多声道（例如，立体声）。相比之下，物理混音器的每个输入和输出通常是单个声音通道。为了从物理混音器获得两个或多个通道的输出，通常使用两个或多个物理输出（至少在模拟声音的情况下；数字输出插孔通常是多通道的）。在Java Sound API中，一条线路中的通道数由当前流经该线路的数据的[`AudioFormat`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/AudioFormat.html)来指定 。

现在让我们检查一些特定种类的线路和混合器。下图显示了简单的音频输出系统中不同类型的线路，这些线路可以是Java Sound API的实现的一部分：

![以下上下文描述了此图。](https://docs.oracle.com/javase/tutorial/figures/sound/chapter2.anc.gif)

​												**音频输出线的可能配置**

在此示例中，应用程序可以访问音频输入混音器的一些可用输入：一个或多个*片段(clip)*和*源数据线(source data line)*。片段是混音器的输入（一种线路），您可以在播放之前将音频数据加载到其中。源数据线是接收实时音频数据流的混频器输入。应用程序将音频文件中的音频数据预加载到片段中。然后，它将其他音频数据推入源数据线，一次放入一个缓冲区。混音器从所有这些线路读取数据，每条线路可能都有自己的混响，增益和声像控制，并将干音频信号与湿混响混合。混音器将其最终输出传递到一个或多个输出端口，例如扬声器，耳机插孔和线路输出插孔。

尽管各条线在图中均显示为单独的矩形，但它们都是混频器“拥有”的，可以视为混频器的组成部分。混响，增益和声像矩形表示处理控件（而不是线条），混频器可以将这些控件应用于流经线条的数据。

请注意，这仅是API支持的混合器的一个示例。并非所有音频配置都具有所示的所有功能。单个源数据行可能不支持平移，混音器可能未实现混响，等等。A

一个简单的音频输入系统可能类似于：

![以下上下文描述了此图](https://docs.oracle.com/javase/tutorial/figures/sound/chapter2.anc2.gif)

​												**音频输入线的可能配置**

此处，数据从一个或多个输入端口（通常是麦克风或线路输入插孔）流入混频器。应用增益(gain)和声相(pan)，混频器通过混频器的目标数据线将捕获的数据传递到应用程序。目标数据线是混音器输出，其中包含流输入声音的混合。最简单的混合器只有一条目标数据线，但是有些混合器可以将捕获的数据同时传送到多条目标数据线。

#### Line接口层次结构

现在我们已经看到了有关Line和Mixer的一些功能图片，让我们从更具编程性的角度进行讨论。基本[`Line`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/Line.html)接口的子接口定义了几种类型的line线路 。接口层次结构如下所示。

![以下上下文描述了此图](https://docs.oracle.com/javase/tutorial/figures/sound/chapter2.anc3.gif)

​												**Line接口层次结构**

基本接口， [`Line`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/Line.html)描述了所有line共有的最小功能：

- Controls——Data lines和ports通常具有一组Controls，这些Controls会影响通过该line的音频信号。Java Sound API指定了可用于处理声音各方面的control类，例如：gain（增益，影响信号的分贝音量），pan（声相，影响声音的左右位置，reverb（混响，将混响添加到声音来模拟不同种类的室内声学效果）和sample rate（采样率，这会影响播放速率以及声音的音高）。

- 打开或关闭状态——成功打开一条line可确保已将资源分配给该line。Mixer的lines是有限的，因此有时可能多个应用程序（或同一个应用程序），会竞争使用mixer的lines。关闭line表示该line此时使用的任何资源都可以释放。

- Events，事件–一个line打开或关闭时都会生成事件。`Line`的子接口可以引入其他类型的事件。当一个line生成一个事件时，该事件将发送给所有已注册“监听”该线路上事件的对象。应用程序可以创建这些对象，注册它们以监听线路事件，并根据需要对事件做出反应。

  

现在，我们将检查接口`Line`的子接口。

[`Ports`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/Port.html)是用于向音频设备输入音频或从音频设备输出音频的简单线路。如前所述，一些常见的端口类型是麦克风，线路输入，CD-ROM驱动器，扬声器，耳机和线路输出。

[`Mixer`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/Mixer.html)接口代表一个混合器，当然，正如我们所见，它代表一种硬件或软件设备。`Mixer`接口提供了获得混音器线路的方法。其中包括将音频输入到混合器的source lines(源线)和混合器将其混合音频传递出去的target lines(目标线)。对于音频输入混音器，源线是诸如麦克风输入之类的输入端口，而目标线[`TargetDataLines`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/TargetDataLine.html)（如下所述）是将音频传送到应用程序的目标线 。另一方面，对于音频输出混音器，应用程序向其传送音频数据的源线是 [`Clips`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/Clip.html)或 [`SourceDataLines`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/SourceDataLine.html)（如下所述），而目标线是诸如扬声器之类的输出端口。

一个`Mixer`被定义为具有一个或多个源线，和一个或多个目标线。请注意，此定义意味着混合器实际上不需要混合数据。它可能只有一条源数据线路。该`Mixer`API旨在包括各种设备，但典型情况下支持混合。

`Mixer`接口支持同步; 也就是说，您可以将两个或更多混音器的线路视为同步组。然后，您可以通过向组中的任何一条线路发送一条消息来启动，停止或关闭所有这些数据行，而不必分别控制每一条线路。使用支持此功能的混音器，您可以在线路之间获得精确到采样的同步。

通用`Line`接口不提供开始和停止播放或记录的方法。为此，您需要一条data line数据线。该 [`DataLine`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/DataLine.html)接口提供了`Line`的以下与媒体相关的其他附加功能：

- Audio format，音频格式——每条数据线都有与其数据流相关联的音频格式。

- Media position，介质位置——数据行可以报告其在介质中的当前位置，以样本帧表示。这表示自数据线打开以来从数据线捕获或呈现的样本帧数。

- Buffer size，缓冲区大小——这是数据线内部缓冲区的大小（以字节为单位）。对于源数据线，内部缓冲区是可以向其写入数据的缓冲区，对于目标数据线，内部缓冲区是可以从其读取数据的缓冲区。

- level，电平（音频信号的当前幅度）

- 开始和停止playback播放或capture捕获

- 暂停和继续播放或捕获

- Flush，刷新（从队列中丢弃未处理的数据）

- Drain，清空（阻塞方式运行，直到所有未处理的数据从队列中清空，并且数据行的缓冲区变空）

- Active status，活动状态——如果数据线参与到音频混合器中的音频数据的主动提交或捕获，则视为活跃的。

- Events，事件——当在数据线上开始和停止提交或捕获数据时，将产生`START`和`STOP`事件（这里很绕，不会翻译，贴上原文：`START` and `STOP` events are produced when active presentation or capture of data from or to the data line starts or stops.）。

`TargetDataLine`从mixer接收音频数据。通常，mixer已从麦克风等端口捕获了音频数据。在将数据放入目标数据线的缓冲区之前，它可能会处理或混合捕获的音频。`TargetDataLine`接口提供了从目标数据线的缓冲区读取数据并确定当前有多少数据可读取的方法。

`SourceDataLine`接收音频数据用于播放。它提供了将数据写入源数据线路的缓冲区以进行播放，以及确定该线路准备接收多少数据而不会阻塞的方法。

`Clip`是一条数据线，可以在播放之前将音频数据加载到其中。由于数据是预加载的而不是流式传输的，因此在播放之前会知道clip的持续时间，您可以选择媒体中的任何起始位置。clip可以循环播放，这意味着在播放时，两个指定循环点之间的所有数据将重复指定次数或无限期重复。

本节介绍了sampled-audio API的大多数重要接口和类。随后的部分说明如何在应用程序中访问和使用这些对象。

## （二）访问音频系统资源

Java Sound API采用了灵活的系统配置方法。可以在计算机上安装不同种类的音频设备（mixers）。该API没怎么设想（makes few assumptions?）有什么已安装的设备以及其有什么功能。相反，它提供了让系统报告可用音频组件的方法以及让程序访问它们的方法。

以下各节说明你的程序如何了解计算机上已安装了哪些采样音频资源，以及如何获取对可用资源的访问。其中，资源包括混合器和混合器所拥有的各种类型的线路。

### AudioSystem类

[`AudioSystem`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/AudioSystem.html)类充当音频组件（audio components）的交流中心，音频组件包括来自第三方供应商内置服务和单独安装的服务。`AudioSystem`用作访问这些已安装的采样音频资源的应用程序的入口点。您可以查询`AudioSystem`来了解已安装了哪些资源，然后可以访问它们。例如，应用程序可能会通过询问`AudioSystem`类是否存在具有特定配置的混合器来开始，例如在前面的讨论中已说明的输入或输出配置之一。然后，程序将从混合器中获取数据线，依此类推。

以下是应用程序可以从`AudioSystem`中获得的一些资源：

- Mixers，混音器——系统通常安装了多个混音器。通常至少有一个用于音频输入，一个用于音频输出。可能还有一些混音器没有I / O端口，而是接受来自应用程序的音频，并将混合的音频传递回该程序。本`AudioSystem`类提供所有已安装的混音器的列表。
- Lines，线路——即使每条线路都与一个混合器相关联，应用程序也可以直接从`AudioSystem`中获得一条线路，而无需明确地和混音器打交道。
- Format conversions，格式转换——应用程序可以使用格式转换将音频数据从一种格式转换为另一种格式。
- Files and streams，文件和流——`AudioSystem`类提供在音频文件和音频流之间进行转换的方法。它还可以报告声音文件的文件格式，并可以以不同的格式写入文件。

### Information Objects，信息对象

Java Sound API中的几个类提供了关于相关接口的有用信息。例如， [`Mixer.Info`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/Mixer.Info.html)提供有关已安装的混音器的详细信息，例如混音器的供应商，名称，描述和版本。 [`Line.Info`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/Line.Info.html)获取特定线路的类。`Line.Info`的子类包括 [`Port.Info`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/Port.Info.html)和[`DataLine.Info`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/DataLine.Info.html)，分别获取与特定端口和数据线路有关的详细信息。这些类中的每一个都将在下面的相应部分中进行进一步说明。重要的是不要将`Info`对象与它所描述的混音器或线路对象混淆。

#### Getting a Mixer，获取一个混音器

**通常，使用Java Sound API的程序需要做的第一件事是获取一个混音器**，或者至少是一个混音器的一条线路，以便您可以将声音传入或传出计算机。您的程序可能需要一个特定类型的混音器，或者您可能希望显示所有可用混音器的列表，以便用户可以选择一个。无论哪种情况，您都需要了解安装了哪些类型的混音器。`AudioSystem`提供以下方法：

```java
static Mixer.Info[] getMixerInfo()
```

此方法返回的每个[`Mixer.Info`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/Mixer.Info.html)对象都标识一种已安装的混音器。（通常，系统最多只有一个给定类型的混合器。如果碰巧有不止一种给定类型的混合器，则返回的数组仍然只有`Mixer.Info`该类型的一个。）应用程序可以遍历`Mixer.Info`对象，根据需要来找到一个合适的混音器。`Mixer.Info`包括以下字符串，以确定一种混音器：

- Name，名称
- Version，版本
- Vendor，供应商
- Description，描述

这些是可以随意设定内容的字符串，因此需要特定混合器的应用程序必须知道期望的内容，和要去比对内容的字符串（难翻译， 给原文：what to expect and what to compare the strings to.）。提供混音器的公司应在其文档中包含此信息。替代地，并且可能更典型的情况是，应用程序将向用户显示所有`Mixer.Info`对象的字符串，并让用户选择相应的混合器。

找到合适的混合器后，应用程序将调用以下`AudioSystem`的方法以获得所需的混音器Mixer：

```java
static Mixer getMixer(Mixer.Info info)
```

如果您的程序需要具有某些功能的混音器，而不需要特定供应商生产的特定混音器，该怎么办？而且，如果您不认为用户知道应该选择哪种混音器的时候呢？在这种情况下，`Mixer.Info`对象中的信息将没有太大用处。相反，您可以遍历`getMixerInfo`返回的所有`Mixer.Info`对象，通过调用`getMixer`为每个`Info`对象获取一个混合器，并向每个混合器查询其功能。例如，您可能需要一个混音器，它可以将其混合音频数据同时写入到一定数量的目标数据线。在这种情况下，您将使用此`Mixer`的方法查询每个混音器：

```java
int getMaxLines(Line.Info info)
```

在这里， [`Line.Info`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/Line.Info.html)将指定一个`TargetDataLine`。的`Line.Info`类将在下节讨论。

#### 获取所需类型的线路line

有两种方法可以获得一条line线路：

- 直接从 [`AudioSystem`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/AudioSystem.html)对象中获得
- 从您已经从`AudioSystem`对象获得的混合器中获得

##### 直接从AudioSystem获得线路

假设您尚未获得混合器，并且您的程序是一个简单的程序，实际上只需要某种类型的线路line；调音台的详细信息对您而言并不重要。您可以使用以下`AudioSystem`的方法：

```Java
static Line getLine(Line.Info info)
```

与前面讨论的`getMixer`方法类似。与[`Mixer.Info`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/Mixer.Info.html)不同的是 ， 用作参数的[`Line.Info`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/Line.Info.html)不会存储文本信息来指定所需的行。相反，它存储有关所需线路类的信息。

`Line.Info`是一个抽象类，因此您可以使用其子类之一（ [`Port.Info`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/Port.Info.html)或 [`DataLine.Info`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/DataLine.Info.html)）获取一条线路line。以下的节选代码使用`DataLine.Info`子类获取并打开目标数据线路：

```java
TargetDataLine line;
DataLine.Info info = new DataLine.Info(TargetDataLine.class, 
    format); // format是一个AudioFormat类型的对象
if (!AudioSystem.isLineSupported(info)) {
    // 处理错误.
}
try {		// 获取并打开线路line.
    line = (TargetDataLine) AudioSystem.getLine(info);
    line.open(format);
} catch (LineUnavailableException ex) {
        // 处理错误.
    //... 
}
```

此代码获取一个 [`TargetDataLine`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/TargetDataLine.html)对象，而不指定其类和音频格式以外的任何属性。您可以使用类似的代码来获取其他类型的行。对于`SourceDataLine`类或`Clip`类，只需将line变量的`TargetDataLine`类替换为`SourceDataLine`或`Clip`，并在`DataLine.Info`构造函数的第一个参数中进行替换。

对于`Port`类，您可以在如下代码中使用`Port.Info`的静态实例：

```java
if (AudioSystem.isLineSupported(Port.Info.MICROPHONE)) {
    try {
        line = (Port) AudioSystem.getLine(
            Port.Info.MICROPHONE);
    }
}
```

请注意使用`isLineSupported`方法来查看混合器是否具有所需类型的线。

回想一下，一个源数据线路是混音器的输入，如果混音器代表一个音频输入设备，那么源数据线路可以代表一个`Port`对象，或者如果混音器代表一个音频输出装置，那么源数据线路可以是一个`SourceDataLine`或`Clip`对象。类似地，目标线路是混音器的输出：用于音频输出混音器的`Port`对象，和用于音频输入混频器的`TargetDataLine`对象。如果调音台根本不连接任何外部硬件设备怎么办？例如，考虑一个内部的或纯软件的混音器，它从应用程序中获取音频并将其混合后的音频传递回该程序。这种混合器的输入线具有`SourceDataLine`或`Clip`对象，输出线具有`TargetDataLine`对象。

您还可以使用以下`AudioSystem`方法来了解有关任何已安装的混音器支持的指定类型的源线路和目标线路的更多信息：

```java
static Line.Info[] getSourceLineInfo(Line.Info info)
static Line.Info[] getTargetLineInfo(Line.Info info)
```

请注意，这些方法中的每一个返回的数组元素都表示线路的唯一种类，而不一定是所有的线路。（不太会翻译，给原文：Note that the array returned by each of these methods indicates unique types of lines, not necessarily all the lines.）例如，如果混音器的两条线或两条不同混音器的线具有相同的`Line.Info`对象，则这两条线在返回的数组中将仅由一个`Line.Info`表示。

##### 从混音器Mixer获得线路

`Mixer`接口包括上述`AudioSystem`用于访问源线和目标线的方法的变体。这些`Mixer`方法包括采用`Line.Info`作为参数的方法，就像`AudioSystem`的方法一样。但是，`Mixer`还包括以下变体，它们不带任何参数：

```java
Line.Info[] getSourceLineInfo()
Line.Info[] getTargetLineInfo()
```

这些方法返回特定混合器的所有`Line.Info`对象的数组。一旦获得数组，就可以遍历它们，调用`Mixer`的 `getLine`方法来获取每一条线路，然后调用`Line`的 `open`方法来在你的程序中使用各条线路。

#### 选择输入和输出端口

上一节有关如何获得所需类型的线的方法，也适用于端口以及其他类型的线。您可以通过将一个`Port.Info`对象传递给`AudioSystem`（或`Mixer`）的，可以接受一个`Line.Info`作为参数的`getSourceLineInfo`和`getTargetLineInfo`方法，来获取所有的源端口（即输入端口）和目标端口（即输出端口）。然后，您遍历返回的对象数组，并调用Mixer的`getLine`方法来获取每个端口。

然后可以通过调用`Line`的 `open`方法打开每个`Port`。打开端口意味着您打开了端口的开关——也就是说，您允许声音进入或流出端口。同样，您可以关闭不希望声音传播的端口，因为某些端口可能在您获得它们之前就已经打开。某些平台默认情况下将所有端口保留为打开状态；或者用户或系统管理员可能已使用其他应用程序或操作系统软件选择了将某些端口打开或关闭。

**警告：**如果要选择某个端口并确保声音确实正在传入或传出该端口，则可以按照说明打开该端口。但是，这可以视为对用户有害的行为！例如，用户可能会关闭扬声器端口，以免打扰她的同事。如果您的程序突然破坏了她的愿望并开始大肆歌唱，她会很不高兴。举另外一个例子，用户可能希望确保自己的计算机的麦克风永远不会在不知情的情况下打开，以避免窃听。通常，建议不要打开或关闭端口，除非您的程序响应用户的意图（通过用户界面表示）。相反，请遵守用户或操作系统已选择的设置。

所连接的混音器正常运行之前，无需打开或关闭端口。例如，即使所有输出端口都已关闭，您也可以开始将声音回放到音频输出混音器中。数据仍然会流入混合器，播放不会被阻止，用户只是什么也听不到。用户一旦打开输出端口，便会从该端口上的声音开始听到声音，该声音将从媒体播放到的某处开始。

另外，您无需访问端口即可了解混音器是否具有某些端口。例如，要了解某个混音器是否是音频输出混音器，可以调用`getTargetLineInfo`以查看它是否具有输出端口。除非您要更改端口设置（例如，端口的打开或关闭状态或它们可能具有的任何控件controls的设置），否则没有理由访问端口本身。

#### 音频资源使用许可

Java Sound API包含一个 [`AudioPermission`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/AudioPermission.html)类，该类指示小程序（或运行有安全管理器的应用程序）可以对采样音频系统进行哪种访问。录制声音的权限是单独控制的。应当谨慎授予此权限，以帮助防止诸如未经授权的窃听之类的安全风险。默认情况下，小程序和应用程序被授予以下权限：

- 一个运行在小程序安全管理器下的*小程序*可以播放音频，但不能录制音频。
- 没有安全管理器的*应用程序*可以播放和录制音频。
- 使用默认安全管理器运行的应用程序可以播放但不能录制音频。

通常，小程序在安全管理器的监督下运行，并且不允许录制声音。另一方面，应用程序不会自动安装安全管理器，这样是能够录制声音的。（但是，如果为应用程序显式调用了默认安全管理器，则不允许该应用程序录制声音。）

即使已被授予显式权限，小应用程序和应用程序也可以在与安全管理器一起运行时记录声音。

如果您的程序无权录制（或播放）声音，则在尝试打开一条线路时会引发异常。除了捕获异常并将问题报告给用户以外，您在程序中无能为力，因为无法通过API来更改权限。（如果可以，那它们将毫无意义，因为这样的话将没有什么安全的方法了！）通常，权限是在一个或多个策略配置文件中设置的，用户或系统管理员可以使用文本编辑器或Policy Tool程序对其进行编辑。

## （三）播放音频

回放(playback)有时称为*演示(presentation)*或*渲染(rendering)*。这些是通用术语，也适用于音频以外的其他类型的媒体。基本特征是将数据序列传递到某个地方，以供用户最终感知。如果数据是基于时间的（如音频一样），则必须以正确的速率传送。音频甚至比视频要多，因此保持数据流的速率非常重要，因为声音播放的中断通常会产生很大的喀哒声(clicks)或令人讨厌的失真。Java Sound API旨在帮助应用程序平稳连续播放音频，即使是很长的音频。

之前，您了解了如何从音频系统或调音台获得线路。在这里，您将学习如何通过一条线播放声音。

如您所知，可以使用两种线路来播放声音：[`Clip`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/Clip.html)和[`SourceDataLine`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/SourceDataLine.html)。两者之间的主要区别在于，使用`Clip`可以一次指定所有声音数据，然后再播放，而使用`SourceDataLine`可以在播放过程中连续写入新的数据缓冲。尽管在很多情况下都可以在`Clip`或`SourceDataLine`之间随意选用，但以下条件有助于确定哪种线更适合特定情况：

- 使用`Clip`时，你有非实时声音数据可以预先加载到内存中。

  例如，您可能将短音频文件读入clip。如果您想播放音频不止一次，则`Clip`比`SourceDataLine`更方便，尤其是当您要播放循环（重复播放全部或部分声音）时。如果您需要在声音中的任意位置开始播放，则`Clip`接口提供了一种轻松进行播放的方法。最后，从`Clip`播放的延迟通常比从`SourceDataLine`播放的缓冲的延迟少。换句话说，由于声音已预先加载到clip中，因此可以立即开始播放，而不必等待缓冲区被填满。

- 将`SourceDataLine`用作流数据，例如无法一次放入内存的长声音文件，或者在播放之前无法得知其数据的声音。

  作为后一种情况的示例，假设您正在监视声音输入——即在捕获声音时播放声音。如果您没有可以直接从输出端口发送回输入音频的混音器，则您的应用程序将必须获取捕获的数据并将其发送到音频输出混音器。在这种情况下，`SourceDataLine`比`Clip`更合适。当您响应用户的输入以交互方式合成或操纵声音数据时，会发生另一个无法预先知道的声音示例。例如，想象一个游戏，当用户移动鼠标时，通过将声音从一种声音“变形”到另一种声音来给出听觉反馈。声音转换的动态性质要求应用程序在播放期间不断更新声音数据，而不是在播放开始前一次性提供到位。

### 使用Clip

您可以如前面在《[获取所需类型的行](https://docs.oracle.com/javase/tutorial/sound/accessing.html#113154)》中所述获得`Clip`；为第一个参数构造一个`DataLine.Info`对象`Clip.class`，并将`DataLine.Info`作为参数传递给`AudioSystem`或`Mixer`的`getLine`方法。

获得一条线只是意味着您已经有了一种引用它的方式。`getLine`实际上并没有为您预留线路。由于混音器可能只有有限数量的所需类型的线路，因此可能发生在调用`getLine`获取clip之后，正准备播放音频时，另一个应用程序会插一脚进来并抢走clip。要实际使用clip，您需要通过调用以下`Clip`方法之一，将其保留供程序专用：

```java
void open(AudioInputStream stream)
void open(AudioFormat format, byte[] data, int offset, int bufferSize)
```

尽管上面第二种`open`方法中有参数`bufferSize`，`Clip`（与`SourceDataLine`不同）并没有用于将新数据写入缓冲区的方法。这里的`bufferSize`参数仅指定要加载到clip中的字节数组的数量。它不像使用`SourceDataLine`时的缓冲区那样，让您可以将更多数据加载到其中。

打开clip后，您可以使用`Clip`的`setFramePosition`或`setMicroSecondPosition`方法指定应在数据中的哪一点开始播放。否则，它将从头开始。您也可以使用`setLoopPoints`方法将播放设置为重复循环。

准备开始播放时，只需调用该`start`方法。要停止或暂停clip，请调用`stop`方法，如果要继续播放，则再次调用`start`方法。clip会记住停止播放的媒体位置，因此不需要显式的暂停和恢复方法。如果您不希望它在中断处继续播放，则可以使用上述提到的<u>帧或微秒定位</u>方法将剪辑“倒回”到开头（或其他任何位置）。

`Clip`的音量和活跃状态（活跃与非活跃）可以分别通过调用`DataLine`的`getLevel`和`isActive`方法来观测。正在播放音频的`Clip`是活跃的。

### 使用SourceDataLine

获得一个`SourceDataLine`与获得一个`Clip`相似。打开`SourceDataLine`也类似于打开`Clip`，目的是再次保留该线路。但是，您可以使用另一种方法，该方法继承自`DataLine`：

```java
void open(AudioFormat format)
```

请注意，打开`SourceDataLine`时，您尚未将任何声音数据与线路关联，这与打开`Clip`不同。相反，您只需指定要播放的音频数据的格式。系统使用默认缓冲区长度。

您还可以使用此变体方法规定一定的缓冲区长度（以字节为单位）：

```java
void open(AudioFormat format, int bufferSize)
```

为了与类似方法保持一致，该`bufferSize`参数以字节表示，但必须与整数个帧相对应。

除了使用上述open方法之外，还可以使用`Line`的不带参数的`open()`方法打开`SourceDataLine`。在这种情况下，将以其默认音频格式和缓冲区大小打开该线路。但是，您之后将无法更改它们。如果您想知道线路的默认音频格式和缓冲区大小，则可以调用`DataLine`的 `getFormat`和`getBufferSize`方法，甚至在线路打开之前就可以调用。

一旦`SourceDataLine`打开，就可以开始播放声音。为此，您可以调用`DataLine`的start方法，然后将数据重复写入该线路的播放缓冲区。

start方法允许线路在缓冲区中有任何数据后立即开始播放声音。您可以通过以下方法将数据放入缓冲区：

```java
int write(byte[] b, int offset, int length)
```

数组的偏移量以字节表示，数组的长度也是如此。

该线路开始尽快地将数据发送到混音器。当混音器本身将数据传递到其目标时，`SourceDataLine`生成一个`START`事件。（在Java Sound API的典型实现中，源代码线路将数据提供给混合器的那一刻与混合器将数据提供给其目标的那一刻之间的延迟可以忽略不计，即，该延迟远小于一个采样时间。该`START`事件被发送到该线路的侦听器，如下面在《[监视线路状态](https://docs.oracle.com/javase/tutorial/sound/playing.html#113711)》中所述。现在该线路被认为是活跃的，因此`DataLine`的`isActive`方法将返回`true`。注意，所有这些仅在缓冲区包含要播放的数据时发生，而不一定是在调用start方法时。如果您在新的`SourceDataLine`调用了`start`，但从未将数据写入缓冲区，因此该行永远不会处于活动状态，`START`也永远不会发送事件。（但是，在这种情况下，`DataLine`的`isRunning`方法将返回`true`。）

那么，您如何知道要向缓冲区写入多少数据，以及何时发送第二批数据呢？幸运的是，您无需计算第二次写入操作的时间，即可与第一个缓冲区的末尾同步！相反，您可以利用该`write`方法的阻塞行为：

- 一旦将数据写入缓冲区，该方法即返回。它不会等到缓冲区中的所有数据播放完毕。（如果它等了，您可能没有办法在不造成音频不连续的情况下写入下一个缓冲。）
- 可以尝试写入比缓冲区更多的数据。在这种情况下，该方法将阻塞（不返回），直到您实际请求的所有数据都已放入缓冲区中为止。换句话说，一次将一个缓冲中有价值的数据写入缓冲区并进行播放，直到剩余的数据全部放入缓冲区中为止，此时该方法返回。无论该方法是否阻塞，只要可以写入此调用中最后一个缓冲区的数据，它将立即返回。同样，这意味着您的代码很可能会在最后一个缓冲区的数据播放完成之前恢复控制。
- 虽然在许多情况下可以写入比缓冲区容纳的数据更多的数据，但是如果您想确定下一次发出的写操作不会阻塞，则可以将写入的字节数限制为`DataLine`的 `available`方法返回的数字。

这是一个示例，它循环访问从流中读取的大块数据，一次将一个大块写入`SourceDataLine`以便回放：

```java
//从流中读取大块并将其写入源数据线路
line.start();
while (total < totalToRead && !stopped){
    numBytesRead = stream.read(myData, 0, numBytesToRead);
    if (numBytesRead == -1) break;
    total += numBytesRead; 
    line.write(myData, 0, numBytesRead);
}
```

如果您不希望该`write`方法阻塞，则可以先调用该`available`方法（在循环内部），以找出不阻塞即可写入的字节数，然后在从流中读取数据之前将字节数限制为变量`numBytesToRead`。但是，在给定的示例中，阻塞并不重要，因为write方法是在循环内调用的，直到最后一次循环迭代中写入最后一个缓冲区后，该方法才会完成。无论您是否使用阻塞技术，您都可能希望在与应用程序其余部分分离的线程中调用此播放循环，以使您的程序在播放长音频时不会冻结。在循环的每次迭代中，您可以测试用户是否已请求停止播放。此类请求需要设置上面的代码中使用的boolean型变量`stopped`为`true`。

由于`write`在所有数据播放完毕之前返回，那么您如何知道实际播放完成的时间呢？一种方法是在写入最后一个缓冲区的数据后调用`DataLine`的`drain`方法。此方法将阻塞，直到播放完所有数据为止。当控制权返回到程序时，如果需要，您可以释放该线路，而不必担心过早中断任何音频样本的播放：

```java
line.write(b, offset, numBytesToWrite); 
//这是对write的最后一次调用
line.drain();
line.stop();
line.close();
line = null;
```

当然，您可以故意过早停止播放。例如，应用程序可能会向用户提供“停止”按钮。调用`DataLine`的`stop`方法可立即停止播放，即使在缓冲区中间也是如此。这会将所有未播放的数据保留在缓冲区中，因此，如果您随后调用`start`，则会从中断处继续播放。如果那不是您想发生的事情，则可以通过调用`flush`丢弃缓冲区中剩余的数据。

每当停止数据流时，无论是通过`drain`方法，`stop`方法还是`flush`方法进行了该停止操作，还是因为在应用程序再次调用`write`写入新数据之前已到达播放缓冲区的末尾，`SourceDataLine`都会生成一个`STOP`事件。一个`STOP`事件并不一定意味着`stop`被调用，也并不一定意味着之后调用`isRunning`将返回`false`。但是，这确实意味着`isActive`将返回`false`。（当`start`被调用后，`isRunning`方法就返回`true`，即使生成了`STOP`事件；在调用`stop`后`isRunning`就返回`false`。）重要的是要认识到`START`和`STOP`事件对应于`isActive`，而不是`isRunning`。

### 监控线路状态

开始播放音频后，如何知道完成的时间呢？我们在上面看到了一个解决方案，在写入最后一个数据缓冲区后调用`drain`方法，但是该方法仅适用于`SourceDataLine`。对于`SourceDataLines`和`Clips`都适用的另一种方法是，进行注册，每当线路改变了它的状态时，接收来自线路的通知。这些通知以`LineEvent`对象的形式来组织，其中有四种类型：`OPEN`，`CLOSE`，`START`和`STOP`。

程序中实现`LineListener`接口的任何对象都可以注册以接收此类通知。要实现`LineListener`接口，该对象仅需要一个带有`LineEvent`参数的update方法。若要将此对象注册为该线路的侦听器之一，请调用`Line`的以下方法：

```java
public void addLineListener(LineListener listener)
```

每当该线路打开open，关闭close，开始start或停止stop时，它都会向其所有侦听器发送一条`update`消息。您的对象可以查询它收到的`LineEvent`。首先，您可以调用`LineEvent.getLine`以确保停止的线路是您想要的那一条。在我们讨论的这种情况下，您想知道音频是否结束，因此您可以查看`LineEvent`是否为`STOP`类型。如果是的话，您可以检查音频的当前位置（音频也存储在`LineEvent`对象中），并将其与声音的长度（如果知道）进行比较，以查看音频是否到达结尾，而不是因为其他的原因（例如用户单击“停止”按钮，尽管您可能可以在代码的其他位置确定该原因）。

同样，如果您需要知道何时打开open，关闭close或开始start该线路，则可以使用相同的机制。`LineEvents`会由各种不同类型的线路生成，而不仅仅是`Clips`和`SourceDataLines`。但是对于`Port`，您不能指望通过获取事件来了解线路的打开或关闭状态。例如，`Port`在最初创建时可能就是打开的，因此您无需调用该`open`方法，并且`Port`也不会生成`OPEN`事件。（请参阅前面有关《[选择输入和输出端口](https://docs.oracle.com/javase/tutorial/sound/accessing.html#113216)》的讨论 。）

### 同步多线路播放

如果要同时播放多个音频轨道，则可能要使它们完全在同一时间开始和停止。一些混合器用它们的`synchronize`的方法促进这种行为，它允许使用例如`open`，`close`，`start`和`stop`操作，一组线路使用单个命令控制，而不必单独控制每个数据线路。此外，将操作应用于线路的精确度是可控制的。

要找出特定的混频器是否为指定的一组数据线路提供此功能，请调用`Mixer`接口的`isSynchronizationSupported`方法：

```java
boolean isSynchronizationSupported(Line[] lines, boolean  maintainSync)
```

第一个参数指定一组特定的数据线，第二个参数表示必须保持同步的精度。如果第二个参数是`true`，该查询询问混合器是否能够*一直*保持在指定的线路控制样本精确的精度 ; 否则，仅在开始和停止操作期间需要精确的同步，而在整个播放过程中则不需要。

<u>我想，第二个参数的意思是，true代表每时每刻都保持样本的精确同步，false代表只在开始和停止的时候保证同步。</u>

### 处理输出音频

某些源数据线具有信号处理控件，例如增益gain，声相pan，混响reverb和采样率sample-rete控件。类似的控件，尤其是增益控件，也可能出现在输出端口上。有关如何确定线路是否具有此类控件以及如何使用它们的更多信息，请参阅《[使用控件处理音频](https://docs.oracle.com/javase/tutorial/sound/controls.html)》。

## （四）捕获音频

*捕获*是指从计算机外部获取信号的过程。音频捕获的常见应用是录制，例如将麦克风输入录制到音频文件中。但是，捕获并不等同于录制，因为录制意味着应用程序始终会保存传入的声音数据。捕获音频的应用程序不一定会存储音频。取而代之的是，它可能会对传入的数据进行某些处理（例如将语音转录为文本），但是一旦音频缓冲结束，就立即丢弃每个音频缓冲。

如 《[样本包概述](https://docs.oracle.com/javase/tutorial/sound/sampled-overview.html)》中所述，Java Sound API实现中的典型音频输入系统包括：

1. 输入端口，例如麦克风端口或输入端口，音频数据从其流入。
2. 混合器，可以把输入数据放置在其中。
3. 一个或多个目标数据行，应用程序可以从中检索数据。

不太会翻译这三条，以下为原文：

1. An input port, such as a microphone port or a line-in port, which feeds its incoming audio data into:
2. A mixer, which places the input data in:
3. One or more target data lines, from which an application can retrieve the data.

通常，一次只能打开一个输入端口，但是也可以使用音频输入混合器来混合来自多个端口的音频。另一种情况是混音器没有端口，而是通过网络获取音频输入。

《[线路接口层次](https://docs.oracle.com/javase/tutorial/sound/sampled-overview.html#lineHierarchy)》下简要介绍了`TargetDataLine`接口。`TargetDataLine`与`SourceDataLine`接口直接相似，在《[播放音频](https://docs.oracle.com/javase/tutorial/sound/playing.html)》中对此进行了广泛的讨论 。回想一下该`SourceDataLine`接口包括：

- 一种将音频发送到调音台的`write`方法
- 一种确定可以在不阻塞的情况下将多少数据写入缓冲区的`available`方法

同样，`TargetDataLine`包括：

- 一种从混音器获取音频的`read`方法
- 一种确定在不阻塞的情况下可以从缓冲区读取多少数据的`available`方法

### 设置TargetDataLine

《[访问音频系统资源](https://docs.oracle.com/javase/tutorial/sound/accessing.html)》中描述了获取目标数据线的过程， 但为方便起见，在此再次列出该过程：

```java
TargetDataLine line;
DataLine.Info info = new DataLine.Info(TargetDataLine.class, 
    format); // format is an AudioFormat object
if (!AudioSystem.isLineSupported(info)) {
    // Handle the error ... 

}
// Obtain and open the line.
try {
    line = (TargetDataLine) AudioSystem.getLine(info);
    line.open(format);
} catch (LineUnavailableException ex) {
    // Handle the error ... 
}
```

您可以改为调用`Mixer`的`getLine`方法，而不是`AudioSystem`的。

如本例所示，一旦获得目标数据线，就可以通过调用`SourceDataLine`的`open`方法将其保留给应用程序使用，这与《[播放音频](https://docs.oracle.com/javase/tutorial/sound/playing.html)》中源数据线的情况完全相同 。`open`方法的单参数版本使线路的缓冲区使用默认大小。您可以根据应用程序的需要，通过调用两个参数的版本来设置缓冲区大小：

```java
void open(AudioFormat format, int bufferSize)
```

### 从TargetDataLine读取数据

线路打开后，就可以开始捕获数据了，但是它还不处于活跃状态。要实际开始音频捕获，请使用`DataLine`的`start`方法。这开始将输入的音频数据传送到线路的缓冲区，以供您的应用程序读取。您的应用程序仅在准备好开始从线路开始读取时，才应该调用`start`；否则，浪费大量的处理时间来填充捕获缓冲区，只是使其溢出（即丢弃数据）。

要开始从缓冲区检索数据，请调用`TargetDataLine`的`read`方法：

```java
int read(byte[] b, int offset, int length)
```

此方法尝试从数组中的字节位置`offset`开始，将`length`长度字节的数据读取到数组`b`中。该方法返回实际读取的字节数。

与`SourceDataLine`的`write`方法一样，您可以请求传输超出缓冲区中的实际容量的数据量，因为即使您请求传输很多的缓冲数据，方法也会阻塞直到传递了请求的数据量为止。

为避免应用程序在录制过程中挂起，可以在循环中调用`read`方法，直到检索到所有音频输入为止，如以下示例所示：

```java
// 假设已经获取并打开了TargetDataLine, line 
ByteArrayOutputStream out  = new ByteArrayOutputStream();
int numBytesRead;
byte[] data = new byte[line.getBufferSize() / 5];

// 开始捕获音频.
line.start();

// 在这里，stopped是另一个线程设置的全局布尔值
while (!stopped) {
   // 从TargetDataLine读取下一个数据块.
   numBytesRead =  line.read(data, 0, data.length);
   // 保存此数据块.
   out.write(data, 0, numBytesRead);
}     
```

请注意，在此示例中，将读取数据的字节数组的大小设置为了线路缓冲区的大小的五分之一。如果改为将其设置为与线路的缓冲区一样大并尝试读取整个缓冲区，则需要非常精确的时序，因为如果混音器从线路中读取数据时向线路中传输数据，那么数据将会被丢弃 。如此处所示，通过使用线路缓冲区大小的一部分，您的应用程序将可以更好地与混音器共享对线路缓冲区的访问。

`TargetDataLine`的`read`方法有三个参数：一个字节数组，在数组中的偏移量，以及你想要读取的数据的字节数。在此示例中，第三个参数只是字节数组的长度。该`read`方法返回实际读入数组的字节数。

通常，如本例所示，您是从循环中的线路line中读取数据。在`while`循环内，以任何适于应用程序的方式处理检索到的每个数据块——在这个例子中，将它们写入了`ByteArrayOutputStream`。这里未显示使用单独的线程来设置boolean型变量 `stopped`来终止循环。当用户单击“停止”按钮时，以及在侦听器从该线路接收到`CLOSE`或`STOP`事件时，都可以将该布尔值设置为`true`。对于`CLOSE`事件，侦听器是必需的，对于`STOP`事件，也建议使用侦听器。否则，如果在某种程度上停止了该线路而没有将其设置为`true`，`while`循环将在每次迭代中捕获零字节，从而运行很快并且浪费CPU周期。一个更详尽的代码示例将显示如果捕获动作再次变为活动状态，则重新进入循环。

与源数据线一样，可以`drain`或`flush`目标数据线。例如，如果要将输入记录到文件中，则可能需要在用户单击“停止”按钮时调用`drain`方法。`drain`方法将导致混频器的剩余数据被传递到目标数据线的缓冲区。如果不清除数据，捕获的音频可能会在尾部被过早地截断。

在某些情况下，您可能想刷新`flush`数据。无论如何，如果您既不刷新也不清空数据，则数据将留在混频器中。这意味着当重新开始捕获时，在新录制的开始时会有一些剩余的声音，这可能是不希望的。所以，重新启动捕获之前刷新`flush`目标数据线路可能很有用。

### 监控线路状态

因为`TargetDataLine`接口是继承于`DataLine`，所以目标数据线以与源数据线相同的方式生成事件。您可以注册一个对象，以便在目标数据线路打开open，关闭close，开始start或停止stop时接收事件。有关更多信息，请参见前面有关《[监视线路状态](https://docs.oracle.com/javase/tutorial/sound/playing.html#113711)》的讨论 。

### 处理传入的音频

像某些源数据线一样，某些混频器的目标数据线也具有信号处理控件（signal-processing controls），例如增益（gain），声像（pan），混响（reverb）或采样率控件（sample-rate controls）。输入端口可能也具有类似的控件，尤其是增益控件。在下一节中，您将学习如何确定某行是否具有此类控件，以及如何使用它们。

## （五）使用控件处理音频

前面的部分讨论了如何播放或捕获音频样本。隐含的目标是尽可能忠实地提供样本，而不进行修改（除了可能将样本与其他音频线路的样本混合在一起）。但是，有时您希望能够修改音频信号。用户可能希望它听起来更大声，更安静，更饱满，更具回响感，音高更高或更低，等等。该页面讨论了提供这些信号处理的Java Sound API功能。

有两种方法可以应用信号处理：

- 您可以通过查询[`Control`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/Control.html)对象，然后根据用户需要设置控件controls来使用混频器或其组成部分线路所支持的任何处理 。混音器和线路支持的典型控件包括增益gain，声相pan和混响reverberation控件。
- 如果混音器或其线路未提供所需的处理类型，则程序可以直接对音频字节进行操作，并根据需要进行操作。

此页面将更详细地讨论第一种技术，因为第二种技术没有特定的API。

### 控件介绍

混频器在其某些或全部线路上可以具有各种信号处理控件。例如，用于音频捕获的混频器可能具有带增益控制的输入端口，以及带增益和平移控制的目标数据线。用于音频播放的混音器可能在其源数据线上具有采样率控件。在每种情况下，都可以通过`Line`接口的方法来访问控件。

因为`Mixer`接口继承自`Line`，所以混音器本身可以具有自己的一组控件。这些控件可能用作影响所有混音器的源线或目标线的主控件。例如，混频器可能具有一个主增益控制，该主增益控制的分贝值被叠加到其目标线上的各个增益控制的值。

混音器自己的其他控件可能会影响混音器内部用于处理的特殊线路，无论是源线还是目标线。例如，全局混响控件可能会选择混响的类型，以应用于输入信号的混合，并且这种“湿”（混响）信号在传送到混音器的目标线路之前会混进“干”信号中。

如果混音器或其任何线路具有控件，您可能希望通过程序用户界面中的图形对象来显示控件，以便用户可以根据需要调整音频特性。这些控件本身不是图形的；它们只是允许您检索和更改其设置。您可以决定在程序中使用哪种图形表示形式（滑块，按钮等）。

所有控件都作为抽象类`Control`的具体子类实现。许多典型的音频处理控件可以通过基于某种数据类型（例如boolean，enumerated或float）的`Control`抽象子类来描述。例如，布尔控制代表二进制状态控制，例如静音或混响的开/关控制。另一方面，浮点数控件非常适合表示连续可变的控件，例如声像pan，平衡balance或音量volume。

Java Sound API指定以下`Control`的抽象子类：

- [`BooleanControl`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/BooleanControl.html)——表示二进制状态（true or false）控件。例如，静音（mute），独奏（solo）和开/关（on/off）的理想选择将是`BooleanControls`。
- [`FloatControl`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/FloatControl.html)——提供对某个范围的浮点值的控制的数据模型。例如，音量（volume）和声相（pan）是`FloatControls`类型，可以通过刻度盘或滑条进行操作。
- [`EnumControl`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/EnumControl.html)——提供选取一个对象集合中某个元素。例如，您可以将用户界面中的一组按钮与`EnumControl`关联起来，以选择多个预设混响设置之一。
- [`CompoundControl`](https://docs.oracle.com/javase/8/docs/api/javax/sound/sampled/CompoundControl.html)——提供对相关项目集合的访问权限，每个相关项目本身都是`Control`子类的实例。`CompoundControls`代表多控制模块，例如图形均衡器。（图形均衡器通常由一组滑块描绘，每个滑块影响一个`FloatControl`。）

上述`Control`的每个子类都有适合其基本数据类型的方法。大多数类包括设置和获取控件当前值（get and set），获取控件标签等的方法。

当然，每个类都有其特定的方法以及该类表示的数据模型。例如，`EnumControl`有一种方法可以让您获取其可能值的集合，`FloatControl`允许您获取其最小值和最大值以及控件的精度（增量increment或步长step size）。

`Control`的每个子类都有一个对应的`Control.Type`子类，其中包括标识特定控件的静态实例。

下表显示了每个`Control`子类，其对应的`Control.Type`子类以及指示特定种类的控件的静态实例：



| `Control`         | `Control.Type`         | `Control.Type` **实例**                                      |
| ----------------- | ---------------------- | ------------------------------------------------------------ |
| `BooleanControl`  | `BooleanControl.Type`  | `MUTE`–线路静音状态<br> `APPLY_REVERB`–混响开/关             |
| `CompoundControl` | `CompoundControl.Type` | （没有）                                                     |
| `EnumControl`     | `EnumControl.Type`     | `REVERB` – 访问混响设置（每个都是ReverbType的实例）          |
| `FloatControl`    | `FloatControl.Type`    | `AUX_RETURN` – 一条线路上的辅助返回增益<br/>`AUX_SEND` – 辅助发送增益<br/>`BALANCE` – 左右音量平衡<br/>`MASTER_GAIN` – 一条线路上的总体增益<br/>`PAN` – 声相，左右位置<br/>`REVERB_RETURN` – 线路上的混响后增益<br/>`REVERB_SEND` – 线路上的混响前增益<br/>`SAMPLE_RATE` – 播放采样率<br/>`VOLUME` – 线路上的音量 |

Java Sound API的实现可以在其混音器和线路上提供任何或全部的这些控件类型。它也可以提供Java Sound API中未定义的其他控件类型。可以通过这四个抽象子类中的任何一个的具体子类或不是继承于这四个抽象子类的其他`Control`子类（自定义的）来实现此类控件类型。应用程序可以查询每条线路以找到其支持的控件。

### 获得具有所需控件的线路

在许多情况下，应用程序将仅（simply?）显示该线路所支持的任何控件。如果该线路没有任何控件，那么好吧。但是，如果找到具有某些控件的线路很重要，该怎么办？在这种情况下，您可以使用`Line.Info`获取具有正确特性的线，如前面在 [获取所需类型的线下所述](https://docs.oracle.com/javase/tutorial/sound/accessing.html#113154)。

例如，假设您希望使用一个输入端口，以便用户设置声音输入的音量。以下代码摘录显示了如何查询默认混音器以确定其是否具有所需的端口和控件：

```java
Port lineIn;
FloatControl volCtrl;
try {
  mixer = AudioSystem.getMixer(null);
  lineIn = (Port)mixer.getLine(Port.Info.LINE_IN);
  lineIn.open();
  volCtrl = (FloatControl) lineIn.getControl(

      FloatControl.Type.VOLUME);

  // Assuming getControl call succeeds, 
  // we now have our LINE_IN VOLUME control.
} catch (Exception e) {
  System.out.println("Failed trying to find LINE_IN"
    + " VOLUME control: exception = " + e);
}
if (volCtrl != null)
  // ...
```

### 从线路上获取控件

需要在其用户界面中公开控件的应用程序，可以简单地查询可用的线路和控件，然后为每个感兴趣的线路上的每个控件显示适当的用户界面元素。在这种情况下，该程序的唯一任务是为用户提供控件上的“句柄”。不知道这些控件对音频信号有什么作用。只要程序知道如何将线路的控制映射到用户界面元素，Java Sound API的`Mixer`，`Line`以及`Control`结构通常会解决其余的事。

例如，假设您的程序播放声音。您正在使用`SourceDataLine`，如先前在[获取所需类型的线路](https://docs.oracle.com/javase/tutorial/sound/accessing.html#113154)中所述，您可以通过调用`Line`的以下方法访问该线路的控件：

```java
Control[] getControls()
```

然后，对于此返回数组中的每个控件，您可以使用以下`Control`的以下方法获取控件的类型：

```java
Control.Type getType()
```

知道特定的`Control.Type`实例后，您的程序可以显示相应的用户界面元素。当然，为特定的`Control.Type`选择“相应的用户界面元素” 取决于程序所采用的方法。一方面，您可能使用相同类型的元素来表示同一类的所有`Control.Type`实例。这就需要你来查询`Control.Type`实例使用的*类*，例如`Object.getClass`方法。假设结果与`BooleanControl.Type`相匹配。在这种情况下，您的程序可能会显示一个通用复选框或切换按钮，但是如果其类与`FloatControl.Type`匹配，则您可能会显示一个图形滑块。

另一方面，您的程序可能会区分不同类型的控件（甚至是同一类的控件），并为每个控件使用不同的用户界面元素。这将要求您测试`Control`的`getType`方法返回的*实例*。然后，例如，如果类型与`BooleanControl.Type.APPLY_REVERB`匹配，则您的程序可能会显示一个复选框；如果类型匹配`BooleanControl.Type.MUTE`，则可以显示一个切换按钮。

### 使用控件更改音频信号

既然您知道了如何访问控件并确定控件的类型，本节将介绍如何使用`Controls`来更改音频信号的各个方面。本节并不涵盖所有可用的控件；相反，这里提供了一些示例来向您展示如何入门。这些示例包括：

- 控制线路的静音状态
- 更改线路的音量
- 在各种混响预设中选择

假设您的程序已经访问了所有混合器，它们的线路和这些线路上的控件，并且它具有一个数据结构来管理控件及其相应的用户界面元素之间的逻辑关联。然后，将用户对这些控件的操作转换为相应的`Control`方法就变得相当简单。

以下小节描述了一些必须调用才能影响对特定控件所做的更改的方法。

<以上一句话没看懂，给原文>

The following subsections describe some of the methods that must be invoked to affect the changes to specific controls.

#### 控制线路的静音状态

控制任何线路的静音状态仅需调用`BooleanControl`的以下方法即可：

```java
void setValue(boolean value)
```

（程序大概通过引用其控制管理数据结构，来知道静音是`BooleanControl`的一个实例。）要使通过线路的信号静音，程序要调用上面的方法，将其指定为`true`值。要关闭静音，使信号流过线路，程序将调用参数设置为`false`的方法。

#### 更改线路的音量

假设您的程序将特定的图形滑块与特定线路的音量控件相关联。音量控制（即`FloatControl.Type.VOLUME`）的值使用`FloatControl`的以下方法来设置：

```java
void setValue(float newValue)
```

检测到用户移动了滑块，程序将获取滑块的当前值，并将其作为参数`newValue`传递给上述方法。这将改变流经“拥有”控件的线路的信号的音量。

#### 在各种混响预设中选择

假设我们的程序有一个混音器，其中的一条线路具有`EnumControl.Type.REVERB`类型的控件。调用`EnumControl`方法：

```java
java.lang.Objects[] getValues()
```

通过调用该方法，控件产生一个`ReverbType`对象数组。如果需要，可以使用`ReverbType`的以下方法访问每个对象的特定参数设置：

```java
int getDecayTime() 
int getEarlyReflectionDelay() 
float getEarlyReflectionIntensity() 
int getLateReflectionDelay() 
float getLateReflectionIntensity() 
```

例如，如果一个程序只想要一个听起来像在洞穴中的混响设置，则它可以遍历`ReverbType`对象，直到找到其`getDecayTime`返回值大于2000 的对象为止。有关这些方法的详细说明，包括一些典型的返回值，请参阅的API参考文档`javax.sound.sampled.ReverbType`。

但是，通常，程序会为该`getValues`方法返回的数组中的每个`ReverbType`对象创建一个用户界面元素，例如单选按钮。当用户单击这些单选按钮之一时，程序将调用`EnumControl`的以下方法：

```java
void setValue(java.lang.Object value) 
```

其中`value`设置为与新启用的按钮相对应的`ReverbType`。然后，通过“拥有”该`EnumControl`的线路发送的音频信号，将根据构成控件当前电流的参数设置`ReverbType`（即，该方法`ReverbType`的`value`参数中指定的特定参数）回响`setValue`。

因此，从我们的应用程序的角度来看，使用户能够从一个混响预设（即ReverbType）变换到另一个混响预设，仅是将`getValues`返回的数组的每个元素连接到一个单独的单选按钮的问题。

### 直接处理音频数据

`Control`API允许Java Sound API的一种实现或混音器的第三方提供程序通过控件提供任意种类的信号处理。但是，如果没有混频器提供您所需的那种信号处理呢？这将需要更多的工作，但是您可能可以在程序中实现信号处理。因为Java Sound API允许您以字节数组的形式访问音频数据，所以可以选择任何方式更改这些字节。

如果您要处理传入的声音，则可以从`TargetDataLine`中读取字节，然后对其进行操作。可以产生令人神往的结果的算法简单的（An algorithmically trivial example that can yield sonically intriguing results）示例是，通过以相反的顺序排列其帧来向后播放声音。这个琐碎的示例可能在您的程序中用处不大，但是有许多复杂的数字信号处理（DSP）技术可能更合适。一些示例包括均衡（equalization），动态范围压缩（dynamic-range compression），峰值限制（peak limiting）和时间拉伸或压缩（time stretching or compression），以及特殊效果，例如延迟（delay），合唱（chorus），镶边（flanging），失真（distortion）等。

要播放处理后的声音，可以将经过处理的字节数组放入`SourceDataLine`或`Clip`中。当然，字节数组不必从现有的音频产生。您可以从头开始合成音频，尽管这需要一些声学知识，或者需要使用音频合成功能。对于处理或合成，您可能需要查阅音频DSP教科书中感兴趣的算法，或者将第三方的信号处理功能库导入程序。要播放合成声音，请考虑`javax.sound.midi`包中的`Synthesizer`API 是否满足您的需求。稍后您将`javax.sound.midi`在《[合成声音](https://docs.oracle.com/javase/tutorial/sound/MIDI-synth.html)》下 了解更多信息。