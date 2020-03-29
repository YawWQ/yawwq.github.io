---
layout: post
title:  "BEAST 教程2 添加Object和交互式Animat"
date:   2020-03-29"
excerpt: "往world中增加objects，并让animats与它们互动。我们在本节中将会创建Mouse类并完成吃Cheese的功能。"
tag:
- 进化学习
- 机器学习
comments: false
---

# 内容介绍

•  往world中增加objects

•  让animats与objects互动

•  手写简单的neural controller

•  通过继承创建更多特殊对象

•  传感器的运作

当我们希望我们的仿真能让Animats与对象具有互动关系，比如说吃东西或者掉进某个洞里，则需要添加一些代码来响应。

# 编写会吃奶酪（Cheese）的Animat

这将会吃cheese的类就是Mouse，Mouse的世界只有两个类型：其他老鼠和奶酪，奶酪会用黄色点表示。

我们给Mouse一个简单的sensor，叫做NearestAngleSensor，告诉我们Mouse与指定类型的最近对象的角度：左边是负角度，右边是正角度，0是正前方。（那正后方不管咯？（杠（但是按前面的shrew来理解animat只跟随前面的东西

引入头文件：

    #include "wxsimenv.h"
    #include "animat.h"
    #include "sensor.h"

定义Cheese类：

    class Cheese : public WorldObject
    {
    public:
        Cheese()
        {
            This.Radius = 2.5f;                     // Cheeses are quite small 奶酪比较小
            This.SetColour(ColourPalette[COLOUR_YELLOW]);// Cheeses are yellow
            This.InitRandom = true;                 // Cheases are scattered 奶酪是分散的
        }
        virtual ~Cheese(){}
    // When a Cheese is Eaten, it reappears in a random location.
    // 当奶酪被吃掉，会在一个随机地点重新出现
        void Eaten()
        {
            This.Location = myWorld->RandomLocation();
        }
    };

Cheese的构造函数将半径设置为2.5了，颜色设置为黄色，
initRandom标志设置为true，这样Cheese将随机分布在World中。

Cheese类带有强制的虚拟析构函数。

Mouse碰到Cheese时会调用Eaten方法，Cheese会重新出现在World中其他地方。

myWorld指向Cheese所在的任何World的指针，RandomLocation是World的方法，返回世界中的随机坐标。

定义Mouse类：

    class Mouse : public Animat
    {
    public:
        Mouse()
        {
            This.Add("angle", NearestAngleSensor<Cheese>());
            This.InitRandom = true;
        }
    virtual void Control()
        {
            double o = This.Sensors["angle"]->GetOutput();
            This.Controls["right"] = 0.5 - (o > 0.0 ? o : 0.0);
            This.Controls["left"] = 0.5 + (o < 0.0 ? o : 0.0);
        }
    virtual void OnCollision(WorldObject* obj)
        {
            Cheese* cheese;
    if (IsKindOf(obj,cheese)) {
                cheese->Eaten();
            }
    Animat::OnCollision(obj);
        }
    };


和先前一样，Mouse从Animat类中派生，但是这次只有一个sensor，也就是简单的NearestAngleSensor，它的范围实际上超出了World，并和往常一样告诉Mouse在World中随机的地方出现。

Control方法进行了简单的计算，让Mouse有期望中的行为：
右轮速度以0.5半速加上（本人认为此处应该是minus减去）与最近奶酪的夹角的向左幅度，
左轮以0.5半速加上夹角的向右幅度。

这里的OnCollision方法也是覆写的，当对象碰撞时被调用，参数是指向与Mouse碰撞的对象的指针，如果是Cheese就调用Eaten方法。

最后，当重写的OnCollision方法结束后，Animat的原生OnCollision方法仍会进行工作，比如避免Mouse穿过屏障或者其他Mouse，Animat::OnCollision会被调用，并传递指针。

