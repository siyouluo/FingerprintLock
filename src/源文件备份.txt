#include<reg52.h>
#include<intrins.h>
#define MAIN_Fosc 11059200//宏定义主时钟频率
#define LINE1 	0x80			//1602屏地址定义 第一行地址
#define LINE2 	0xc0			//1602屏地址定义 第二行地址
#define DataPort P0	  //LCD1602操作位定义

typedef unsigned char INT8U;
typedef unsigned char uchar;
typedef unsigned int INT16U;
typedef unsigned int uint;

sbit EN = P3^4;     //读写数据使能   0：停止 1：启动
sbit RS = P3^5;     //寄存器选择 0:指令寄存器 1:数据寄存器
sbit RW = P3^6;     //读写控制 0：写  1：读
sbit KEY_DOWN=P2^4;
sbit KEY_OK=P2^2;
sbit KEY_CANCEL=P2^0;
sbit beep=P2^6; 

uchar flag=0;
extern char local_date=0;  //全局变量，当前箭头位置
extern unsigned int finger_id = 0;

//uart 函数
void Uart_Init(void)
{
    SCON=0x50;   //UART方式1:8位UART;   REN=1:允许接收 
    PCON=0x00;   //SMOD=0:波特率不加倍 
    TMOD=0x20;   //T1方式2,用于UART波特率 
    TH1=0xFD; 
    TL1=0xFD;   //UART波特率设置:FDFD，9600;FFFF,57600
    TR1=1;	 //允许T1计数 
    EA=1;	 //开总中断
}

void Uart_Send_Byte(unsigned char c)//UART Send a byte
{
	SBUF = c;
	while(!TI);		//发送完为1 
	TI = 0;
}

unsigned char Uart_Receive_Byte()//UART Receive a byteg
{	
	unsigned char dat;
	while(!RI);	 //接收完为1 
	RI = 0;
	dat = SBUF;
	return (dat);
}
//延时函数
void Delay_us(int i)
{
	while(--i);
}

void Delay_ms(INT16U ms)
{
     INT16U i;
	 do{
	      i = MAIN_Fosc / 96000; 
		  while(--i);   //96T per loop
     }while(--ms);
}

//蜂鸣器函数
void Beep_Times(unsigned char times)
{
	unsigned char i=0;
	for(i=0;i<times;i++)
	{
		 beep=0;
		 Delay_ms(200);
		 beep=1;
		 Delay_ms(200);
	}
}

//按键操作函数
void Key_Init(void)
{
    //定义按键输入端口
	KEY_DOWN=1;		// 下一项
	KEY_OK=1;		// 确认
	KEY_CANCEL=1;	// 取消
}

// 1602液晶函数
//写指令
void LCD1602_WriteCMD(unsigned char cmd)  
{
  	EN=0;
  	RS=0;
  	RW=0;
  	Delay_us(10);
  	EN=1; 
  	Delay_us(10);
  	DataPort=cmd;
  	Delay_us(10);
  	EN=0;
}
//写数据
void LCD1602_WriteDAT(unsigned char dat)
{
  	EN=0;
  	RS=1;
  	RW=0;
  	Delay_us(10);
  	EN=1; 
  	Delay_us(10);
  	DataPort=dat;
  	Delay_us(10);
  	EN=0;
}
//液晶繁忙检测
void LCD1602_CheckBusy(void)
{
	
	uchar busy;
	DataPort=0xff; 
	RS=0;
	RW=1;
	do
	{
		EN=1;
		busy=DataPort;
		EN=0;
	}while(busy&0x80);
	
}
//液晶初始化函数
void LCD1602_Init(void)  
{
	Delay_ms(15);      		//上电延时15ms
  	LCD1602_WriteCMD(0x38); //写显示指令(不检测忙信号)
  	Delay_ms(5);
  	LCD1602_CheckBusy();
  	LCD1602_WriteCMD(0x38); //写显示指令
  	LCD1602_CheckBusy();
  	LCD1602_WriteCMD(0x01); //清屏
  	LCD1602_CheckBusy();
  	LCD1602_WriteCMD(0x06); //显示光标移动设置
  	LCD1602_CheckBusy();
  	LCD1602_WriteCMD(0x0c); //显示开及光标设置  
}

