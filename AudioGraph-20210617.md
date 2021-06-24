# AudioGraph
## 写在前面
本文档记录了一部分 AudioGraph 的基本使用。  
如果想使用 C++ 进行开发，但对 C++/WinRT 不了解，可以先阅读笔者之前的笔记，然后回头阅读此笔记。  
如果想使用 C# 进行开发，则不应阅读此笔记，微软的文档和其他网络会提供需要的信息。  

## 介绍
AudioGraph 是微软的 WinRT API 中提供的低延迟音频 API。  
<a href="https://docs.microsoft.com/zh-cn/windows-hardware/drivers/audio/low-latency-audio">微软文档对低延迟音频 API 的介绍</a>

### 工作原理
AudioGraph 使用基于节点的工作流程。如同名字一样，AudioGraph 类似一个图，其中包含许多节点，以及节点之间的连接关系。

## 示例
### 编写一个播放音频文件的控制台应用
此应用读取并播放一个音频文件。

#### 运行过程
0. 初始化。
1. 创建一个 `AudioGraph` 对象 `graph`。
2. 创建一个 `AudioDeviceOutputNode` 对象 `output`，用于将音频信号输出到音频设备。
3. 创建一个 `AudioFileInputNode` 对象 `input`，用于从音频文件读取音频信号。
4. 将 `input` 与 `output` 相连。
5. 启动 `graph`，工作一段时间后，停止 `graph`。

