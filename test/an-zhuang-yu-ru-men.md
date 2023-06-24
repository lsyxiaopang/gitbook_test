# 安装与入门

{% hint style="info" %}
**需要注意的事项:**由于grbl的硬件配置与软件配置均是在**初始化时进行的,**因此本库仅负责与grbl之间进行G-code的通讯,具体的接线和相关设置需要由用户事先准备好
{% endhint %}

## 安装这个库

由于这个库目前还在完全的开发阶段,因此还没有已经发布的库,只能将代码嵌入到原始工程文件中进行操作

请在[这个网页中](http://10.92.1.90:7654/117lab/fdurop\_3D\_mag/raspcode/-/tree/test\_branch)将库下载下来(在内网条件),并且解压到你的工作目录

你需要将`Controller.py`与你的工程文件放到一个目录下,之后即可调用

{% hint style="info" %}
**有关环境配置:**本库采用了一下库实现功能

* `pandas`
* `numpy`
* `loguru`
* 其他的树莓派库(可选,实际上用户应当自己编写`MeasureControl`)
{% endhint %}

## 库的使用

### 编写MeasureControl

首先在使用时,用户需要自行编写`MeasureControl`并且作为一个参数传给`Controller`中,为了保证`Controller`可以正确处理`MeasureControl`的信息,**你的类中应当包含以下方法与属性**:

* 不接受任何参数的`__init__`(初始化)方法
* 拥有`measure`方法,`Controller`在每一次移动完成时都会调用这个方法,其同时接收一个元组`(x,y,z,measure)`指示了当前仪器所在的位置,以及是否进行测量的控制位
* 拥有`rough_measure_out`属性,`Controller`在调用`measure_refresh_call`时会传入这个参数,其中的内容应当是`MeasureControl`到目前为止所测量的所有数据(个人建议是一个`DataFrame`)
* 拥有`result_now`属性(主要[将在adaptive](an-zhuang-yu-ru-men.md#shi-yong-adaptive)中使用)

<details>

<summary>一个最为简单的<code>MeasureControl</code>的例子</summary>

这个例子中的`MeasureControl`什么也干不了,不过是一个展示最基本要求的好例子,或者如果你**希望不进行任何的测量操作**,也是可行的选择

```python
class MeasureControl:
    def __init__(self)->None:
        self.rough_measure_out=0
        self.result_now=0
    def measure(self,cmd_input)->None:
        pass
```

</details>

<details>

<summary>一个完整的<code>MeasureControl</code>的例子</summary>

以下代码来自于我为磁场传感器编写的代码,比较复杂,但是基本展示了所有`MeasureControl`应当实现的功能

```python
import qmc5883l as qmc
import adafruit_mlx90393 as mlx
import board
import time
import numpy as np
import pandas as pd
from loguru import logger

class MeasureControl:
    def __init__(self) -> None:
        logger.info("Trying to connect to the measure device!")
        self.init()
        
    def init(self):
        i2c=board.I2C()
        self.i2c=i2c
        self.mlx=mlx.MLX90393(i2c,0x18,oversampling=mlx.OSR_3)#*将过采样率提高
        self.qmc=qmc.QMC5883L(i2c)
        #*这里还缺一个激活TMAG的
        self.qmc.field_range=qmc.FIELDRANGE_8G
        self.qmc.oversample=qmc.OVERSAMPLE_512
        self.qmc.mode_control=qmc.MODE_CONTINUOUS#*开启QMC
        self.mlx.oversampling=mlx.OSR_3#*将过采样率开高
        logger.info("Measure device connected!")
        self.device_table=[
            [500,(0,0),lambda:self.mlx_measure(mlx.RESOLUTION_17,mlx.GAIN_5X),"MLX 500Gs"],     #*格式:[量程,坐标偏移,操作命令,名称]
            [250,(0,0),lambda:self.mlx_measure(mlx.RESOLUTION_17,mlx.GAIN_2_5X),"MLX 250Gs"],      
            [50,(0,0),lambda:self.mlx_measure(mlx.RESOLUTION_16,mlx.GAIN_1X),"MLX 50Gs"],
            [8,(8.001,0.41),lambda:self.qmc_measure(qmc.FIELDRANGE_8G),"QMC 8G"],#TODO 检查QMC芯片的偏移坐标
            [2,(8.001,0.41),lambda:self.qmc_measure(qmc.FIELDRANGE_2G),"QMC 2G"]      
        ]
        self.rough_measure_out=pd.DataFrame()
        
        
    def mlx_measure(self,res,gain):
        self.mlx.resolution_x=res
        self.mlx.resolution_y=res
        self.mlx.resolution_z=res
        self.mlx.gain=gain
        t=self.mlx.magnetic#*读走一次数据
        # time.sleep(0.1)
        x,y,z=self.mlx.magnetic#TODO在这里不考虑多次采样与误差了,未来会考虑
        return y/100,x/100,z/100#!输出单位uT,转换为Gs
    
    def mlx_multimeasure(self,f):
        num=10
        m_data=[]
        for i in range(num):
            m_data.append(self.safe_measure(f))
        data_array=np.array(m_data)
        xmean=data_array[:,0].mean()
        ymean=data_array[:,1].mean()
        zmean=data_array[:,2].mean()
        xerr=data_array[:,0].std()/np.sqrt(num-1)
        yerr=data_array[:,1].std()/np.sqrt(num-1)
        zerr=data_array[:,2].std()/np.sqrt(num-1)
        return xmean,xerr,ymean,yerr,zmean,zerr

    def qmc_measure(self,range):#!需要对x轴和y轴旋转
        self.qmc.field_range=range
        t=self.qmc.magnetic
        time.sleep(0.2)
        x,y,z=self.qmc.magnetic#!x,y调换
        return x,-y,z
    
    def safe_measure(self,measure_func):#!此处定义了如果出现错误之后重新加载
        retry_max=5#!此处为最大重试次数
        retry_now=0
        while True:
            try:
                if retry_now>0:
                    self.init()
                    time.sleep(0.2)
                data=measure_func()
                break
            except:
                retry_now+=1
                if retry_now>retry_max:
                    logger.error("Maximum number of retries reached. Exiting!")
                    raise
                else:
                    self.i2c.deinit()
                    logger.warning("An error happened in measure device. Retry time {}".format(retry_now))
                    time.sleep(0.3)#*先暂停一下,防止过快
        return data            
    
    def measure(self,cmd_input):
        x0,y0,z,_=cmd_input
        logger.info("One measure started at:({},{},{})",x0,y0,z)
        device=""
        for i in range(len(self.device_table)):
            test_device=self.device_table[i]
            bx,by,bz=self.safe_measure(test_device[2])
            x=x0+test_device[1][0]
            y=y0+test_device[1][1]
            device=test_device[3]
            if i+1<(len(self.device_table)):
                new_dev=self.device_table[i+1]
                if(max(abs(bx),abs(by),abs(bz))<3*new_dev[0]/4):#阈值选取3/4
                    logger.debug("Device {} get({},{},{}),switching to the next device!",
                                device,bx,by,bz)
                    continue
                else:
                    logger.debug("Device {} get({},{},{}),measure complete!",
                                device,bx,by,bz)
                    bx,xerr,by,yerr,bz,zerr=self.mlx_multimeasure(test_device[2])
                    break
            else:
                logger.debug("Device {} get({},{},{}),measure complete!",
                                device,bx,by,bz)
                bx,xerr,by,yerr,bz,zerr=self.mlx_multimeasure(test_device[2])
                break
        out_data_dict={
            "xmean":[bx],
            "xstd":[xerr],#!注意,这里两个名称不符,应当统一使用**测量误差**
            "ymean":[by],
            "ystd":[yerr],
            "zmean":[bz],
            "zstd":[zerr],
            "inx":[x],
            "iny":[y],
            "inz":[z],
            "measure device":[device]
            }
        odf=pd.DataFrame(out_data_dict)
        self.rough_measure_out=pd.concat([self.rough_measure_out,odf])
        self.result_now=(bx,by,bz)
```



</details>

### 编写回调函数

由于这个库是**异步运行**的,因此通过回调函数来使调用者知道各个不同的事件

回调函数也是保存在一个类中的,但由于`Controller`会调用的回调函数很多,但是实际上用到的很少,**因此使用类继承来实现默认行为**,基类的代码如下所示

```python
class ControllerCallbackBase:
    def status_refresh_call(self,newstatus: str):
        pass

    def coordinate_refresh_call(self,newcoordinate:tuple):
        pass
    
    def cmd_refresh_call(self,new_cmd):
        pass

    def measure_refresh_call(self,datalist):
        pass

    def no_command_call(self):
        pass

```

其中每个回调函数的目的如下所示:

* `status_refresh_call`:如果装置状态发生了改变`("MOVING","MEASUREING","NO CMD","STOP")`,这个函数会被调用,并且参数为当前的状态信息
* `coordinate_refresh_call`:`Controller`每隔一段时间会轮询一下电机的状态,其中参数包含了当前的电机坐标
* `coordinate_refresh_call`:每当一个新的命令被执行时,会调用这个函数(未来有更复杂的命令时会很有用)
* `measure_refresh_call`:每当测量完成一次,会调用这个函数,参数传入了`MeasureControl.rough_data_out`这个用户自行完成的属性
* `no_command_call`:发生于所有命令执行完毕,提醒用户没有进一步指令了

<details>

<summary>一个完整回调函数的例子</summary>

下面展示了一个回调函数是如何工作的

在这里需要特别提醒一下,在回调函数类初始化函数中**可以包含你自己的类**,实现数据传输

```python
class ApplicationControllerCallback(Controller.ControllerCallbackBase):
    def __init__(self,cui) -> None:
        self.cui=cui
        super().__init__()

    def measure_refresh_call(self, datalist):
        measureout=get_measure_out(datalist)
        if measureout:
            self.cui.data_out.clear()
            self.cui.data_out.add_item_list(measureout)
        return super().measure_refresh_call(datalist)

    def status_refresh_call(self, newstatus: str):
        self.cui.check_flag(newstatus)
        return super().status_refresh_call(newstatus)
    
    def cmd_refresh_call(self, new_cmd):
        self.cui.refresh_cmd(new_cmd)
        return super().cmd_refresh_call(new_cmd)

    def no_command_call(self):
        self.cui.no_cmd()
        return super().no_command_call()

    def coordinate_refresh_call(self, newcoordinate: tuple):
        self.cui.refresh_coordinate(newcoordinate)
        return super().coordinate_refresh_call(newcoordinate)

```

</details>

<details>

<summary>一个简易回调函数的例子</summary>

下面这个回调函数就比较简单,其只在没有命令的时候才会有非默认的行为,即保存数据并且退出

同时,我们也需要注意到,在调用回调函数时,**变量的作用域回到了用户中**,比如说,例子中tc就是一个全局变量,定义了此时使用的Controller

```python
class BatchControllerCallback(Controller.ControllerCallbackBase):
    def __init__(self,signal) -> None:
        super().__init__()

    def no_command_call(self):
        logger.info("Finished!")
        outdf=tc.measure_control.rough_measure_out
        outdf.to_csv("output.csv")
        pid=os.getpid()
        os.kill(pid,signal.SIGTERM)
        return super().no_command_call()
```

</details>

### 连接
在完成以上步骤之后,要做的就是初始化和连接
{% hint style="info" %}
需要注意,这里所有和硬件相关的内容均需要用户自行配置
此时的电机需要支持一下功能:
- 复位功能(确保限位开关正确安装)
- 信息反馈符号位设置正确
{% endhint %}
首先是初始化,用户需要初始化回调函数和自己的`MeasureControl`,接着向`Controller`的初始化器中传入相关信息
```python
ct = Controller(None,"COM9",my_controller_callback,my_measure_control)
```
这里展示了一个在Windows中初始化一个`Controller`的例子(假定连接于`COM9`)

在初始化时,函数会以阻断形式进行归位(`$H`)操作,归位完成且没有其他异常后函数会返回,用户可以进行下一步操作
### 添加指令
接下来是向`Controller`中添加指令,假如我们要移动到某一个点`(x,y,z)`并且要执行测量,那么我们可以使用`Controller.add_cmd(point,True)`将这一条命令添加进去

{% hint style="info" %}
这里添加指令的操作在加载了路径规划器后会有更加丰富的结果,可以参考相关的文档
{% endhint %}

### 运行指令
需要注意到的是,我们这里运行指令**不是通过调用某一个函数实现**,而是通过将`move_start`属性设置为`True`,机器就会自动开始运行,相似地,将`move_start`属性为`False`,机器也就会停止.

{% hint style="warning" %}
这里的`move_start`设置为`False`后系统**仍然会完成当前的运动**,因此不能被用作紧急停止装置
{% endhint %}

而当所有命令执行完毕后,系统将自动将`move_start`设置为`False`,并且同时调用 `no_command_call`提醒用户所有命令已经执行完毕



