# S2600IP_bios_mod
## 事由
近日深水宝淘了一片intel原装S2600IP4，双路E5，16内存槽，支持双槽4显卡，平民deeplearning神器呢。这板比一般的SSI EEB规格还要大一圈，机箱的选择就很有限，正规支持这种尺寸的貌似只有supermicro的塔式机箱(卧式放机柜的暂时放弃)，最后选了个TT P5加一顿diy，勉强装上。然后装上两颗es版的2680V2，死活点不亮，status led一直显示黄色。装个正式版的cpu就能点亮。怀疑是bios的问题，刷了几个其他版本的bios，也是黄。于是疑点落在了cpu microcode上，就开始了bios mod的旅程。


## 工具
#### bios mod涉及到的工具有
- [MCExtractor](https://github.com/platomav/MCExtractor), 从bios提取cpu microcode
- [MMTool](https://www.baidu.com/s?wd=mmtool), 一般的bios文件都可以用mmtool添加cpu microcode，但intel这款主板的bios，mmtool搞不掂
- [WinHex](https://www.baidu.com/s?wd=mmtool), 十六进制编辑工具
- [UEFITool](https://github.com/LongSoft/UEFITool), UEFI bios解包导入导出工具。

## 过程
#### 查找缺失的cpu microcode
找到同样是C602芯片组的bios，这里用到的有两款，一款是华擎的EP2C602，一款是supermicro的X9DRI。先查看S2600IP4的bios有些什么cpu microcode，用MCExtractor可以提取bios的cpu microcode，发现只有4个，分别是206D5、206D6、206D7和306E4，分别对应是正式版E5和正式版E5 V2。再看EP2C602的bios，比S2600IP4的bios多了306E0、306E2和306E3，而X9DRI的bios还多了一个306E7。到此，我决定先把306E0、306E2和306E3加入到S2600IP4的bios里面，试试效果。
#### 查找cpu microcode在bios内的位置
这里再次强调一下，S2600IP4的bios无法通过MMTool导入新的cpu microcode，这是其中一个坑爹的地方，所以以下描述的方法，需要一点点动手能力。首先，用UEFITool打开S2600IP4的bios，虽然FIT头部已经显示出cpu microcode的条目及其偏移，但无法导出和编辑。此时，我们先对bios生成report文件，打开report文件，查找microcode关键字，得到两处匹配，注意力集中在这个匹配，记下GUID
```
 Volume          | FFSv2                 | 001503F0 | 00080000 | C5364C33 | -- AD3FFFFF-D28B-44C4-9F13-9EA98A97F9F0
 File            | Raw                   | 00150438 | 0007FC18 | 30A64852 | --- Microcode
 Free space      |                       | 001D0050 | 000003A0 | F13B8DCD | --- Volume free space
```
用UEFITool_NE_A51打开bios文件，查找GUID，定位到这个文件段，如果把这个文件段导出，可以看到里面的内容就是cpu microcode。我们的目标，就是把新的内容导入到这个文件段里面，并修改FIT，添加新的cpu microcode条目。
#### 向bios导入新的cpu microcode
坑爹的地方又来了，UEFITool无法把上面提到的文件段导出——修改——导入。所以，我们选择直接修改bios文件，包括导入新的cpu microcode和新的FIT条目。在阅读MCExtractor源码及intel的开发手册Intel 64 and IA-32 Architectures Software Developer's Manual Vol 3A, Ch 9.11.1后，发现cpu microcode可以通过一个模式串定位，甚至，我们直接用前面导出的文件段头部作为目标串，在bios文件里查找，就能定位cpu microcode的位置。定位位置后，我们在原有的4段cpu microcode后面、原有的0xFF的区域，用新的(来自EP2C602的bios，用MCExtractor导出的)cpu mocrocode覆盖，注意，不是插入，不要改变文件大小。
#### 为新导入的cpu microcode添加FIT条目
导入完新的cpu microcode，还要为这些内容添加FIT条目，对bios文件进行文本搜索"_FIT_"，得到如下内容
```
Offset      0  1  2  3  4  5  6  7   8  9  A  B  C  D  E  F
000493F0   5F 46 49 54 5F 20 20 20  20 00 00 00 00 01 00 00   _FIT_           
00049400   60 00 B1 FF 00 00 00 00  00 00 00 00 00 01 01 00   ` ?            
00049410   60 34 B1 FF 00 00 00 00  00 00 00 00 00 01 01 00   `4?            
00049420   60 7C B1 FF 00 00 00 00  00 00 00 00 00 01 01 00   `|?            
00049430   60 BC B1 FF 00 00 00 00  00 00 00 00 00 01 01 00   `急            
00049440   00 00 FF FF 00 00 00 00  00 00 00 00 00 01 02 00                 
00049450   00 C0 FF FF 00 00 00 00  00 04 00 00 00 01 07 00    ?            
00049460   70 00 71 00 01 01 7F 00  00 00 00 00 00 00 08 00   p q             
00049470   70 00 71 00 01 00 7F 00  00 00 00 00 00 00 0A 00   p q             
```
观察从第二行开始，到第五行，就是原有的4个cpu microcode偏移(这个偏移可以从UEFITool里面FIT页面得到)，现在要添加新的偏移。同样，把第六行开始的内容，往下复制，腾出三行空间。计算新cpu microcode条目的偏移，并写入这三行，得到以下结果
```
Offset      0  1  2  3  4  5  6  7   8  9  A  B  C  D  E  F
000493F0   5F 46 49 54 5F 20 20 20  20 00 00 00 00 01 00 00   _FIT_           
00049400   60 00 B1 FF 00 00 00 00  00 00 00 00 00 01 01 00   ` ?            
00049410   60 34 B1 FF 00 00 00 00  00 00 00 00 00 01 01 00   `4?            
00049420   60 7C B1 FF 00 00 00 00  00 00 00 00 00 01 01 00   `|?            
00049430   60 BC B1 FF 00 00 00 00  00 00 00 00 00 01 01 00   `急            
00049440   60 22 B2 FF 00 00 00 00  00 00 00 00 00 01 01 00   `"?            
00049450   60 2C B2 FF 00 00 00 00  00 00 00 00 00 01 01 00   `,?            
00049460   60 5C B2 FF 00 00 00 00  00 00 00 00 00 01 01 00   `\?            
00049470   00 00 FF FF 00 00 00 00  00 00 00 00 00 01 02 00                 
00049480   00 C0 FF FF 00 00 00 00  00 04 00 00 00 01 07 00    ?            
00049490   70 00 71 00 01 01 7F 00  00 00 00 00 00 00 08 00   p q             
000494A0   70 00 71 00 01 00 7F 00  00 00 00 00 00 00 0A 00   p q             
```
保存文件，刷入主板，开机，点亮。发现手上这两颗es版的cpu，就是306e2那个cpu microcode，之前点不亮，是因为bios里面没有这个cpu microcode。
