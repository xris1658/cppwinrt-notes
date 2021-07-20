# AudioGraph

突然发现我有好长时间没碰这个了。

这次代码实现的功能是选择音频输出设备。当电脑上同时接入多个输出设备时，可以选择一个输出设备进行输出。

## 示例
这次的示例中，我们对之前输出正弦波的程序代码进行修改。这里只写出部分代码，用于显示添加代码的位置：
```cpp
// 预编译头文件：pch.h
// ...
#include <winrt/Windows.Foundation.h>
#include <winrt/Windows.Foundation.Collections.h> // 新增的头文件
#include <winrt/Windows.Media.h>
#include <winrt/Windows.Media.Audio.h>
#include <winrt/Windows.Media.MediaProperties.h>
#include <winrt/Windows.Devices.Enumeration.h> // 新增的头文件
```
```cpp
// 文件：main.cpp
// ...
using namespace winrt;
using namespace winrt::Windows::Foundation;
using namespace winrt::Windows::Foundation::Collections; // 新增的 using
using namespace winrt::Windows::Media;
using namespace winrt::Windows::Media::Audio;
using namespace winrt::Windows::Media::MediaProperties;
using namespace winrt::Windows::Devices::Enumeration; // 新增的 using
// ...
int main()
{
    // ...
    auto asyncAudioOutputList = DeviceInformation::FindAllAsync(DeviceClass::AudioRender);
    auto audioOutputList = asyncAudioOutputList.get();
    std::wcout << L"共找到" << audioOutputList.Size() << L"个音频输出设备：" << std::endl;
    for (std::uint32_t i = 0; i < audioOutputList.Size(); ++i)
    {
        std::wcout << i + 1 << L": " << audioOutputList.GetAt(i).Name().c_str() << std::endl;
    }
    std::wcout << L"请选择要使用的音频输出设备：";
    std::uint32_t deviceIndex = 0;
    do
    {
        std::wcin >> deviceIndex;
        if (deviceIndex != 0 && deviceIndex <= audioOutputList.Size())
        {
            break;
        }
        std::wcout << L"输入错误！请重新输入：";
    } while (1);
    std::wcout << L"选择的音频输出设备：" << audioOutputList.GetAt(deviceIndex - 1).Name().c_str() << std::endl;
    AudioGraphSettings settings(Render::AudioRenderCategory::Media);
    settings.PrimaryRenderDevice(audioOutputList.GetAt(deviceIndex - 1)); // 新增
    // ...
}
```
讲几个关键点：
- 新增了头文件 `winrt/Windows.Devices.Enumeration.h`，其中包含了设备相关功能：`DeviceInformation` 用于访问设备信息，`DeviceInformationCollection` 是 `DeviceInformation` 的容器。
- 新增了头文件 `winrt/Windows.Foundation.Collections.h`，其中包含了`DeviceInformationCollection` 的基类模板 `IVectorView` 和迭代器类型接口模板 `IIterable` 的声明。如果不添加此头文件，将会出现编译错误：
    ```
    main.cpp(48,50): error C3779: 'winrt::impl::consume_Windows_Foundation_Collections_IVectorView<winrt::Windows::Foundation::Collections::IVectorView<winrt::Windows::Devices::Enumeration::DeviceInformation>,T>::Size': a function that returns 'auto' cannot be used before it is defined
    ```
    较旧的编译器可能会出现链接错误。

    为什么会出现错误？因为这些函数的前向声明中使用的返回类型为 `auto`，且没有函数体或者**尾置返回类型（trailing return type）**。换言之，这个函数的返回类型需要编译器进行推导，但是这个声明不提供任何关于返回类型的信息。之后函数被调用时，函数实现还没出现，导致编译器无法推断返回类型。

    参考：[Why does my C++/WinRT project get errors of the form “consume_Something: function that returns ‘auto’ cannot be used before it is defined”? | The Old New Thing](https://devblogs.microsoft.com/oldnewthing/20190530-00/?p=102529)

    什么是尾置返回类型？这是 C++11 中新增的声明方式，用于为返回类型*为 `auto`* 或*包含 `auto`* 的函数标明返回类型。
    ```cpp
    auto add(auto a, auto b) -> decltype(a + b); // 返回类型为表达式 a+b 结果的类型
    ```
    如此一来，函数的返回类型将在声明时得知，方便进行编译。  
    读者可以参阅 [EMCPP 的条款 3（非官方中译）](https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/1.DeducingTypes/item3.md)。
- 关于头文件的包含次序，笔者选择将 `winrt/Windows.Foundation.Collections.h` 放在 `winrt/Windows.Devices.Enumeration.h` 之前，但实际上放在后面也没问题。

## 惯用法：CRTP
下面是 `DeviceInformation` 类的声明：
```cpp
struct __declspec(empty_bases) DeviceInformation : winrt::Windows::Devices::Enumeration::IDeviceInformation,
impl::require<
    DeviceInformation, // 派生类本身
    winrt::Windows::Devices::Enumeration::IDeviceInformation2>
```
这种将派生类作为基类的模板参数的编程惯用法称为 **Curiously Recurring Template Pattern（CRTP）**，中文翻译成**奇异递归模板模式**或**奇特重现模板模式**，常用于实现静态多态。