Simulation的设置和之前差不多，但是这回将有两个Group对象：

    class MouseSimulation : public Simulation
    {
        Group<Mouse> theMice;
        Group<Cheese> theCheeses;
    public:
        MouseSimulation():
        theMice(30)
        theCheeses(50)
        {
            This.Add("Mice", theMice);
            This.Add("Cheeses", theCheeses);
        }
    };
    

宏：

    BEGIN_SIMULATION_TABLE
        ADD_SIMULATION("Mice", MouseSimulation)
    END_SIMULATION_TABLE


# 引入神经控制器（Neural Controllers）

模拟仿真提供了两种类型的人工神经网络（ANN）：FeedForwardNet，具有一个隐藏层的前馈网络；DynamicalNet，一个完全递归的动力学神经网络。

神经网络的配置：注意每个节点的输出将通过一个简单的阈值函数，而不是这种神经网络通常使用的sigmoid函数。输入节点上的两个权重将确保如果角度为正，则激活神经网络的一侧；反之，激活另一侧。与原始输出一样，两个输出节点偏置-0.5确保每个输出至少为0.5。

调整Mouse以使用此神经网络：

include前馈神经网络头文件：

    #include "feedforwardnet.h"

为FeedForwardNet添加私有成员：

    private:
        FeedForwardNet myBrain;

并且构造Mouse时得在myBrain中设置：

    Mouse():
    myBrain(1, 2, 2, false)
    {
        vector<float> ffnConfig;
        ffnConfig.push_back( 1.0f); // 1st input 1st weight 第一个输入的第一个权重
        ffnConfig.push_back( 0.0f); // 1st input bias 第一个输入偏置
        ffnConfig.push_back(-1.0f); // 2nd input 1st weight 第二个输入的第一个权重
        ffnConfig.push_back( 0.0f); // 2nd input bias 第二个输入的偏置
        ffnConfig.push_back( 1.0f); // 1st hidden 1st weight (for input 1) 第一个隐藏层的第一个权重（给输入1的）
        ffnConfig.push_back( 0.0f); // 1st hidden 2nd weight (for input 2) 第一个隐藏层的第二个输入权重（给输入2的）
        ffnConfig.push_back(-0.5f); // 1st hidden bias 第一个隐藏层的偏置
        ffnConfig.push_back( 0.0f); // 2nd hidden 1st weight (for input 1)  第二个隐藏层的第一个权重（给输入1的）
        ffnConfig.push_back( 1.0f); // 2nd hidden 2nd weight (for input 2) 第二个隐藏层的第二个权重（给输入2的）
        ffnConfig.push_back(-0.5f); // 2nd hidden bias 第二个隐藏层的偏置
    This.myBrain.SetConfiguration(ffnConfig); // set
    This.Add("angle", NearestAngleSensor<Cheese>());
        This.InitRandom = true;
    }

在这一行中myBrain(1, 2, 2, true)（此处应是false吧） 
中使用一个输入，两个隐藏层，两个输出配置内置的FeedForwardNet，false表示简单的阈值函数而不是sigmoid。

最后Mouse需要一种新的Control方法，该方法将传感器中的输入给输入到myBrain中，然后使用网络的输出设置motors。

    virtual void Control()
    {
        This.myBrain.SetInput(0, sensors["angle"]->GetOutput());
    This.myBrain.Fire();
    This.Controls["left"] = This.myBrain.GetOutput(0);
        This.Controls["right"] = This.myBrain.GetOutput(1);
    }

# 使Mouse更加挑剔

只要有cheese，mice就会继续吃，有些mice喜欢gruyere，不会碰其他的，而另一部分则喜欢普通的旧奶酪。

要达成这样的特性，首先，我们要通过继承cheese基础类来创建新cheese：

    class Stilton : public Cheese
    {
        Stilton()
        {
            This.SetColour(0.2f, 1.0f, 0.2f);
        }
        virtual ~Stilton(){}
    };
    class Gruyere : public Cheese
    {
        Gruyere()
        {
            This.SetColour(1.0f, 0.5f, 0.0f);
        }
        virtual ~Gruyere(){}
    };

