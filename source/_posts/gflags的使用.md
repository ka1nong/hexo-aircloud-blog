---
title: gflags的使用
date: 2018-08-07 20:45:17
tags:
---
<h2>简介</h2>

google开源的gflags是一套命令行参数解析工具，比getopt功能更强大，使用起来更加方便，gflags还支持从环境变量、配置文件读取参数（可用gflags代替配置文件）。我们在程序中有默认的指定参数，同时希望可以通过命令行来指定不同的值,那么使用gflags是最好不过的了。

参考：
https://blog.csdn.net/cn_wk/article/details/61198182
https://blog.csdn.net/jcjc918/article/details/50876613
<h2>头文件</h2>

使用flags需要包含头文件
	
    ```c++
    #include <gflags/gflags.h>
    ```

<h2>支持的参数类型</h2>

gflags主要支持的参数类型如下几种:
* DEFINE_bool: 布尔类型
* DEFINE_int32: 32 位整数
* DEFINE_int64: 64 位整数
* DEFINE_uint64: 无符号 64 位整数
* DEFINE_double: 浮点类型 double
* DEFINE_string: C++ string 类型

<h2>参数定义</h2>

定义参数通过DEFINE_type宏实现，如下所示，分别定义了一个bool和一个string类型的参数，该宏的三个参数含义分别为命令行参数名，参数默认值，以及参数的帮助信息。
	```c++
    DEFINE_bool(big_menu, true, "Include 'advanced' options in the menu listing"); 
    DEFINE_string(languages, "english,french,german", "comma-separated list of languages to offer in the 'lang' menu"); 
    ```
    
gflag不支持列表，用户通过灵活借助string参数实现，比如上述的languages参数，可以类型为string，但可看作是以逗号分割的参数列表。


<h2>参数访问</h2>

当参数被定义后，通过FLAGS_name就可访问到对应的参数，比如上述定义的big_menu、languages两个参数就可通过FLAGS_big_menu、FLAGS_languages访问。
以上的访问方式，仅在参数定义和访问在同一个文件（或是通过头文件包含）时，FLAGS_name才能访问到参数，如果要访问其他文件里定义的参数，则需要使用DECLARE_type。

	```c++
    比如在foo.cc中
    EFINE_string(color, "red", "the color you want to use"); 
    这是如果你需要在foo_test.cc中使用color这个参数，你需要加入
    DECLARE_string(color);
    ```
    
<h2>规范</h2>

使用 DECLARE_，它的作用就相当于用 extern 声明变量。为了方便的管理变量，我们推荐在 .cc 或者 .cpp 文件中 DEFINE 变量，然后只在对应 .h 中或者单元测试中 DECLARE 变量。例如，在 foo.cc 定义了一个 gflags 变量 DEFINE_string(name, 'bob', '')，假如你需要在其他文件中使用该变量，那么在 foo.h 中声明 DECLARE_string(name)，然后在使用该变量的文件中 include "foo.h" 就可以。当然，这只是为了更好地管理文件关联，如果你不想遵循也是可以的。

<h2>参数检查</h2>

如果你定义的 gflags 参数很重要，希望检查其值是否符合预期，那么可以定义并注册参数的值的检查函数。如果采用 static 全局变量来确保检查函数会在 main 开始时被注册，可以保证注册会在 ParseCommandLineFlags 函数之前。如果默认值检查失败，那么 ParseCommandLineFlags 将会使程序退出。如果之后使用 SetCommandLineOption() 来改变参数的值，那么检查函数也会被调用，但是如果验证失败，只会返回 false，然后参数保持原来的值，程序不会结束。看下面的程序示例：
	
    ```c++
 
    #include <stdint.h>
    #include <stdio.h>
    #include <iostream>
    #include <gflags/gflags.h>

    // 定义对 FLAGS_port 的检查函数
    static bool ValidatePort(const char* name, int32_t value) {
        if (value > 0 && value < 32768) {
            return true;
        }
        printf("Invalid value for --%s: %d\n", name, (int)value);
        return false;
    }

    /**
     *  设置命令行参数变量
     *  默认的主机地址为 127.0.0.1，变量解释为 'the server host'
     *  默认的端口为 12306，变量解释为 'the server port'
     */
    DEFINE_string(host, "127.0.0.1", "the server host");
    DEFINE_int32(port, 12306, "the server port");

    // 使用全局 static 变量来注册函数，static 变量会在 main 函数开始时就调用
    static const bool port_dummy = gflags::RegisterFlagValidator(&FLAGS_port, &ValidatePort);

    int main(int argc, char** argv) {
        // 解析命令行参数，一般都放在 main 函数中开始位置
        gflags::ParseCommandLineFlags(&argc, &argv, true);
        std::cout << "The server host is: " << FLAGS_host
            << ", the server port is: " << FLAGS_port << std::endl;

        // 使用 SetCommandLineOption 函数对参数进行设置才会调用检查函数
        gflags::SetCommandLineOption("port", "-2");
        std::cout << "The server host is: " << FLAGS_host
            << ", the server port is: " << FLAGS_port << std::endl;
        return 0;
    }
    ```