//液晶显示函数		入口参数：addr起始地址，pointer指针地址，index下标，num个数
void LCD1602_Display(unsigned char addr,unsigned char* pointer,unsigned char index,unsigned char num)
{
  	unsigned char i;
  	LCD1602_CheckBusy();	//判断忙信号
  	LCD1602_WriteCMD(addr);	//写入地址
  	for(i=0;i<num;i++)		//写入数据
  	{
     	LCD1602_CheckBusy();			   //判断忙信号
     	LCD1602_WriteDAT(pointer[index+i]);//写入数据     
  	}
}

//AS608指纹模块
volatile unsigned char AS608_RECEICE_BUFFER[32]; //volatile:系统总是重新从它所在的内存读取数据，即使它前面的指令刚刚从该处读取过数据

//FINGERPRINT通信协议定义
code unsigned char AS608_Get_Device[10] ={0x01,0x00,0x07,0x13,0x00,0x00,0x00,0x00,0x00,0x1b};//口令验证
code unsigned char AS608_Pack_Head[6] = {0xEF,0x01,0xFF,0xFF,0xFF,0xFF};  //协议包头
code unsigned char AS608_Get_Img[6] = {0x01,0x00,0x03,0x01,0x00,0x05};    //获得指纹图像
code unsigned char AS608_Get_Templete_Count[6] ={0x01,0x00,0x03,0x1D,0x00,0x21 }; //获得模版总数
code unsigned char AS608_Search[11]={0x01,0x00,0x08,0x04,0x01,0x00,0x00,0x03,0xE7,0x00,0xF8}; //搜索指纹搜索范围0 - 999,使用BUFFER1中的特征码搜索
code unsigned char AS608_Search_0_9[11]={0x01,0x00,0x08,0x04,0x01,0x00,0x00,0x00,0x13,0x00,0x21}; //搜索0-9号指纹
code unsigned char AS608_Img_To_Buffer1[7]={0x01,0x00,0x04,0x02,0x01,0x00,0x08}; //将图像放入到BUFFER1
code unsigned char AS608_Img_To_Buffer2[7]={0x01,0x00,0x04,0x02,0x02,0x00,0x09}; //将图像放入到BUFFER2
code unsigned char AS608_Reg_Model[6]={0x01,0x00,0x03,0x05,0x00,0x09}; //将BUFFER1跟BUFFER2合成特征模版
code unsigned char AS608_Delete_All_Model[6]={0x01,0x00,0x03,0x0d,0x00,0x11};//删除指纹模块里所有的模版
volatile unsigned char  AS608_Save_Finger[9]={0x01,0x00,0x06,0x06,0x01,0x00,0x0B,0x00,0x19};//将BUFFER1中的特征码存放到指定的位置
//code unsigned char AS608_num_of_finger_in_lib1[7]={0x01,0x00,0x04,0x1F,0x00,0x00,0x24};//查看指纹库的命令
//code unsigned char AS608_num_of_finger_in_lib2[7]={0x01,0x00,0x04,0x1F,0x01,0x00,0x25};
//code unsigned char AS608_num_of_finger_in_lib3[7]={0x01,0x00,0x04,0x1F,0x02,0x00,0x26};
//code unsigned char AS608_num_of_finger_in_lib4[7]={0x01,0x00,0x04,0x1F,0x03,0x00,0x27};

 //发送包头
void AS608_Cmd_Send_Pack_Head(void)
{
	int i;	
	for(i=0;i<6;i++) //包头
	{
		Uart_Send_Byte(AS608_Pack_Head[i]);   
	}		
}

//发送指令
void AS608_Cmd_Check(void)
{
	int i=0;
	AS608_Cmd_Send_Pack_Head(); //发送通信协议包头
	for(i=0;i<10;i++)
	{		
		Uart_Send_Byte(AS608_Get_Device[i]);
	}
}

//接收反馈数据缓冲
void AS608_Receive_Data(unsigned char ucLength)
{
	unsigned char i;				 
	for (i=0;i<ucLength;i++)
		AS608_RECEICE_BUFFER[i] = Uart_Receive_Byte();
}

