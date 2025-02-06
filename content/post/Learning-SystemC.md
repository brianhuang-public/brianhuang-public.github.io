---
date: '2025-01-27T18:27:33+08:00'
draft: false
title: 'Learning System C basic in one page'
---


# Intro

SystemC用于硬件仿真的cmodel模型建立，相比传统C语言的cmodel模型，有以下好处。
- 可以实现精确至cycle级的仿真模型

# Note

本文是基于https://www.learnsystemc.com/ 整理的笔记，大部分内容该网站都很详细了，在此之上，对于其中的细节部分进行整理。

# sc module

`SC_MODULE`该define用于声明module，对于每个module，本质上就是C++的类，需要使用构造函数进行声明，声明方法如下：

- 使用`SC_CTOR` marco，该marco会生成一个默认的构造函数，该构造函数不支持多参数的传入。
- 

```
// Learn with Examples, 2020, MIT license
#include <systemc>
using namespace sc_core;

SC_MODULE(MODULE_A) { // approach #1, use systemC provided SC_MODULE macro
  SC_CTOR(MODULE_A) { // default constructor
    std::cout << name() << " constructor" << std::endl; // name() returns the object name, which is provided during instantization
  }
};
struct MODULE_B : public sc_module { // approach #2, this uses c++ syntax and is more readiable
  SC_CTOR(MODULE_B) {
    std::cout << name() << " constructor" << std::endl;
  }
};
class MODULE_C : public sc_module { // approach #3, use class instead of struct
public: // have to explicitly declare constructor function as public 
  SC_CTOR(MODULE_C) {
    std::cout << name() << " constructor" << std::endl;
  }
};

int sc_main(int, char*[]) { // systemC entry point
  MODULE_A module_a("module_a"); // declare and instantiate module_a, it's common practice to assign module name == object name
  MODULE_B module_b("modb"); // declare and instantiate module_b, module name != object name
  MODULE_C module_c("module_c"); // declare and instantiate module_c
  sc_start(); // this can be skipped in this example because module instantiation happens during elaboration phase which is before sc_start
  return 0;
}
```

## SC_HAS_PROCESSED
`SC_HAS_PROCESSED` 与`SC_CTOR`的区别：
- 这两个marco中，`SC_HAS_PROCESSED`不会有默认构造函数的声明，`SC_CTOR`是有默认构造函数的声明的。

因此使用`SC_HAS_PROCESSED` 时，需要显示声明构造函数。

```
SC_MODULE(MODULE_C) { // pass additional input argument(s)
  const int i;
  SC_HAS_PROCESS(MODULE_C); // OK to use SC_CTOR, which will also define an un-used constructor: MODULE_A(sc_module_name);
  MODULE_C(sc_module_name name, int i) : i(i) { // define the constructor function
    SC_METHOD(func_c); // register member function
  }
  void func_c() { // define function
    std::cout << name() << ", additional input argument" << std::endl;
  }
};

```

## Simulation process(SC_METHOD SC_THREAD SC_CTHREAD)

sc module中可以定义`SC_METHOD SC_THREAD SC_CTHREAD`的仿真进程，相当于面向对象编程中的方法。这三个的区别是：
- SC_METHOD：不耗时，类似systemverilog中的function，在method的内部不能使用wait()，即非阻塞的。
- SC_THREAD：耗时，可被阻塞。
- SC_CTHREAD：只能被时钟事件 / 静态事件驱动。

SC_THREAD中既能实现SC_METHOD的功能也能实现SC_CTHREAD的功能。
SC_CTHREAD shall not be invoked from the end_of_elaboration callback.

## Simulation stages

与UVM component的phase机制类似，sc module也有类似的回调机制。具体顺序如下：

- 构造函数调用
- before_end_of_elaboration()
- end_of_elaboration()
- start_of_simulation()
- sc_stop() (not callback, users call this)
- end_of_simulation()
- deconsturctor() 

## timing unit
SC_SEC, SC_MS, SC_US, SC_NS, SC_PS, SC_FS

时间对象：sc_time

