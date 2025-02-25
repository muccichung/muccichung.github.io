+++
title = "联想M920X解锁BIOS高级菜单教程"
date = "2025-02-25T14:50:44-06:00"
+++

去年闲暇的时候，把10年前的大主机换成了小巧的联想M920X，搭配CC150、Sparkle Arc A380半高刀卡。在官方说明中，这张A380仅兼容intel 10代以上平台，但我们可以关闭CSM，在老平台将其点亮，而要发挥其全部性能，则必须打开Resizable Bar和Above 4G Decoding。联想官方BIOS中隐藏了以上两个高级选项，需要解锁，另外，本机已提前解锁电源识别和PCIE功耗。

使用到的工具：CH341A编程器，以及WinHex、UEFITool等铺助修改软件（后面提供下载地址）。

**注意：刷BIOS有风险，存在丢失保修和设备损坏之风险！**

#### 备份

使用CH341A编程器分别读取两个独立的BIOS芯片，保存为16mb.bin、8mb.bin并备份；

#### 合并

打开WinHex，通过“工具>文件工具>文件合并”合并上面保存的两个文件，命名为main.bin；

<!--excerpt-->

#### 修改

使用UEFITool Ne打开main.bin，Ctrl+F切换到Text，查找“setup mode”，在搜索结果中双击定位，找到名为Setup的DXE driver，记录下GUID，如本例：899407D7-99FE-43D8-9A21-79EC328CAC21，右键PE32 image section，选择Extract as is并保存；

<img alt="双击定位，记住右边的GUID。" src="/posts/2025-05-25-lenovo-m920x-unlock-bios/setupmode.png" />

Windows下打开CMD，输入D:（ifrextractor.exe和以上生成的.sct文件所在盘）,输入CD XXX（ ifrextractor.exe所在文件夹地址），再输入`ifrextractor "Section_PE32_image_Setup_Setup.sct" verbose`， 会自动生成一个txt文件 ；

<img alt="先进所在盘，再进所在文件夹。" src="/posts/2025-05-25-lenovo-m920x-unlock-bios/cmd.png" />

在UEFITool Ne中继续查找“AMITSE”，上级为AMITSE即正解，右键PE32 image section，选择Extract as is并保存；

再往下滚动，找到setupdata，往下展开2次，右键第二个setupdata，选择Extract body并保存；

获取以上四个文件后，打开在线工具[UEFI Editor](https://boringboredom.github.io/UEFI-Editor/)，逐一对应上传，打开后在页面左下角通过Search功能，搜索对应BIOS选项；

<img alt="要对应上传，有提示。" src="/posts/2025-05-25-lenovo-m920x-unlock-bios/uefieditor.png" />

取消Suppress if列的红框表示取消隐藏该项内容，需要注意的是，取消某个隐藏项，需同时取消其上级项的隐藏；如Above 4G，其菜单逐层为Chipset>System Agent(SA) Configuration>Above 4GB MMIO BIOSassignment，Chipset的红框也要取消，按需取消隐藏后，点击UEFI files下载保存；

<img alt="above 4g在chipset底下的底下。" src="/posts/2025-05-25-lenovo-m920x-unlock-bios/chipset.png" />

使用UEFITool（换软件了）打开main.bin，Ctrl+F搜索899407D7-99FE-43D8-9A21-79EC328CAC21（同上），右键PE32 image section，选择Replace as is，选择在UEFI Editor中保存的sct文件替换；

<img alt="确认replace成功。" src="/posts/2025-05-25-lenovo-m920x-unlock-bios/replace.png" />

#### 添加Resizable Bar

ReBarUEFI项目：[https://github.com/xCuri0/ReBarUEFI](https://github.com/xCuri0/ReBarUEFI)

添加FFS模块：[https://github.com/xCuri0/ReBarUEFI/wiki/Adding-FFS-module](https://github.com/xCuri0/ReBarUEFI/wiki/Adding-FFS-module)

继续使用UEFITool打开main.bin，Ctrl+F切换到GUID项，搜索“A0327FE0-1FDA-4E5B-905D-B510C45A61D0”，在搜索结果中双击定位，右键选择Insert after，导入ReBarDxe.ffs，确认成功，保存为new_main.bin；

#### 分割

打开WinHex，将new_main.bin，通过“工具>文件工具>文件分割”分割成一个16MB、一个8MB的文件，分别命名new_16mb.bin和new_8mb.bin；

#### 刷入

将新分割的两个文件刷入对应的芯片中；

在BIOS中开启Above 4G Decoding、关闭CSM，进入桌面，运行ReBarState.exe，选择32，重启电脑即可。

<img alt="状态都正常，然后跑个分试试。" src="/posts/2025-05-25-lenovo-m920x-unlock-bios/rebar.png" />

以上使用到工具的地址：

UEFITool：[https://github.com/LongSoft/UEFITool](https://github.com/LongSoft/UEFITool)

WinHex：[http://www.winhex.com/winhex/index-m.html](http://www.winhex.com/winhex/index-m.html)

IFRExtractor：[https://github.com/LongSoft/IFRExtractor-RS](https://github.com/LongSoft/IFRExtractor-RS)

#### 后续

使用他人备份的BIOS刷机，会出现各种奇怪问题，如开机慢，这时我们需要替换一个纯净的ME（状态为configured）；

需要开启OverClocking Performance Menu，请修改M1UKT67A或更早版本BIOS [下载地址](https://newsupport.lenovo.com.cn/driveDownloads_detail.html?driveId=126093)；通过调节AC Loadline和CPU Core Vlotage offset，可以有效降低功耗和温度。

<img alt="温度下降明显，可上更高性能的CPU。" src="/posts/2025-05-25-lenovo-m920x-unlock-bios/acloadline.png" />