//FINGERPRINT_获得指纹图像命令
void AS608_Cmd_Get_Img(void)
{
    unsigned char i;
    AS608_Cmd_Send_Pack_Head(); //发送通信协议包头
    for(i=0;i<6;i++) //发送命令 0x1d
	{
       Uart_Send_Byte(AS608_Get_Img[i]);
	}
}

//将图像转换成特征码存放在Buffer1中
void FINGERPRINT_Cmd_Img_To_Buffer1(void)
{
 	unsigned char i;
	AS608_Cmd_Send_Pack_Head(); //发送通信协议包头      
   	for(i=0;i<7;i++)   //发送命令 将图像转换成 特征码 存放在 CHAR_buffer1
	{
		Uart_Send_Byte(AS608_Img_To_Buffer1[i]);
	}
}
//将图像转换成特征码存放在Buffer2中
void FINGERPRINT_Cmd_Img_To_Buffer2(void)
{
	unsigned char i;
	AS608_Cmd_Send_Pack_Head(); //发送通信协议包头
	for(i=0;i<7;i++)   //发送命令 将图像转换成 特征码 存放在 CHAR_buffer1
	{
		Uart_Send_Byte(AS608_Img_To_Buffer2[i]);
	}
}

//搜索全部用户999枚
void AS608_Cmd_Search_Finger(void)
{
	unsigned char i;	   	    
	AS608_Cmd_Send_Pack_Head(); //发送通信协议包头
	for(i=0;i<11;i++)
	{
		Uart_Send_Byte(AS608_Search[i]);   
	}
}

//转换成特征码
void AS608_Cmd_Reg_Model(void)
{
	unsigned char i;	   		    
	AS608_Cmd_Send_Pack_Head(); //发送通信协议包头
	for(i=0;i<6;i++)
	{
		Uart_Send_Byte(AS608_Reg_Model[i]);   
	}
}

//删除指纹模块里的所有指纹模版
void FINGERPRINT_Cmd_Delete_All_Model(void)
{
	unsigned char i;    
    AS608_Cmd_Send_Pack_Head(); //发送通信协议包头   
    for(i=0;i<6;i++) //命令删除指纹模版
	{
      	Uart_Send_Byte(AS608_Delete_All_Model[i]);   
	}	
}

//保存指纹
void AS608_Cmd_Save_Finger( unsigned int storeID )
{
	unsigned long temp = 0;
	unsigned char i;
	AS608_Save_Finger[5] =(storeID&0xFF00)>>8;
	AS608_Save_Finger[6] = (storeID&0x00FF);
	for(i=0;i<7;i++)   //计算校验和
		temp = temp + AS608_Save_Finger[i]; 
	AS608_Save_Finger[7]=(temp & 0x00FF00) >> 8; //存放校验数据
	AS608_Save_Finger[8]= temp & 0x0000FF;		   
	AS608_Cmd_Send_Pack_Head(); //发送通信协议包头	
	for(i=0;i<9;i++)  
		Uart_Send_Byte(AS608_Save_Finger[i]);      //发送命令 将图像转换成 特征码 存放在 CHAR_buffer1
}