除了颜色和类型，这两种奶酪和以前完全一样。

接下来，创建两种mouse：PickyMouse和OldFashionedMouse：

    class PickyMouse : public Mouse
    {
    public:
        virtual ~PickyMouse(){}
    virtual void OnCollision(WorldObject* obj)
        {
            Gruyere* cheese;
    if (IsA(obj, cheese)) {
                cheese->Eaten();
            }
    This.Animat::OnCollision(obj);
        }
    };
    class OldFashionedMouse : public Mouse
    {
    public:
        virtual ~OldFashionedMouse(){}
    virtual void OnCollision(WorldObject* obj)
        {
            Cheese* cheese;
    if (IsA(obj, cheese)) {
                cheese->Eaten();
            }
    This.Animat::OnCollision(obj);
        }
    };


# 修改传感器

现在需要修改传感器来检测特定类型的奶酪。

仿真中的传感器十分灵活，而每个传感器都有四个元素：

1. 传感器类，决定看向哪个个体：

	a.  basic Sensor class 将考虑整个World的所有个体
    
	b.  AreaSensor可以采用指定形状，并且只查看那个区域中的个体
    
	c.  TouchSensor只考虑接触的个体
BeamSensor从特定点投射一定角度范围和半径的射线，范围为0的激光只看前方，范围为2 x pi (360 degrees) 的激光将在任何方向检测

2. 匹配函数，决定对象是否对传感器感兴趣

3. 评估函数，获取有关对象的特定信息，比如说有多远，什么角度，多大，或者只保持了所感测物体的数量

4. 缩放函数，获取评估函数的输出并修改表示合适的输出


其实传感器函数不是真正的函数，而是函数对象，因此实际不存在NearestAngleSensor之类的，只是一堆附加在类上的函数对象。NearestAngleSensor就是在用正确的对象设置Sensor，并返回指向该Sensor的指针：

    template <class T>
    Sensor* NearestAngleSensor()
    {
        Sensor* s = new Sensor(Vector2D(0.0, 0.0), 0.0);
        s->SetMatchingFunction(new MatchKindOf<T>);
        s->SetEvaluationFunction(new EvalNearestAngle(s, 1000.0));
        s->SetScalingFunction(new ScaleLinear(-PI, PI, -1.0, 1.0));
    return s;
    }

因此要更改PickyMouse的匹配功能，只需要添加下行到PickyMouse的构造函数：

    sensors["cheese sensor"]->SetMatchingFunction(new MatchExact<Gruyere>);
    
这样旧的匹配功能将被删除然后被替代。


# BEAST教程索引

[BEAST 教程0 代码说明](https://yawwq.github.io/BEAST-%E6%95%99%E7%A8%8B0-%E4%BB%A3%E7%A0%81%E8%AF%B4%E6%98%8E/)

[BEAST 教程1 创建第一个Animat](https://yawwq.github.io/BEAST-%E6%95%99%E7%A8%8B1-%E5%88%9B%E5%BB%BA%E7%AC%AC%E4%B8%80%E4%B8%AAAnimat/)

[BEAST 教程2 添加Object和交互式Animat](https://yawwq.github.io/BEAST-%E6%95%99%E7%A8%8B2-%E6%B7%BB%E5%8A%A0Object%E5%92%8C%E4%BA%A4%E4%BA%92%E5%BC%8FAnimat/)

[BEAST 教程3 引入遗传算法](https://yawwq.github.io/BEAST-%E6%95%99%E7%A8%8B3-%E5%BC%95%E5%85%A5%E9%81%97%E4%BC%A0%E7%AE%97%E6%B3%95/)

# Reference

https://minerva.leeds.ac.uk/webapps/blackboard/content/listContent.jsp?course_id=_505438_1&content_id=_6865832_1&mode=reset