#### 代码
代码如下：
```cpp
// 文件：pch.h
#pragma once

#include <winrt/Windows.Foundation.h>
#include <winrt/Windows.Media.h>
#include <winrt/Windows.Media.Audio.h>
#include <winrt/Windows.Storage.Pickers.h>
```
```cpp
// 文件：main.cpp
#include "pch.h"

using namespace winrt;
using namespace Windows::Foundation;
using namespace Windows::Media;             // 多媒体功能
using namespace Windows::Media::Audio;      // 多媒体中的音频功能
using namespace Windows::Storage::Pickers;  // 文件功能

#include <iostream>
#include <thread>
#include <chrono>
#include <type_traits>

int main()
{
    // 0
    init_apartment();
    std::ios::sync_with_stdio(false);
    std::locale::global(std::locale(""));
    std::wcout.imbue(std::locale(""));
    std::wcin.imbue(std::locale(""));

    // 1
    AudioGraphSettings settings(Render::AudioRenderCategory::Media);
    auto asyncInitResult = AudioGraph::CreateAsync(settings);
    auto initResult = asyncInitResult.get();
    if(initResult.Status() != decltype(initResult.Status())::Success) // 状态本身是限域枚举类型，我们不关心具体类型名，且类型名较长，所以使用 decltype 进行类型推导。
    {
        std::wcout << L"初始化异步操作失败，错误代码：" << static_cast<std::underlying_type_t<decltype(initResult.Status())>>(initResult.Status()) << std::endl; // 输出错误代码。状态本身是限域枚举类型，所以需要显式转换为其底层类型，且使用 decltype。
        return 0;
    }
    AudioGraph graph = initResult.Graph();
    std::wcout << L"初始化异步操作成功" << std::endl;

    // 2
    auto asyncOutputResult = graph.CreateDeviceOutputNodeAsync();
    auto outputResult = asyncOutputResult.get();
    if(outputResult.Status() != decltype(outputResult.Status())::Success)
    {
        std::wcout << L"初始化输出失败，错误代码：" << static_cast<std::underlying_type_t<decltype(outputResult.Status())>>(outputResult.Status()) << std::endl;
        return 0;
    }
    auto output = outputResult.DeviceOutputNode();
    std::wcout << L"初始化输出成功" << std::endl;

    // 3
    std::wcout << L"输入文件：";
    wchar_t charBuffer[1024];
    std::wcin.getline(charBuffer, 1024);
    auto state = std::wcin.rdstate();
    if(state != std::ios_base::goodbit)
    {
        std::wcout << L"读取文件路径失败！错误代码：" << state << std::endl;
        return 0;
    }
    std::wstring filePath(charBuffer);
    if(filePath.front() == L'\"')
    {
        filePath.erase(0, 1);
        filePath.erase(filePath.length() - 1);
    }
    auto asyncFile = Windows::Storage::StorageFile::GetFileFromPathAsync(filePath.c_str());
    auto file = asyncFile.get();
    auto asyncInputResult = graph.CreateFileInputNodeAsync(file);
    auto inputResult = asyncInputResult.get();
    if(inputResult.Status() != AudioFileNodeCreationStatus::Success)
    {
        std::wcout << L"初始化输入失败，错误代码：" << static_cast<std::underlying_type_t<decltype(inputResult.Status())>>(inputResult.Status()) << std::endl;
        return 0;
    }
    AudioFileInputNode input = inputResult.FileInputNode();
    std::wcout << L"初始化输入成功" << std::endl;
    std::wcout << L"路径：" << filePath.c_str() << std::endl;
    std::wcout << L"文件类型：";
    std::wcout << file.FileType().c_str() << std::endl;
    std::wcout << L"持续时间：";
    auto inputDuration = input.Duration();
    std::wcout << inputDuration << std::endl;  // C++20

    // 4
    input.AddOutgoingConnection(output);

    // 5
    graph.Start();
    std::wcout << L"正在播放..." << std::endl;
    std::this_thread::sleep_for(inputDuration);
    graph.Stop();
    std::wcout << L"播放结束" << std::endl;
    return 0;
}
```
讲几个关键点：
- 初始化时调整本地化设置，否则无法正常读取和输出宽字符。
- 大部分操作是异步的，我们进行这些操作之后需要阻塞主线程，等待操作完成后获取结果。
以步骤 1 为例：
    ```cpp
    // 1（显式指定部分类型）
    IASyncOperation<CreateAudioGraphResult> asyncInitResult = AudioGraph::CreateAsync(settings);
    CreateAudioGraphResult initResult = asyncInitResult.get();
    if(initResult.Status() != decltype(initResult.Status())::Success)
    {
        std::wcout << L"初始化异步操作失败，错误代码：" << static_cast<std::underlying_type_t<decltype(initResult.Status())>>(initResult.Status()) << std::endl;
        return 0;
    }
    AudioGraph graph = initResult.Graph();
    std::wcout << L"初始化异步操作成功" << std::endl;
    ```
    第一行的异步操作返回一个 `IAsyncOperation<TResult>`，其中 `TResult` 为结果类型。然后第二行使用 `get()` 获取结果。  
    如果读者经常使用标准库中的异步操作，可以发现，这种工作方式和标准库中异步操作的使用方式类似，只是将异步操作名称换了一下，将返回类型从 `std::future<TResult>` 换成了 `IAsyncOperation<TResult>`。

    *如果读者之前没有使用过异步操作，但比较熟悉多线程，可参阅 EMCPP 的 <a href="https://github.com/kelthuzadx/EffectiveModernCppChinese/blob/master/7.TheConcurrencyAPI/Item35.md">条款 35（非官方中译）</a> 。*

- 读取文件路径时使用 `getline`，而不是 `operator>>`。这用于处理路径带空格等字符的情况。
- 读取文件路径时会检测首尾是否有双引号。使用时可以直接将文件拖至控制台窗口中。
- `inputDuration` 的类型是`winrt::Windows::Foundation::TimeSpan`，它是标准库中 `std::chrono::duration` 的特化。
- `std::chrono::duration` 的输出操作是 C++20 中添加的内容，所以要在项目属性的“配置属性”→“常规”中，将“C++ 语言标准”调整为“预览 - 最新 C++ 工作草案中的功能 (/std:c++latest)”。
- 使用 C++/WinRT 时，编译操作会生成非常大的中间文件（Debug x64 配置下，大小约为 435MB），其中包含一千多个头文件。建议将中间目录放在固态硬盘等读写速度较快的存储介质上，这可以节省大量的时间。

### 编写一个输出音频信号的控制台应用
此应用生成 440Hz 的正弦波，并播放 2 秒。