测试：
	
    ```c++
    #命令行指定非法值，程序解析参数时直接退出
    ➜  test ./gflags_test -port -2 
    Invalid value for --port: -2
    ERROR: failed validation of new value '-2' for flag 'port'
    # 这里参数默认值合法，但是 SetCommandLineOption 指定的值不合法，程序不退出，参数保持原来的值 
    ➜  test ./gflags_test        
    The server host is: 127.0.0.1, the server port is: 12306
    Invalid value for --port: -2
    The server host is: 127.0.0.1, the server port is: 12306
    ```

建议在定义参数后，立即注册检查函数。RegisterFlagValidator()在检查函数注册成功时返回true；如果参数已经注册了检查函数，或者检查函数类型不匹配，返回false。

<h2>初始化参数</h2>

在引用程序的main()里通过 google::ParseCommandLineFlags(&argc, &argv, true); 即完成对gflags参数的初始，其中第三个参数为remove_flag，如果为true，gflags会移除parse过的参数，否则gflags就会保留这些参数，但可能会对参数顺序进行调整。 比如 "/bin/foo" "arg1" "-q" "arg2"  会被调整为 "/bin/foo", "-q", "arg1", "arg2"，这样更好理解。

<h2>命令行指定参数</h2>

比如要在命令行指定languages参数的值，可通过如下4种方式，int32, int64等类型与string类似
* app_containing_foo --languages="chinese,japanese,korean"
* app_containing_foo -languages="chinese,japanese,korean"
* app_containing_foo --languages "chinese,japanese,korean"
* app_containing_foo -languages "chinese,japanese,korean"

对于bool类型，则可通过如下几种方式指定参数，
* ./gflags_test -debug_switch  # 这样就是 true
* ./gflags_test -debug_switch=true # 这样也是 true
* ./gflags_test -debug_switch=1 # 这样也是 true
* ./gflags_test -debug_switch=false # 0 也是 false

特殊参数
* --help 打印定义过的所有参数的帮助信息
* --version 打印版本信息 通过google::SetVersionString()指定
* --nodefok  但命令行中出现没有定义的参数时，并不退出（error-exit）
* --fromenv 从环境变量读取参数值 --fromenv=foo,bar表明要从环境变量读取foo，bar两个参数的值。通过export FLAGS_foo=xxx; export FLAGS_bar=yyy 程序就可读到foo，bar的值分别为xxx，yyy。
* --tryfromenv 与--fromenv类似，当参数的没有在环境变量定义时，不退出（fatal-exit）
* --flagfile 从文件读取参数值，--flagfile=my.conf表明要从my.conf文件读取参数的值。在配置文件中指定参数值与在命令行方式类似，另外在flagfile里可进一步通过--flagfile来包含其他的文件。    


<h2>使用 flagfile</h2>

如果我们定义了很多参数，那么每次启动时都在命令行指定对应的参数显然是不合理的。gflags库已经很好的解决了这个问题。你可以把 flag 参数和对应的值写在文件中，然后运行时使用 -flagfile 来指定对应的 flag 文件就好。文件中的参数定义格式与通过命令行指定是一样的。
例如，我们可以定义这样一个文件，文件后缀名没有关系，为了方便管理可以使用 .flags：

	```c++
    --host=10.123.14.11
    --port=23333
    ```
    
然后命令行指定：
	```c++
    ➜  test ./gflags_test --flagfile server.flags 
	The server host is: 10.123.14.11, the server port is: 23333
    ```
    
<h2>使用示例</h2>

	```c++
    //main.cpp
    #include <gflags/gflags.h>
    #include <iostream>

    typedef int int32;

    DEFINE_bool(big_menu, true, "Include 'advanced' options in the menu listing");
    DEFINE_string(languages, "english,french,german", "comma-separated list of languages to offer in the 'lang' menu");
    DEFINE_int32(port, 0, "What port to listen on");

    static bool ValidatePort(const char* flagname, int32 value)
    {
        if (value > 0 && value < 32768)   // value is ok 
            return true;

        std::cout << "error: flagn=" << flagname << "  value:" << value << std::endl;
        return false;
    }

    static const bool port_dummy = google::RegisterFlagValidator(&FLAGS_port, &ValidatePort);

    int main(int argc, char* argv[])
    {
        google::ParseCommandLineFlags(&argc, &argv, true);
        std::cout << "big menu value is:" << FLAGS_big_menu << std::endl;
        std::cout << "languages value is" << FLAGS_languages << std::endl;
        return 0;
    }

    ```
编译命令：

	```c++
    > g++ main.cpp -o main -lgflags
    ```

输出：
	```c++
    > ./main --languages="chennhuang test" --port=0  // port不可以等于0
    error: flagn=port  value:0
    error: flagn=port  value:0
    ERROR: failed validation of new value '0' for flag 'port'
    
    > ./main --languages="chennhuang test" --port=1
    big menu value is:1
    languages value ischennhuang tes
    ```
