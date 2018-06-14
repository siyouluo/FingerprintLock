# Fingerprint_Lock
A design of fingerprint lock with 51-MCU and AS608. It can be equipped to mostly doors without any confliction.
这是一个基于51单片机（STC89C52）和指纹识别模块（AS608）的指纹锁全套解决方案
文件（夹）说明：
1、AS608_datasheet：内含两个PDF文件，介绍AS608模块相关软、硬件信息，其中重要的是AS608模块与单片机的通讯方式；
2、Board_Layout:内含一个*.rst文件，是电路布局图，需要用lochmaster软件打开。读者可按照该电路图焊接电路板。如果没有安装lochmaster软件，可直接查看另外两个PDF文件；
3、src:这是项目C语言源文件以及源文件备份文件（.txt）。读者可以直接使用其中的*.hex文件或者使用keil编译环境自行编译；
4、schematic：这是用Multisim绘制的原理图（由于Multisim中缺少指纹模块元件，暂不可仿真分析，此处仅供指导电路板的焊接）。如果没有安装Multisim，可直接查看另一个PDF文件；
5、manifest：这是制作整个电路系统所需的元件信息。

补充说明：
1、由于LCD1602液晶耗电较为严重，6800mAh的电池电量仅能使用不到一天，实际使用时，必须保证电池始终处于充电状态。使用电池的目的是为了防止停电后指纹锁无法工作（应急）。若要降低功耗，可考虑将LCD1602背光灯的供电断开，或者使用一个三极管和一个IO口进行控制；
2、普通电机所能输出端的力矩有限，本系统中采用了减速电机，增大输出力矩，并在机械锁上连接省力杠杆，增加驱动力。两者之间使用绳子软连接。实际应用时可根据机械锁开锁方式及难易程度进行相应调整