#### 运行过程
0. 初始化。
1. 创建一个 `AudioGraph` 对象 `graph`。
2. 创建一个 `AudioDeviceOutputNode` 对象 `output`，用于将音频信号输出到音频设备。
3. 创建一个 `AudioFrameInputNode` 对象 `frameInput`，用于从内存中的音频缓冲区读取音频信号。
4. 将 `frameInput` 与 `output` 相连。
5. 为 `frameInput` 添加量子开始事件。
6. 启动 `graph`后，紧接着启动 `frameInput`，工作 2s 后，停止 `frameInput`，紧接着停止 `graph`。

#### 代码
代码如下：
```cpp
// 预编译头文件：pch.h
#pragma once

#include <Unknwn.h> // IUnknown 接口的定义位置。

#include <winrt/Windows.Foundation.h>
#include <winrt/Windows.Media.h>
#include <winrt/Windows.Media.Audio.h>
#include <winrt/Windows.Media.MediaProperties.h>
```
```cpp
// 文件：main.cpp
#include "pch.h"

#include <cstdint>

struct __declspec(uuid("5B0D3235-4DBA-4D44-865E-8F1D0E4FD04D")) __declspec(novtable) IMemoryBufferByteAccess: ::IUnknown
{
    virtual HRESULT __stdcall GetBuffer(std::uint8_t** value, std::uint32_t* capacity) = 0;
};

using namespace winrt;
using namespace winrt::Windows::Foundation;
using namespace winrt::Windows::Media;
using namespace winrt::Windows::Media::Audio;
using namespace winrt::Windows::Media::MediaProperties;

#include <cmath>
#include <iostream>
#include <thread>
#include <chrono>

double theta = 0.0;
std::uint64_t outputSampleCount = 0U;

void frameInputQuantumStarted(const AudioFrameInputNode&, const FrameInputNodeQuantumStartedEventArgs&);

int main()
{
    init_apartment();
    std::ios::sync_with_stdio(false);
    std::locale::global(std::locale(""));
    std::wcout.imbue(std::locale(""));
    std::wcin.imbue(std::locale(""));
    AudioGraphSettings settings(Render::AudioRenderCategory::Media);
    auto asyncInitResult = AudioGraph::CreateAsync(settings);
    auto initResult = asyncInitResult.get();
    if(initResult.Status() != decltype(initResult.Status())::Success)
    {
        std::wcout << L"初始化异步操作失败，错误代码：" << static_cast<std::underlying_type_t<decltype(initResult.Status())>>(initResult.Status()) << std::endl;
        return 0;
    }
    std::wcout << L"初始化异步操作成功" << std::endl;
    AudioGraph graph = initResult.Graph();
    auto asyncOutputResult = graph.CreateDeviceOutputNodeAsync();
    auto outputResult = asyncOutputResult.get();
    if(outputResult.Status() != decltype(outputResult.Status())::Success)
    {
        std::wcout << L"初始化输出失败，错误代码：" << static_cast<std::underlying_type_t<decltype(outputResult.Status())>>(outputResult.Status()) << std::endl;
        return 0;
    }
    auto output = outputResult.DeviceOutputNode();
    std::wcout << L"初始化输出成功" << std::endl;
    AudioEncodingProperties encodingProperties = graph.EncodingProperties();
    std::wcout << L"采样率：" << encodingProperties.SampleRate() << L"Hz" << std::endl;
    std::wcout << L"位深度：" << encodingProperties.BitsPerSample() << L"bit" << std::endl;
    std::wcout << L"声道数：" << encodingProperties.ChannelCount() << std::endl;
    auto frameInput = graph.CreateFrameInputNode(encodingProperties);
    frameInput.AddOutgoingConnection(output);
    frameInput.Stop();
    frameInput.QuantumStarted(frameInputQuantumStarted);
    std::wcout << L"正在播放..." << std::endl;
    graph.Start();
    frameInput.Start();
    std::this_thread::sleep_for(std::chrono::seconds(2));
    frameInput.Stop();
    graph.Stop();
    std::wcout << outputSampleCount << std::endl;
    return 0;
}

void frameInputQuantumStarted(const AudioFrameInputNode& sender, const FrameInputNodeQuantumStartedEventArgs& args)
{
    if(auto requiredSamples = args.RequiredSamples(); requiredSamples != 0)
    {
        auto properties = sender.EncodingProperties();
        auto bufferSize = requiredSamples * properties.BitsPerSample() * properties.ChannelCount();
        AudioFrame frame(bufferSize >> 3);
        {
            AudioBuffer buffer = frame.LockBuffer(AudioBufferAccessMode::Write);
            IMemoryBufferReference reference = buffer.CreateReference();
            std::uint8_t* dataInBytes = nullptr;
            std::uint32_t capacityInBytes = 0U;
            auto byteAccess = reference.as<IMemoryBufferByteAccess>();
            check_hresult(byteAccess->GetBuffer(&dataInBytes, &capacityInBytes));
            float* dataInFloat = reinterpret_cast<float*>(dataInBytes);
            double freq = 440.0;
            double amplitude = 0.5;
            auto sampleRate = properties.SampleRate();
            double sampleIncrement = (freq * std::acos(-1) * 2.0) / sampleRate;
            for(int i = 0; i < requiredSamples; ++i)
            {
                double value = amplitude * std::sin(theta);
                dataInFloat[i * properties.ChannelCount()] = static_cast<float>(value);
                dataInFloat[i * properties.ChannelCount() + 1] = static_cast<float>(value);
                theta += sampleIncrement;
                ++outputSampleCount;
            }
        }
        sender.AddFrame(frame);
    }
}
```
#### 说明
这个示例中多了不少东西，这里说明一下：
- COM 类 `IMemoryBufferByteAccess` 用于将缓冲区等转换为指向内存指针。继承的类为 `IUnknown`。注意此处的 `IUnknown` 为 `Unknwn.h` 中的类，而不是 `winrt::Windows::Foundation` 中的类。
- 由于 COM 的出现，**一定要正确安排代码的次序**。必须先包含 `Unknwn.h` ，然后再包含 WinRT 头文件；必须先写 `IMemoryBufferByteAccess` 的定义，然后再使用 WinRT 的命名空间。做错任何一个都会出现 `IUnknown` 类重定义（或者 `IUnknown` 不明确）的编译错误。
- 在缓冲区中，多个声道的采样使用交错（interleaving）的方式存储（例：`LRLRLRLR`），使用这种存储方式方便回放。另一种存储方式是非交错（deinterleaving）（例：`LLLLRRRR`），使用这种存储方式方便进行信号处理（如傅里叶变换）。
- 缓冲区存放的数据类型为 32 位单精度浮点（`float`），不是 32 位有符号整型（`std::int32_t`）。

