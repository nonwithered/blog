# MIDI二进制文件格式简析

本文主要参考自[Official MIDI Specifications](https://www.midi.org/specifications)

## Chunks

每个MIDI文件由一系列*chunk*组成，每个*chunk*的前四个字节为魔数（*magic number*），是由四个ASCII字符所组成的类型标识。目前标准格式中已定义的*chunk*类型只有*header*和*track*两种，其魔数分别为`"MThd"`和`"MTrk"`，对于类型未被定义的*chunk*则应该被忽略。

每个*chunk*在其四字符类型之后紧跟一个32位无符号整数，意味这一*chunk*后续将要读入的字节个数，每个*chunk*已经读入的这八个字节不包含在内。

通常来说，一个MIDI文件中首先要存在一个*header chunk*，然后一系列*track chunk*紧随其后，格式大致如下：

> MThd \<length of header data\>
>
> \<header data\>
>
> MTrk \<length of track data\>
>
> \<track data\>
>
> MTrk \<length of track data\>
>
> \<track data\>
>
> ...

## Header Chunks

在目前的标准中，*header chunk*长度固定为14，其由五个部分所组成：

> \<Header Chunk\> = \<chunk type\> \<length\> \<format\> \<ntrks\> \<division\>

其中**type**固定为`"MThd"`，**format**、**ntrks**、**division**均为16位，因此**length**固定为`0x00000006`。

**format**指定了整个文件的组织结构，在当前的标准下，仅支持`0x0000`、`0x0001`、`0x0002`这三种可能的值。

> **0** the file contains a single multi-channel track
>
> **1** the file contains one or more simultaneous tracks (or MIDI outputs) of a
sequence
>
> **2** the file contains one or more sequentially independent single-track patterns

**ntrks**表示整个文件中*track chunk*的总个数，对于**format**为零的文件来说**ntrks**的值总是`0x0001`。

最后的**division**表示了*delta-times*的含义，有*metrical time*和*time-code-based time*这两种形式，这取决于其最高位是否为零。

假如**division**最高位为零，那么就属于*metrical time*形式，这个16位无符号整数的意义为每个四分音符的ticks个数。否则属于*time-code-based time*形式，其低8位表示每帧的ticks个数，高8位为一个8位负数补码，有`-24`、`-25`、`-29`、`-30`这四种可能的值，其意义与*SMPTE*有关，以下内容摘抄自维基百科：

> Sub-second timecode time values are expressed in terms of frames. Common supported frame rates include:
>
> 24 frame/sec (film, ATSC, 2K, 4K, 6K)
>
> 25 frame/sec (PAL (Europe, Uruguay, Argentina, Australia), SECAM, DVB, ATSC)
>
> 29.97 (30 ÷ 1.001) frame/sec (NTSC American System (US, Canada, Mexico, Colombia, etc.), ATSC, PAL-M (Brazil))
>
> 30 frame/sec (ATSC)

在这里可以发现，凭借**division**是完全不足以描述每个时间间隔的实际长度的。在规范中对此有所说明，在默认情况下，乐曲的节奏为4/4拍，速度为每分钟120拍。这类元数据应该在MIDI文件中被指定，对于**format**为`0x0000`的文件应处于其`track chunk`的开始，对于**format**为`0x0001`的文件应包含在第一个`track chunk`之内，对于**format**为`0x0002`的文件应包含在每个`track chunk`之中。

值得一提的是，未来的标准有可能会定义更多种类的**format**和*chunk*，甚至有可能为*header chunk*添加更多的参数从而使其**length**不再为`0x00000006`，因此对于程序的实现者来说，遵守标准是十分必要的。

## Track Chunks

无论**format**的取值是什么，*Track Chunks*的结构都是一致的：

> \<Track Chunk\> = \<chunk type\> \<length\> \<MTrk event\>+

在**length**后是一串连续的*MTrk event*，每个*MTrk event*由两部分组成：

> \<MTrk event\> = \<delta-time\> \<event\>

这里的**delta-time**是一种*variable-length quantity*，其每个字节仅有7个有效位，最高位若非零则说明后面还有下一字节，最多可占据四个字节，因此其最大值为`0x0fffffff`。以下为参考示例：

|Number(hex)|Representation(hex)|
|:-:|:-:|
|00000000|00|
|00000040|40|
|0000007F|7F|
|00000080|81 00|
|00002000|C0 00|
|00003FFF|FF 7F|
|00004000|81 80 00|
|00100000|C0 80 00|
|001FFFFF|FF FF 7F|
|00200000|81 80 80 00|
|08000000|C0 80 80 00|
|0FFFFFFF|FF FF FF 7F|

**delta-time**后紧跟的*event*有三种不同的类型：

> \<event\> = \<MIDI event\> | \<sysex event\> | \<meta-event\>

*MIDI event*的第一个字节通常表示`running status`，其最高位必须非零，对于*sysex event*和*meta-event*来说，这个字节分别为`0xf0`或`0xff`。

## MIDI event

在每个*track chunk*中出现的第一个*MIDI event*必须指定*running status*，而对于后续的*MIDI event*，假如其紧跟在一个*MIDI event*后面，并且与前一个*MIDI event*有着同样的*running status*，那么这一个*MIDI event*的*running status*可以被省略。

每个*MIDI event*根据其*running status*高4位的不同而存在一个或两个单字节参数，这些参数的最高位均为零，*running status*的低4位表示*channel*的编号，每个*MIDI event*只会在对应的*channel*上造成影响。举几个例子，以16进制表示：高4位为`C`意为*Program Change*，其跟随一个单字节参数，表示修改乐器音色；高4位为`9`意为*Note On*，高4位为`8`意为*Note Off*，这两者都跟随两个单字节参数，分别表示音高与力度。

中央C在*MIDI event*中的值为`0x3c`，每升高半音则加一，每降低半音则减一，通过这种规律可以推断出标准音为`0x45`。

*channel*最多有16个，每个*channel*之间是独立的，也就是说不同的*channel*可以通过*Program Change*同时使用不同的音色，而又由于*Note On*和*Note Off*也是独立的，因此同一个*channel*中也可以同时播放不止一个音符。不过打击乐通常位于第10号*channel*。

音色对照表见文章末尾。

## sysex event

这是一种结构稍显复杂的事件信息，它可以包含一连串的`sysex event`，其基本结构如下：

> F0 \<length\> \<bytes to be transmitted after F0\>

这里**length**为**bytes**的长度，如果**bytes**不以`0xf7`结尾，那么其后就要跟随一个变长的**delta-time**，然后在跟随下一个*sysex event*，就像这样：

> F7 \<length\> \<all bytes to be transmitted\>

除第一个*sysex event*之外，后续跟随的*sysex event*首字节应为`0xf7`，且最后一个*sysex event*的最后一字节应为`0xf7`。以下是一串合法的*sysex event*示例：

> F0 03 43 12 00
>
> 81 48
>
> F7 06 43 12 00 43 12 00
>
> 64
>
> F7 04 43 12 00 F7

以上部分的`0x8148`和`0x64`分别表示200-tick *delta-time*和100-tick *delta-time*。

## meta-event

这一部分的格式是比较固定的：

> FF \<type\> \<length\> \<bytes\>

所有*meta-event*以`0xff`起始，然后紧跟一个字节的**type**，再然后跟随一个变长的**length**，**length**的值即为**bytes**的长度。

在*Standard MIDI Files 1.0*中已经预定义了一部分*meta-event*，其中*Copyright Notice*应作为第一个*track*的第一个*event*，*Sequence Number*和*Sequence/Track Name*若存在则必须在任何*delta-times*非零的*event*前出现，*End of Track*必须作为每个*track*的最后一个*event*。

|FF \<type\> \<length\> \<bytes\>|meta-event|
|:-|:-|
|FF 00 02 ssss|Sequence Number|
|FF 01 len text|Text Event|
|FF 02 len text|Copyright Notice|
|FF 03 len text|Sequence/Track Name|
|FF 04 len text|Instrument Name|
|FF 05 len text|Lyric|
|FF 06 len text|Marker|
|FF 07 len text|Cue Point|
|FF 20 01 cc|MIDI Channel Prefix|
|FF 2F 00|End of Track|
|FF 51 03 tttttt|Set Tempo|
|FF 54 05 hr mn se fr ff|SMPTE Offset|
|FF 58 04 nn dd cc bb|Time Signature|
|FF 59 02 sf mi|Key Signature|
|FF 7F len data|Sequencer-Specific Meta-Event|

*Set Tempo*参数的意义为每个四分音符的微秒数。

*Time Signature*参数的意义分别为节奏的分子、以二为底取得节奏分母的对数、节拍器每响一次的*MIDI clocks*个数、每个四分音符所包含的三十二分音符个数（最后这个我也不理解有什么意义），比如以下示例：

>Therefore, the complete event for 6/8 time, where the metronome clicks every three eighth-notes, but there are 24 clocks per quarter-note, 72 to the bar, would be (in hex):
>
> FF 58 04 06 03 24 08
>
>That is, 6/8 time (8 is 2 to the 3rd power, so this is 06 03), 36 MIDI clocks per dotted-quarter (24 hex!), and eight notated 32nd-notes per MIDI quarter note.

*Key Signature*的第一个参数表示乐曲经过了移调所添加的升降号个数，第二个参数取`0x00`或`0x01`意为大调或小调。

## 附

以下音色对照表引用自*General MIDI System Level 1*

### General MIDI Sound Set Groupings(all channels except 10)

|Prog #|Instrument Group|Prog #|Instrument Group|
|:-|:-|:-|:-|
|1-8|Piano|65-72|Reed|
|9-16|Chromatic Percussion|73-80|Pipe|
|17-24|Organ|81-88|Synth Lead|
|25-32|Guitar|89-96|Synth Pad|
|33-40|Bass|97-104|Synth Effects|
|41-48|Strings|105-112|Ethnic|
|49-56|Ensemble|113-120|Percussive|
|57-64|Brass|121-128|Sound Effects|

### General MIDI Percussion Map(Channel 10)

|MIDI Key|Drum Sound|MIDI Key|Drum Sound|MIDI Key|Drum Sound|
|:-:|:-|:-:|:-|:-:|:-|
|35|Acoustic Bass Drum|51|Ride Cymbal 1|67|High Agogo|
|36|Bass Drum 1|52|Chinese Cymbal|68|Low Agogo|
|37|Side Stick|53|Ride Bell|69|Cabasa|
|38|Acoustic Snare|54|Tambourine|70|Maracas|
|39|Hand Clap|55|Splash Cymbal|71|Short Whistle|
|40|Electric Snare|56|Cowbell|72|Long Whistle|
|41|Low Floor Tom|57|Crash Cymbal 2|73|Short Guiro|
|42|Closed Hi Hat|58|Vibraslap|74|Long Guiro|
|43|High Floor Tom|59|Ride Cymbal 2|75|Claves|
|44|Pedal Hi-Hat|60|Hi Bongo|76|Hi Wood Block|
|45|Low Tom|61|Low Bongo|77|Low Wood Block|
|46|Open Hi-Hat|62|Mute Hi Conga|78|Mute Cuica|
|47|Low-Mid Tom|63|Open Hi Conga|79|Open Cuica|
|48|Hi Mid Tom|64|Low Conga|80|Mute Triangle|
|49|Crash Cymbal 1|65|High Timbale|81|Open Triangle|
|50|High Tom|66|Low Timbale|||
