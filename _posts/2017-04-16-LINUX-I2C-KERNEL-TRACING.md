---
layout: post
title: "Linux I2C kernel tracing"
date: 2017-04-17 11:30:00
tags: linux
description: trace I2C kernel module之過程
---

### **前言**
在嵌入式課程的專案中，嘗試開發I2C LKM沒有成功，之後自行研究有了一些心得，遂以此為記
<br>

### **環境**
- 開發板：[nanopi m1](http://wiki.friendlyarm.com/wiki/index.php/NanoPi_M1)
- OS：Debian3.4.39
- Sensor:[DHT12](https://github.com/Bobadas/DHT12_library_Arduino/blob/master/DHT12%E6%95%B0%E5%AD%97%E6%B8%A9%E6%B9%BF%E5%BA%A6%E4%BC%A0%E6%84%9F%E5%99%A8%EF%BC%88V1.3-20160315%EF%BC%89.pdf)
<br>

### **Character Driver**
- 驅動是以character driver的形式實現
- 訊號控制則是透過gpio
- 測試之後發現效果不佳，即便透過spin_lock_irq()/ spin_unlock_irq()取消中斷，傳輸速度最快也只能達到50KHZ~60KHZ
- 發現可能原因在於gpio系統函式存取時間過長
![image alt](https://raw.githubusercontent.com/jarvis1984/DHT12/master/nodelay.png)
<br>

### **I2C LKM**
- 另一方面，nanopi提供的使用者函式庫卻可以快速而精準的發出訊號，所以開始trace其功能實現
![image alt](https://raw.githubusercontent.com/jarvis1984/DHT12/master/twi.png)
- 首先查閱cpu(Allwinner H3)的[datasheet](http://wiki.friendlyarm.com/wiki/images/4/4b/Allwinner_H3_Datasheet_V1.2.pdf)發現cpu提供了三個I2C控制器，透過memory mapping的方式供使用者操作
- 使用者函式庫[matrix](https://github.com/friendlyarm/matrix)提供的函式是以i2c_smbus_access中的ioctl進入系統核心
- 雖然知道是透過ioctl檔案系統操作，但問題是相關操作對應之實現是放在哪個模組內？藉由[Linux下I2C驅動架構全面分析](https://read01.com/o2Dya.html)一文，發現在kernel source code的driver下的i2c目錄包含有i2c相關實現
- 其中i2c-core.c實現了核心功能以及/proc/bus/i2c介面；而i2c-dev.c則實現了檔案系統相關操作介面
- 在i2c-dev.c中搜尋ioctl，發現檔案系統操作介面結構初始化如下
```clike=
static const struct file_operations i2cdev_fops = {
	.owner		= THIS_MODULE,
	.llseek		= no_llseek,
	.read		= i2cdev_read,
	.write		= i2cdev_write,
	.unlocked_ioctl	= i2cdev_ioctl,
	.open		= i2cdev_open,
	.release	= i2cdev_release,
};
```
- 再透過[What is the difference between ioctl(), unlocked_ioctl() and compat_ioctl()?](http://unix.stackexchange.com/questions/4711/what-is-the-difference-between-ioctl-unlocked-ioctl-and-compat-ioctl)一文，了解到kernel自2.6.36之後已將ioctl取代為unlocked_ioctl及compat_ioctl
- 在i2cdev_ioctl中，依據輸入指令執行不同工作，而使用者函式i2c_smbus_access呼叫ioctl時輸出指令為I2C_SMBUS，因此接下來對應的函式是i2cdev_ioctl_smbus
- 在i2cdev_ioctl_smbus中會先呼叫copy_from_user讀取使用者參數並進行相關處理，然後再呼叫i2c_smbus_xfer
- i2c_smbus_xfer定義在i2c-core.c，此時會判斷抽象層adapter是否實現smbus_xfer；若否則呼叫i2c_smbus_xfer_emulated，而i2c_adapter在i2c.h中，其結構如下
```clike=
 /*
    i2c_adapter is the structure used to identify a physical i2c bus along
    with the access algorithms necessary to access it.
  */
struct i2c_adapter {
	struct module *owner;
	unsigned int class;		  /* classes to allow probing for */
	const struct i2c_algorithm *algo; /* the algorithm to access the bus */
	void *algo_data;

	/* data fields that are valid for all devices	*/
	struct rt_mutex bus_lock;

	int timeout;			/* in jiffies */
	int retries;
	struct device dev;		/* the adapter device */

	int nr;
	char name[48];
	struct completion dev_released;

	struct mutex userspace_clients_lock;
	struct list_head userspace_clients;
};
```
- i2c_algorithm結構定義則如下
```clike=
 /* 
    The following structs are for those who like to implement new bus drivers:
    i2c_algorithm is the interface to a class of hardware solutions which can
    be addressed using the same bus algorithms - i.e. bit-banging or the PCF8584
    to name two of the most common. 
  */
struct i2c_algorithm {
	/* If an adapter algorithm can't do I2C-level access, set master_xfer
	   to NULL. If an adapter algorithm can do SMBus access, set
	   smbus_xfer. If set to NULL, the SMBus protocol is simulated
	   using common I2C messages */
	/* master_xfer should return the number of messages successfully
	   processed, or a negative value on error */
	int (*master_xfer)(struct i2c_adapter *adap, struct i2c_msg *msgs,
			   int num);
	int (*smbus_xfer) (struct i2c_adapter *adap, u16 addr,
			   unsigned short flags, char read_write,
			   u8 command, int size, union i2c_smbus_data *data);

	/* To determine what the adapter supports */
	u32 (*functionality) (struct i2c_adapter *);
};
```
- 所以接下來需要先判斷adapter是否實現smbus_xfer，再回到i2c-dev.c查看i2c_dev_init，其中有一段程式碼
```clike=
res = bus_register_notifier(&i2c_bus_type, &i2cdev_notifier); 
```
- 其中i2cdev_notifier變數定義如下
```clike=
static struct notifier_block i2cdev_notifier = {
	.notifier_call = i2cdev_notifier_call,
};
```
- 在文章[Linux usb子系統（一）：子系統架構](http://b8807053.pixnet.net/blog/post/237079804-linux-usb%E5%AD%90%E7%B3%BB%E7%B5%B1%EF%BC%88%E4%B8%80%EF%BC%89%EF%BC%9A%E5%AD%90%E7%B3%BB%E7%B5%B1%E6%9E%B6%E6%A7%8B)中對bus_register_notifier的用途有所說明
> 大多數kernel子系統都是相互獨立的，因此某個子系統可能對其他子系統產生的事件感興趣。為了滿足這個需求，也即是讓某個子系統在發生某個事件時通知其他的子系統，Linuxkernel提供了通知鏈的機制。通知鏈表只能夠在kernel的子系統之間使用，而不能夠在kernel與用戶空間之間進行事件的通知。
- 所以透過bus_register_notifier註冊之後，系統發現符合i2c類型的設即會呼叫i2cdev_notifier_call，而在函式中可看見有處理兩種狀況：BUS_NOTIFY_ADD_DEVICE、BUS_NOTIFY_DEL_DEVICE，所以發現新裝置會呼叫i2cdev_attach_adapter
- 在i2cdev_attach_adapter中可看見註冊i2c的device參數、建立devfs以及初使化檔案系統等操作，但並未看見smbus_xfer；同時adapter是由輸入參數device而來，如果要確定則必須找到device參數初始化的地方
- 因為沒有看到algorithm下之smbus_xfer實現，先假設系統沒有對應的實作，則此時i2c_smbus_xfer會呼叫i2c_smbus_xfer_emulated
- 在i2c_smbus_xfer_emulated中首先會依據protocol對不同類型資料做處理產生輸出訊息，接著呼叫i2c_transfer傳遞訊息
- 而i2c_transfer則會呼叫algorithm對應之master_xfer實作
- 首先在algos資料夾下發現標準實作檔案i2c-algo-bit.c，此檔案有經過核心編譯變產生物件檔，同時裡面包含一組i2c_algorithm實作：
```clike=
const struct i2c_algorithm i2c_bit_algo = {
	.master_xfer	= bit_xfer,
	.functionality	= bit_func,
};
```
- 此時可以注意到在此algorithm並沒有指定smbus_xfer之實作，另外master_xfer則是對應到bit_xfer
- 在bit_xfer中則是呼叫readbytes、sendbytes來分別讀、寫訊息
- 在readbytes、sendbytes中則是又分別呼叫i2c_inb、i2c_outb來進行輸出入
- 然而在i2c_inb、i2c_outb可以看到資料最終是透過sclhi、scllo、setsda、getsda等指令控制訊號；透過udelay控制時序，這跟我用的方法幾乎相同，但是這樣的作法可以達到精準的400khz(2.5us)週期性控制嘛？
- 另外在busses資料夾下有一個i2c-sunxi.c實作檔，此檔案同樣經過編譯，而其i2c_algorithm介面如下：
```clike=
static const struct i2c_algorithm sunxi_i2c_algorithm = {
	.master_xfer	  = sunxi_i2c_xfer,
	.functionality	  = sunxi_i2c_functionality,
};
```
- 同樣的，在這裡也沒有看到沒有指定smbus_xfer對應之介面，所以可以合理懷疑確實沒有smbus_xfer之實作；另外其master_xfer對應到sunxi_i2c_xfer
- 在sunxi_i2c_xfer中會依據重試次數呼叫sunxi_i2c_do_xfer
- 而sunxi_i2c_do_xfer則會啟動Allwinner H3的twi(two wire interface)介面的相關設定，包含重置(twi_soft_reset)、啟動中斷(twi_enable_irq)以及相關參數等，最後再呼叫twi_start在對應register寫入TWI_CTL_STA指令、送出i2c start訊號，不過中斷服務函式是在哪裡註冊的呢？
- 首先觀察模組初始化sunxi_i2c_adap_init，其中一開始呼叫了sunxi_twi_device_scan，很顯然是用來掃描twi週邊裝置
- 觀察sunxi_twi_device_scan可以發現其功能在於設定twi資源以及裝置，包含twi memory-mapped io相關設定
- 接著sunxi_i2c_adap_init會呼叫platform_device_register註冊twi裝置、sunxi_i2c_sysfs設定sysfs
- 最後再呼叫platform_driver_register註冊twi驅動
- 值得注意的是sunxi_i2c_adap_init是以subsys_initcall macro註冊，根據[linux子系统的初始化_subsys_initcall()：那些入口函数](http://blog.csdn.net/yimiyangguang1314/article/details/7312209)一文，其含意為"module_init 调用优先级为6低于subsys_initcall调用优先级4."
- 另外在檔案中發現註冊之驅動sunxi_i2c_driver內容如下：
```clike=
static struct platform_driver sunxi_i2c_driver = {
	.probe		= sunxi_i2c_probe,
	.remove		= __devexit_p(sunxi_i2c_remove),
	.driver		= {
		.name	= SUNXI_TWI_DEV_NAME,
		.owner	= THIS_MODULE,
		.pm		= SUNXI_I2C_DEV_PM_OPS,
	},
};
```
- 在[Linux设备模型(5)_device和device driver](http://www.wowotech.net/linux_kenrel/device_and_driver.html)中提到：
>probe、remove，这两个接口函数用于实现driver逻辑的开始和结束。Driver是一段软件code，因此会有开始和结束两个代码逻辑，就像PC程序，会有一个main函数，main函数的开始就是开始，return的地方就是结束。而内核driver却有其特殊性：在设备模型的结构下，只有driver和device同时存在时，才需要开始执行driver的代码逻辑。这也是probe和remove两个接口名称的由来：检测到了设备和移除了设备（就是为热拔插起的！）。
...
>设备驱动prove的时机有如下几种（分为自动触发和手动触发）： 
> - 将struct device类型的变量注册到内核中时自动触发(device_register，device_add，device_create_vargs，device_create） 
> - 将struct device_driver类型的变量注册到内核中时自动触发（driver_register） 
> -  手动查找同一bus下的所有device_driver，如果有和指定device同名的driver，执行probe操作（device_attach） 
> - 手动查找同一bus下的所有device，如果有和指定driver同名的device，执行probe操作（driver_attach） 
> - 自行调用driver的probe接口，并在该接口中将该driver绑定到某个device结构中----即设置dev->driver（device_bind_driver） 
- 所以可以知道sunxi_i2c_probe會在platform_driver_register註冊driver自動被呼叫，而啟動後會開始設定i2c控制結構(struct sunxi_i2c)相關參數，其中有一行指令如下：
```clike=
ret = request_irq(irq, sunxi_i2c_handler, IRQF_DISABLED, i2c->adap.name, i2c);
```
- 所以中斷服務函式sunxi_i2c_handle在初始化的時候就註冊好了，後續在傳送i2c指令時，在函式sunxi_i2c_do_xfer中啟動irq自然就會依照前面設定好的i2c參數開始執行數據收發
- 另外裡面有一行指令如下：
```clike=
i2c->base_addr = ioremap(res->start, resource_size(res));
```
- 依據[Linux內核中ioremap映射的透徹理解](https://read01.com/Jyz2G.html)一文
>一般來說，在系統運行時，外設的I/O內存資源的物理地址是已知的，由硬體的設計決定。但是CPU通常並沒有為這些已知的外設I/O內存資源的物理地址預定義虛擬地址範圍，驅動程序並不能直接通過物理地址訪問I/O內存資源，而必須將它們映射到核心虛地址空間內（通過頁表），然後才能根據映射所得到的核心虛地址範圍，通過訪內指令訪問這些I/O內存資源。Linux在arch/xxx/include/asm/io.h頭文件中聲明了函數ioremap，用來將I/O內存資源的物理地址映射到核心虛地址空間（3GB－4GB）中，原型如下（各體系結構不一樣）：
void * ioremap(unsigned long phys_addr, unsigned long size, unsigned long flags);
iounmap函數用於取消ioremap所做的映射，原型如下：
void iounmap(void * addr);
- 因此這行指令就是在將twi的mmio實體位置(依據cpu spec)映射到虛擬空間，以供後續存取twi控制器之用
- sunxi_i2c_probec後續還呼叫了sunxi_i2c_hw_init，負責設定硬體如gpio腳位、clock等
- 然後還呼叫了i2c_add_numbered_adapter，此函數定義在i2c-core.c中，從字面上就可以知道是用來註冊i2c對應之adapter；相較之下，在i2c-algo-bit.c珠其實沒有看到初始化函式
- 知道了sunxi_i2c_handler啟動機制之後，接下來觀察此函式作用，從程式碼可以看到其中主要工作流程在其呼叫的sunxi_i2c_core_process
- 進入sunxi_i2c_core_process之後首先會呼叫twi_query_irq_status取得目前狀態，再依照目前狀態執行後續動作
- 而其中訊息輸出入是呼叫twi_put_byte、twi_get_byte等指令
- 而在twi_put_byte、twi_get_byte之中其實只是呼叫writel、readl來寫、讀twi資料暫存器，由此可以明確知道i2c-sunxi.c的控制方法是透過mmio控制twi控制器來進行資料傳輸，而很顯然的這樣的硬體控制方法理論上是比較符合之前的實驗結果，然而因為device參數得傳遞部份還未完全釐清，是否有其他證據支持我們的測試結果就是使用這個方法？
- 一般驅動裝置都會在sysfs產生對應檔案，此時或許可以透過觀察實際產生的檔案來回推i2c實作；觀察初始化程序sunxi_i2c_adap_init發現其中有呼叫建立sysfs的函式sunxi_i2c_sysfs
- 在sunxi_i2c_sysfs中呼叫了device_create_file在裝置目錄下分別建立了info、status、unittest等屬性檔案，以及各屬性對應的資料顯示函式(show)
- 觀察開發板上的sysfs的確可以看見相關檔案，也因此證實了實際上系統函式庫是使用硬體控制器來控制i2c訊號
- 反之，i2c-algo-bit.c之中則沒有看到sysfs相關設定
- 其實由我們的實驗結果就可以知道，透過軟體方式來控制訊號有其成本，在本實驗平台上大概7~8us一個週期就是極限了；也因此無怪乎開發板要提供硬體控制器
<br>

### **待釐清問題**
- i2cdev_detach_adapter之device參數在那以及如何傳入？
- sunxi_i2c_adap_init是被誰、什麼時候呼叫？
<br>

### **程式碼**
https://github.com/jarvis1984/DHT12