//查看当前指纹库中指纹模板数
int AS608_number_of_fingers()
{
 	int num=1;//默认模板库中有一个模板
	uchar i=0;
	AS608_Cmd_Send_Pack_Head(); //发送通信协议包头
	for(i=0;i<6;i++)
	{
	  	Uart_Send_Byte(AS608_Get_Templete_Count[i]);
	}
	AS608_RECEICE_BUFFER[9]=1;//方便后续判断是否接收成功
	AS608_Receive_Data(14);//接收数据
	if(AS608_RECEICE_BUFFER[9]==0) //接收成功
	{
	 	num=AS608_RECEICE_BUFFER[10]*256+AS608_RECEICE_BUFFER[11];//拼接模板总个数			
	}
	return num;
}
//另一种方法查看指纹库中是否有模板 //本来应该查看所有1000个模板位置是否存在模板，但一般只用得到前256个，故从简
/*
uchar AS608_notEmpty()
{
 	uchar exist=0;
	char i=0;
	AS608_Cmd_Send_Pack_Head(); //发送通信协议包头
	for(i=0;i<7;i++)
	{
		  Uart_Send_Byte(AS608_num_of_finger_in_lib1[i]);
	}
	AS608_Receive_Data(10);//接收前10byte数据,除第10字节的确认码外，其余全部丢弃
	if(AS608_RECEICE_BUFFER[9]==0) //接收成功
	{
	AS608_Receive_Data(32);//接收后续32byte数据，此即0~255个模板为是否存在指纹模板的数据
	for(i=0;i<32;i++)//查看这32byte数据，任何一个位置存在模板则返回值为真，否则为假
	{
	 	if(AS608_RECEICE_BUFFER[i])
			exist=1;
	}
	return exist;
	}
}
*/
//添加指纹
void AS608_Add_Fingerprint()
{
	unsigned char id_show[]={0,0,0};
	LCD1602_WriteCMD(0x01); //清屏  
	while(1)
	{
		LCD1602_Display(0x80,"   Add  finger  ",0,16);
		LCD1602_Display(0xc0,"    ID is       ",0,16);
		//按返回键直接回到主菜单
		if(KEY_CANCEL == 0) 
		{
		 	Delay_ms(5);
		 	if(KEY_CANCEL == 0)
		 	{
		 		while(KEY_CANCEL==0);
		 		break;
		 	}	 
		}

		//按切换键指纹iD值加1
		if(KEY_DOWN == 0)
		{
			Delay_ms(5);
			if(KEY_DOWN == 0)
			{
				while(KEY_DOWN==0);
				if(finger_id == 1000)
				{
					finger_id = 1;
				}
				else
					finger_id = finger_id + 1;
			}		
		}

	 	//指纹iD值显示处理 
	 	LCD1602_WriteCMD(0xc0+10);
	 	LCD1602_WriteDAT(finger_id/100+48);
		LCD1602_WriteDAT(finger_id%100/10+48);
	 	LCD1602_WriteDAT(finger_id%100%10+48);

	 	//按确认键开始录入指纹信息 		 			
	 	if(KEY_OK == 0)
	 	{
	 	Delay_ms(5);
	 	if(KEY_OK == 0)
	  	{	
				while(KEY_OK==0);
			  	LCD1602_Display(0x80," Please  finger ",0,16);
			  	LCD1602_Display(0xc0,"                ",0,16);
				while(KEY_CANCEL == 1)
		  		{
			  		//按下返回键退出录入返回fingerID调整状态   
					if(KEY_CANCEL == 0) 
				 	{
				 	 	Delay_ms(5);
						if(KEY_CANCEL == 0)
						{
							while(KEY_CANCEL==0);
				  			break;
						}
						
				  	}
					AS608_Cmd_Get_Img(); //获得指纹图像
					AS608_Receive_Data(12);
					//判断接收到的确认码,等于0指纹获取成功
					if(AS608_RECEICE_BUFFER[9]==0)
				 	{
						Delay_ms(100);
						FINGERPRINT_Cmd_Img_To_Buffer1();
				    	AS608_Receive_Data(12);
						LCD1602_Display(0x80,"Successful entry",0,16);
						Beep_Times(1);
						Delay_ms(1000);
						LCD1602_Display(0x80," Please  finger ",0,16);
			  			LCD1602_Display(0xc0,"                ",0,16);
						while(1)
						{
							if(KEY_CANCEL == 0) 
				 			{
				 	 			Delay_ms(5);
								if(KEY_CANCEL == 0)
								{
									while(KEY_CANCEL==0);
				  					break;
								}
				  			}
					 		AS608_Cmd_Get_Img(); //获得指纹图像
					 		AS608_Receive_Data(12);
							//判断接收到的确认码,等于0指纹获取成功
							if(AS608_RECEICE_BUFFER[9]==0)
							{
							Delay_ms(200);
							LCD1602_Display(0x80,"Successful entry",0,16);
							LCD1602_Display(0xc0,"    ID is       ",0,16);
						 	//指纹iD值显示处理 
						 	LCD1602_WriteCMD(0xc0+10);
						 	LCD1602_WriteDAT(finger_id/100+48);
						 	LCD1602_WriteDAT(finger_id%100/10+48);
						 	LCD1602_WriteDAT(finger_id%100%10+48);
							FINGERPRINT_Cmd_Img_To_Buffer2();
				  			AS608_Receive_Data(12);
							AS608_Cmd_Reg_Model();//转换成特征码
	         				AS608_Receive_Data(12); 
					  		AS608_Cmd_Save_Finger(finger_id);                		         
	          				AS608_Receive_Data(12);
							Beep_Times(1);
							Delay_ms(1000);
							finger_id=finger_id+1;
				    		break;
				  			}
				   		}
				   		break;
					}
				}
		}
		}
	}
}

