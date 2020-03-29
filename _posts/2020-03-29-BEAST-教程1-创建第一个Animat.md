---
layout: post
title:  "BEAST 教程1 创建第一个Animat"
date:   2020-03-29"
excerpt: "创建简单的Animat显示在仿真环境中"
tag:
- 进化学习
- 机器学习
comments: false
---

BEAST的代码风格与面向对象的高级语言如C#更相似，与常规C++的区别主要在于properties的使用，以及存在伪This的引用。

# 内容介绍

• 介绍C++的animats接口

• 从基础animat类派生新的animat

• 简单示例

首先，了解animat.h中包含的c++接口（简略版）：

    class Animat : public WorldObject
    {
    public:
        Animat();
        ~Animat();
        virtual void    Init();
        void            Add(std::string name, Sensor* s);

        virtual void    Control(){}

        virtual void    OnCollision(WorldObject* r){}

    protected:
        SensorContainer sensors;
        ToolContainer tools;
        ControlContainer controls;

        Vector2D        velocity;
        int             maximumSpeed;
        double          maxRotateSpeed;
    };
    
可以看出Animat是从WorldObject派生的，WorldObject是从Drawable派生的，所以还得看一下Drawable和WorldObject。

Drawable是屏幕上的World中一切的基础类，包括world中的animats，objects以和obstacles，以及显示功能（碰撞的点，动画轨迹，传感器输出）。而如此重要的Drawable唯一最重要的特征就是：

    virtual void    Init();

    Vector2D            location;
    double              orientation;

对象的位置从屏幕的左下角开始，作为Vector存储，因此location实际上就是x，y坐标。

与程序中所有orientation一样，这里的orientation弧度是逆时针的并从东边开始计数。

仿真时会调用Init()，进行对象的设置。


WorldObject源自Drawable，提供碰撞检测和Update这种探测生命周期的函数，同时也是仿真中涉及的一切东西的基础类（但是不包括纯Drawable的碰撞点，轨迹及其他显示功能）。

Update会在每一帧的开始处被调用，并且Animat会进行“思考”并移动，这通常取决于传感器的输出。


接下来是Animat类的内容。Animat是移动的，所以还有个描述速度的vector，配备了各种传感器来得知有关环境信息，以及操作环境的工具。

ControlContainer是一系列真值，控制左右轮子。

关于如何将轮子速度设置为0的示例代码：

    Controls["left"] = 0.0;
    
    Controls["right"] = 0.0;

我们可以这样写一些上面这样简单的Animats并将它们放进仿真环境，但是它们不会做任何事。如果需要让它们行动起来，我们需要派生一些新的类。

# 编写活动的Animats

我们可以编写一些彼此跟随的Animats，我们命名为Shrew，这里的Shrew没有视力，只跟随它们的“mother”，如果丢失了mother，它将跟随前面无论是什么的对象。

Shrew就是一个简单的带俩传感器的animat，我们将它写在shrew.cc里。

首先引入头文件：

    #include "wxsimenv.h"

再引入Animat头文件：

    #include "animat.h"

    #include "sensor.h"

接下来我们指定命名空间GASimEnv：

    using namespace GASimEnv;

现在就可以创建新的animat了，也就是shrew：

    class Shrew : public Animat // Shrew is derived from Animat
    {
    public:
        Shrew()
        {
            This.Add("left", ProximitySensor<Shrew>(PI/5, 200.0, -PI/20));
            This.Add("right", ProximitySensor<Shrew>(PI/5, 200.0, PI/20));

            This.InitRandom = true;     // Start in random locations

            This.Radius = 10.0;         // Shrews are a little bigger than usual
        }
        virtual ~Shrew(){}

        virtual void Control()
        {
            This.Controls["left"] = This.Sensors["right"]->GetOutput();
            This.Controls["right"] = This.Sensors["left"]->GetOutput();
        }
    };

注意覆写的Control方法具有关键字virtual，这确保了即使模拟环境不知道它是包含着普通animat对象，特殊的派生animat还是混合产物，都能在运行时选择正确的control function。而如果没有virtual，则会忽略新control方法，而调用原始的空方法。

定义好之后就可以仿真了：

    class ShrewSimulation : public Simulation
    {
        Group<Shrew> grpShrew;

    public:
        ShrewSimulation():
        grpShrew(30)
        {
            This.Add("Shrews", grpShrew);
        }
    };

ShrewSimulation就是从Simulation派生的，grpShrew是个成员集合，一个Group就是一个加了点methods的vector。

ShrewSimulation要做的两件事：配置创建有30个shrews的grpShrew，并把这个group取名为Shrews加到World中。

最后我们使用宏向仿真环境声明新的仿真类：

    BEGIN_SIMULATION_TABLE
        ADD_SIMULATION("Shrews", ShrewSimulation)
    END_SIMULATION_TABLE
    
可以看到模拟将进入一个table，并且已经能够编译从“File”菜单中启动的最多十个仿真环境。

要编译仿真，只需要输入（假设代码文件名为shrew.cc）：

    make shrew

一会再输入下行就可以看到仿真了：

    ./shrew

接下来可以看到屏幕上出现了许多活动的shrew-bots。

# BEAST教程索引

[BEAST 教程0 代码说明](https://yawwq.github.io/BEAST-%E6%95%99%E7%A8%8B0-%E4%BB%A3%E7%A0%81%E8%AF%B4%E6%98%8E/)

[BEAST 教程1 创建第一个Animat](https://yawwq.github.io/BEAST-%E6%95%99%E7%A8%8B1-%E5%88%9B%E5%BB%BA%E7%AC%AC%E4%B8%80%E4%B8%AAAnimat/)

[BEAST 教程2 添加Object和交互式Animat](https://yawwq.github.io/BEAST-%E6%95%99%E7%A8%8B2-%E6%B7%BB%E5%8A%A0Object%E5%92%8C%E4%BA%A4%E4%BA%92%E5%BC%8FAnimat/)

[BEAST 教程3 引入遗传算法](https://yawwq.github.io/BEAST-%E6%95%99%E7%A8%8B3-%E5%BC%95%E5%85%A5%E9%81%97%E4%BC%A0%E7%AE%97%E6%B3%95/)

# Reference

https://minerva.leeds.ac.uk/webapps/blackboard/content/listContent.jsp?course_id=_505438_1&content_id=_6865832_1&mode=reset