如何获取仿真时间和比较仿真时间：
`sc_time_stamp() == sc_time(4, SC_SEC)`
其中sc_time_stamp()会返回一个当前仿真时间的sc_time对象。sc_time(4, SC_SEC)会创建一个4 SC_SEC的时间对象，仿真时间的比较实际上是sc_time对象的比较。

## Event

sc_event 支持如下方法：
  1. void notify(): create an immediate notification (无延时)
  2. void notify(const sc_time&), void notify(double, sc_time_unit):
    - zero time: create a delta notification.
    - non-zero time: create a timed notification at the given time, expressed relative to the simulation time when function notify is called
    -  **notify with time这个函数不会阻塞进程，而是继续执行，例如t = t0触发notify(4, SC_SEC)，那么会继续执行后续方法，在t0+4时刻触发该事件。**
  3. cancel(): delete any pending notification for this event
    - At most one pending notification can exist for any given event. 每个事件最多只能有一个notification在pending
    - Immediate notification cannot be cancelled. 无延时的notify()不能被cancel。
  4. 等待事件wait(event);

## Combined event

`wait(200, SC_NS, e1 | e2 | e3)`

前半部分约束了timeout时刻，200ns 后timeout；后半部分约束了等待的事件(可以使用 | or & 对事件进行逻辑约束)

## Delta cycle

与硬件仿真一致，SC中也有delta cycle的概念。can be thought of as a very small step of time within the simulation, which does not increase the user-visible time.

When is delta cycle used:
  1. notify(SC_ZERO_TIME) causes the event to be notified in the evaluate phase of the next delta cycle, this is called a "delta notification".
  2. A (direct or indirect) call to request_update() causes the update() method to be called in the update phase of the current delta cycle. 
  3. wait(SC_ZERO_TIME); 仿真推进1个delta cycle

## Sensitivity

主要区分静态敏感性和动态敏感性。

### 静态敏感性

静态敏感性是在进程声明时绑定到特定事件的敏感性。这种绑定在模拟启动前完成，进程会对这些绑定的事件保持响应。

- 绑定关系在设计编译时静态定义，不会动态更改。
- 常用于SC_METHOD和SC_THREAD。
- 使用SC_METHOD或SC_THREAD时，通过sensitive关键字指定敏感事件列表。

静态敏感性只能使用sensitive关键字绑定事件，不能使用类似wait(e1|e2)这种格式，绑定时间是在编译时确定的。

### 动态敏感性
可以使用wait来动态指定事件。

## Initialization

SystemC仿真的初始化阶段负责完成以下动作，以确保模型从正确的状态开始运行：

1.  模块实例化
	-	创建所有模块的实例，并调用其构造函数。
	-	在构造函数中定义进程（SC_METHOD、SC_THREAD或SC_CTHREAD）并设置其敏感性。

2.  进程静态敏感性绑定
	-	将进程与敏感信号或事件绑定。例如，通过sensitive关键字，将进程与信号的边沿或事件关联。

3.  初次触发检查
	-	检查每个进程是否需要初次触发：
	-	默认：在仿真启动时触发一次每个定义的SC_METHOD或SC_THREAD。
	-	如果调用了dont_initialize，则跳过该进程的初次触发。

4.  信号初始化
	- 初始化所有sc_signal的值。如果未显式赋值，则使用默认值。
	- 确保初始值在关联的模块中可以被正确读取。

```
SC_MODULE(my_module) {
    sc_in<bool> clk;    // 输入时钟
    sc_signal<int> data; // 数据信号

    SC_CTOR(my_module) {
        SC_METHOD(my_process);
        sensitive << clk.pos();
        // dont_initialize(); // 如果需要禁止初次触发，可以取消注释
    }

    void my_process() {
        std::cout << "Triggered at time: " << sc_time_stamp() << ", data = " << data.read() << std::endl;
    }
};

int sc_main(int argc, char* argv[]) {
    sc_clock clk("clk", 10, SC_NS); // 创建时钟
    sc_signal<int> data;

    my_module uut("uut"); // 实例化模块
    uut.clk(clk);         // 绑定时钟信号

    sc_start(1, SC_NS);   // 启动仿真

    return 0;
}
```


## Process: Method

### next_trigger