//搜索指纹
void AS608_Find_Fingerprint()
{
	unsigned int find_fingerid = 0;
	unsigned char id_show[]={0,0,0};
	do
	{
		LCD1602_Display(0x80," Please  finger ",0,16);
		LCD1602_Display(0xc0,"                ",0,16);
		AS608_Cmd_Get_Img(); //获得指纹图像
		AS608_Receive_Data(12);		
		//判断接收到的确认码,等于0指纹获取成功
		if(AS608_RECEICE_BUFFER[9]==0)
		{			
			Delay_ms(100);
			FINGERPRINT_Cmd_Img_To_Buffer1();
			AS608_Receive_Data(12);		
			AS608_Cmd_Search_Finger();
			AS608_Receive_Data(16);			
			if(AS608_RECEICE_BUFFER[9] == 0) //搜索到  
			{
				//解锁成功//
				
				LCD1602_Display(0x80," Search success ",0,16);
				LCD1602_Display(0xc0,"    ID is       ",0,16);
				Beep_Times(1);					
				//拼接指纹ID数
				find_fingerid = AS608_RECEICE_BUFFER[10]*256 + AS608_RECEICE_BUFFER[11];					
				 //指纹iD值显示处理 
				 LCD1602_WriteCMD(0xc0+10);
				 LCD1602_WriteDAT(find_fingerid/100+48);
				 LCD1602_WriteDAT(find_fingerid%100/10+48);
				 LCD1602_WriteDAT(find_fingerid%100%10+48);
				//配置IO口，执行开锁操作
				if(flag)
				{
				P1=0xfe;						
				Delay_ms(5800);
				P1=0xff;	//电动机停止转动
				Delay_ms(1000);
				P1=0xfd;	//电动机反转复位
				Delay_ms(5300);//电机正转阻力远大于反转，旋转相同角度时，正转需要更多时间	
				P1=0xff;	//电动机停止转动
				}
				flag=1;	//允许后续相关操作：添加或删除指纹模板
				break;							
			}
			else //没有找到
			{
					LCD1602_Display(0x80," Search  failed ",0,16);
					LCD1602_Display(0xc0,"                ",0,16);
				 	Beep_Times(3);
			}
			}		
		}while(KEY_CANCEL == 1);
}
//删除所有存贮的指纹库
void AS608_Delete_All_Fingerprint()
{
		unsigned char i=0;
				LCD1602_Display(0x80,"   empty all    ",0,16);
				LCD1602_Display(0xc0,"   Yes or no ?  ",0,16); 
		do
		 {
			if(KEY_OK==0)
			{
			Delay_ms(5);
			if(KEY_OK==0)
			{
				while(KEY_OK==0);
				LCD1602_Display(0x80,"   emptying     ",0,16);
				LCD1602_Display(0xc0,"                ",0,16); 
				Delay_ms(300);
				LCD1602_WriteCMD(0xc0);
				for(i=0;i<16;i++)
				 {
					LCD1602_WriteDAT(42);// 即'*'
					Delay_ms(100);
				 }
				FINGERPRINT_Cmd_Delete_All_Model();
			  	AS608_Receive_Data(12); 
				LCD1602_Display(0x80,"   All empty    ",0,16);
				LCD1602_Display(0xc0,"                ",0,16);
				Beep_Times(3);
				break;
			}
			}
		 }while(KEY_CANCEL==1);
}