这一示例是从微软的示例程序 <a href="https://github.com/Microsoft/Windows-universal-samples/tree/master/Samples/AudioCreation">AudioCreation</a> 移植而成，其使用的编程语言为 C#。这里需要说明一下将 C# 代码迁移到 C++/WinRT 要做的一些事情：
- 更改属性的获取和设置。
    ```cs
    // C#
    AudioEncodingProperties nodeEncodingProperties = graph.EncodingProperties;
    nodeEncodingProperties.ChannelCount = 1;
    ```
    ```cpp
    // C++/WinRT
    AudioEncodingProperties nodeEncodingProperties = graph.EncodingProperties();
    nodeEncodingProperties.ChannelCount(1);
    ```
    可以看到，这一改动是将属性变成了成员变量的 `get` 和 `set` 函数，这两个函数同名。
- 更改注册事件处理程序的代码。
    ```cs
    // C#
    void node_QuantumStarted(AudioFrameInputNode sender, FrameInputNodeQuantumStartedEventArgs args);
    frameInputNode.QuantumStarted += node_QuantumStarted;
    ```
    ```cpp
    // C++/WinRT
    void frameInputQuantumStarted(const AudioFrameInputNode&, const FrameInputNodeQuantumStartedEventArgs&);
    frameInputNode.QuantumStarted(frameInputQuantumStarted);
    ```
- 要参见更多事项，参见微软文档 
<a href="https://docs.microsoft.com/zh-cn/windows/uwp/cpp-and-winrt-apis/move-to-winrt-from-csharp">从 C# 移动到 C++/WinRT</a>。