void Device_Check(void)
{
		unsigned char i=0;
		AS608_RECEICE_BUFFER[9]=1;				           //串口数组第九位可判断是否通信正常
		LCD1602_Display(0xc0,"Loading",0,7);	           //设备加载中界面							   
		for(i=0;i<8;i++)						           //进度条式更新，看起来美观
		{
			LCD1602_WriteDAT(42);	                       //42对应ASIC码的 *
			Delay_ms(20);						           //控制进度条速度
		}									
		LCD1602_Display(0xc0,"Docking  failure",0,16);      //液晶先显示对接失败，如果指纹模块插对的话会将其覆盖	
		AS608_Cmd_Check();								//单片机向指纹模块发送校对命令
		AS608_Receive_Data(12);							//将串口接收到的数据转存
 		if(AS608_RECEICE_BUFFER[9] == 0)					//判断数据低第9位是否接收到0
		{
			LCD1602_Display(0xc0,"Docking  success",0,16);	//符合成功条件则显示对接成功
		}
}

//主函数
void main()
{							
	finger_id=0;
	LCD1602_Init();			//初始化液晶
	LCD1602_Display(0x80,"Fingerprint Test",0,16);	 //液晶开机显示界面
  	Uart_Init();			//初始化串口
	Key_Init();				//初始化按键
 	Delay_ms(200);          //延时500MS，等待指纹模块复位
	Device_Check();		   	//校对指纹模块是否接入正确，液晶做出相应的提示
	Delay_ms(1000);			//对接成功界面停留一定时间
	while(1)
	{
	    
		/**************进入主功能界面****************/
		LCD1602_Display(0x80,"  search finger ",0,16);	 //第一排显示搜索指纹
		LCD1602_Display(0xc0,"  Add     delete",0,16);	 //添加和删除指纹
		if(local_date==0)
		{
			LCD1602_Display(0x80,  " *",0,2);
			LCD1602_Display(0xc0,  "  ",0,2);
			LCD1602_Display(0xc0+8,"  ",0,2);	
		}
		else if(local_date==1)
		{
			LCD1602_Display(0x80,  "  ",0,2);
			LCD1602_Display(0xc0,  " *",0,2);
			LCD1602_Display(0xc0+8,"  ",0,2);	
		}
		else if(local_date==2)
		{
			LCD1602_Display(0x80,  "  ",0,2);
			LCD1602_Display(0xc0,  "  ",0,2);
			LCD1602_Display(0xc0+8," *",0,2);	
		}			
		//确认键
		if(KEY_OK == 0)
		{
		Delay_ms(5);
		if(KEY_OK == 0)
		{	 
		 	while(KEY_OK == 0);//等待松开按键								
			switch(local_date)
			{
					case 0:  //搜索指纹	
					flag=1;					
					AS608_Find_Fingerprint();																								
					break;	
					
					case 1:	 //添加指纹
					flag=1;	//flag=1，若指纹库为空，则可以直接添加指纹				
					if(AS608_number_of_fingers())
					{
						flag=0;//flag置0由两重作用：
						//1、指纹库中已有指纹，则需要搜索匹配成功，由AS608_Find_Fingerprint()将flag置1，才能添加指纹
						//2、flag=0，则在搜索指纹成功后不执行开锁操作
						AS608_Find_Fingerprint();
					}
					if(flag)
					{
						AS608_Add_Fingerprint();
					}
					break; 					
					
					case 2:	//清空指纹
					flag=0;	//1、在搜索指纹成功后不执行开锁操作；2、若搜索不成功，不执行清空操作
					AS608_Find_Fingerprint();//搜索匹配成功后，函数内部将flag置1，才能清空指纹库
					if(flag)
					{
						AS608_Delete_All_Fingerprint();
		  			}
					break;
			}
		}
		}
		    //切换键
			if(KEY_DOWN == 0)
			{
			Delay_ms(5);
			if(KEY_DOWN == 0)
			{
			 	while(KEY_DOWN == 0); //等待松开按键				
	  	 		if(local_date<=2)
				{
					local_date++;
					if(local_date==3) local_date=0;						
				}		
			}
			}						
			Delay_ms(100); //延时判断100MS检测一次		
	}
}