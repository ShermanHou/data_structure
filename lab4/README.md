# Lab4实验报告

> 实验题目：最短路径中文文本分词
> 
> 姓名：侯新铭 
> 
> 学号：2021201651

## 1. 需求分析

    中文文本没有自然分界符，对文本语义的解析需要进行（相较于英文等语言）额外的分词`segmentation`操作。本次lab目标基于词库，实现一个输入中文文本，返回成功分词后文本的基础软件。

#### (1) 输入

    在项目`segmentation.exe`程序的输入框中输入待分词中文文本。

> 可支持文本长度：经文本长度测试，单次切词最多可支持约1万字文本
> 
> 运行时间：几乎可忽略不计

#### (2) 输出

    在交互界面的输出框显示分词后的字符串，格式为以"/"间隔的纯汉字文本。

#### (3) 功能实现

    我开发了一个基础的软件程序，在其可视化界面中，用户可交互式输入中文文本，点击分词按钮后即可立即看到分词后结果；后续用户直接更改其输入的文本内容，实现实时分词。

#### (4) 使用样例

<img title="" src="/assets/2.png" alt="" width="350" height="280" data-align="center">

## 2. 概要设计

##### 项目架构

①使用Qt Creator开发项目主体，项目文件包括：

<img title="" src="/assets/5.png" alt="" width="200" height="250" data-align="center">

②头文件调用关系为:

```
|-- main.cpp
        |-- window.h
            |-- seg.h

|-- window.cpp
        |-- window.h

|-- seg.cpp
        |-- seg.h
```

③头文件的主体分别为：

**seg.h**

```c++
#ifndef SEG_H
#define SEG_H
#include <queue>
#include <string>
#include <array>
#include <set>
using namespace std;
#define LEN 640
#define INF 0X7FFFFFFF
class Seg
{
public:
    // 载入dict.txt到set类型的dict变量
    bool loadDict(const string &location);
    string cut(string &s);

private:
    set<string> dict;
};

#endif
```

**window.h**

```c++
#ifndef WINDOW_H
#define WINDOW_H

#include <QMainWindow>
#include "seg.h"


QT_BEGIN_NAMESPACE
namespace Ui { class window; }
QT_END_NAMESPACE

class window : public QMainWindow
{
    Q_OBJECT

public:
    window(QWidget *parent = nullptr);
    ~window();

private slots:
    void on_pushButton_clicked();

private:
    Ui::window *ui;
    Seg seg;
};
#endif

```

## 3. 详细设计

#### (1) 分词功能实现

①词典加载

```c++
#include "seg.h"
#include <fstream>
#include <codecvt>
#include <locale>

// 载入dict.txt到set类型的dict变量
bool Seg::loadDict(const string &location)
{
    ifstream fin(location); // 通过ifstream流读取文件
    if (!fin.is_open())
    {
        return false;
    }

    string line;       // 将文件逐行读取到字符串line中，截取第一部分
    while (!fin.eof()) // 读取到文件末尾的EOF前一直执行while循环
    {
        getline(fin, line);
        int end = line.find_first_of(' '); // end对应第一列后的空格的索引下标,也即单词字符串的长度
        if (end != -1)                     // find_first_of()函数返回值不为-1，即找到了所给' '字符
        {
            dict.insert(line.substr(0, end)); // 将该词语子串插入到类内的private变量dict中
        }
     }
    return true;
}
```

②分词函数

采用最短路径匹配分词算法实现，主体循环遍历文本全部字符，其内包含如下3部分：

- 借助分隔符切分出由中文字符构成的单句

- 基于词典构建单句对应的有向无环图

- 执行dijkstra算法进行分词

- 将该单句切分出的词逆序添加到路径向量中

遍历完全部单句，主体循环结束，得到了由全部分词构成的路径向量，添加”/“作为分隔符，转化为分词后字符串，即为所求。

```c++
// 使用dijkstra算法获得最短路径
string Seg::cut(string &s)
{
    int startPos = 0;
    int fullLen = s.size();
    vector<string> path; // 使用vector记录最终所求的最短路径，方便进行插入、倒置等操作
    int count = 0;       // 用来记录上一句插词结束后path中的词数，作为下一句向path中插词的位置标记

    while (startPos < fullLen)
    {
        // 判断中文字符方式：基于中文字符由3字节构成，转化为unsigned int必然大于0x7f
        while (!((unsigned int)s[startPos] > 0x7f)) // 找到首中文字符的位置作为sentence的起始
        {
            startPos++;
            continue;
        }
        if(startPos>=fullLen) break;
        int endPos = startPos;
        do
        {
            endPos += 3;                                                 // 注意到一个中文字符在utf-8中占3个字节，故以3为步长
        } while (endPos < fullLen and ((unsigned int)s[endPos] > 0x7f)); // 找到连续中文字符末位置作为sentence的末尾的下一位
        string sentence = s.substr(startPos, endPos - startPos);

        int num = sentence.length() / 3; // 恰好为当前sentence包含的词数
        array<int, LEN> g;
        g.fill(-1);
        array<array<int, LEN>, LEN> graph;
        graph.fill(g); // 定义2dim array graph,索引到的数值记录sentence各位置之间的可达性。每个位置均初始化为-1，表示不可达

        for (int i = 0; i < num; i++)
        {
            graph[i][i + 1] = 2; // 每个字符和下一个字符显然可连通，距离为2，即闭区间跨越2个词可达到
        }

        for (int i = 0; i <= num - 2; i++)
        {
            for (int j = 2; j <= num - i and j <= 12; j++) // dict最长词长度为12字符，作为查找上限
            {
                string checkStr = sentence.substr(i * 3, j * 3);
                if (dict.count(checkStr))
                {
                    graph[i][i + j] = 1; // 更新gragh，表示i节点和i+j节点可处于同一个词内，距离为1
                }
            }
        }
        // 下述部分执行dijkstra算法过程，对当前的sentence进行切词
        // 初始化
        array<int, LEN> d; // 记录dijkstra算法执行到当前时刻，各节点到初始节点的距离
        d.fill(INF);
        d[0] = 0;
        array<int, LEN> preNum; // 记录各节点当前所得的最小路径的前驱节点序号
        preNum.fill(-1);
        array<int, LEN> used; // 记录是否已作为最短路径节点使用过
        used.fill(0);
        for (int i = 1; i <= num; i++) // 初始化与初始节点直接相连的各节点
        {
            if (graph[0][i] > 0)
            {
                d[i] = graph[0][i];
                preNum[i] = 0;
            }
        }
        // dijkstra算法主体
        for (int i = 1; i <= num; i++) // 遍历寻找到未被使用过的节点中距初始节点的最短距离，该节点序号记为k
        {
            int min = INF;
            int k = 0;
            if (!used[i] and d[i] < min)
            {
                min = d[i];
                k = i;
            }
            used[k] = 1; // k节点被使用，更新used

            for (int j = 1; j <= num; j++)
            {
                if (graph[k][j] > 0 and graph[k][j] + min < d[j]) // 借助k节点可以构造出到节点j到初始节点的更短距离，更新d和preNum
                {
                    d[j] = graph[k][j] + d[k];
                    preNum[j] = k;
                }
            }
        }

        while (num != 0) // 从当前centence最后字符处往前不断找前驱节点，过程中把词edges构建为路径vector path
        {
            string word = sentence.substr(preNum[num] * 3, (num - preNum[num]) * 3); // 截取以当前位置为末节点的词edge
            wstring_convert<codecvt_utf8<wchar_t>> cov;                              // 欲借助c++的iswpunct()和iswspace()来判断是否为中文标点，需先将string转化为wchar_t
            wchar_t ch = cov.from_bytes(word)[0];
            if (!iswpunct(ch) && !iswspace(ch)) // 判断为无（非）中文标点的word，插入path中固定位置count处，自然地实现了逆序插入
            {
                path.insert(path.begin() + count, word);
            }
            num = preNum[num];
        }
        count = path.size(); // 完成当前centence的分词后，更新位置标记count
        startPos = endPos;   // 更新startPos，向后推进
    }
    string output;
    for (int i = 0; i < count; i++) // 返回path中各词用"/"连接后的字符串
    {
        output.append(path[i] + "/");
    }
    return output;
}
```

#### (2) GUI界面开发

①使用Qt Creator创建`Qt Widgets Application`类型项目，生成主函数**main.cpp**为：

```c++
#include "window.h"

#include <QApplication>

// 使用Qt Creator创建project后自生成main代码，执行后，运行应用程序
int main(int argc, char *argv[])
{
    QApplication a(argc, argv);
    window w;
    w.show();
    return a.exec();
}
```

②在项目中新建**Q MainWindow**类型文件组，命名为`window`，其中包括：

```
|-- window.h
|-- window.cpp
|-- window.ui
```

③进而使用Qt Designer设计GUI界面，主要包括输入框、输出框、push按钮三个组件，如下图所示：
<img title="" src="/assets/6.png" alt="" width="195" height="140" data-align="center">

④在**window.h**中的`class window`中实例化已经写好的`class Seg`以调用`loadDict`和`cut`函数，最后实现qt中的接口槽函数`on_pushButton_clicked()`：

```c++
void window::on_pushButton_clicked()
{
    string s = ui->plainTextEdit->toPlainText().toStdString();
    string output = seg.cut(s);
    ui->textBrowser->setText(QString::fromStdString(output));
}
```

## 4. 调试分析

时间复杂度：

- dict通过哈希表形式的set定义, 调用check函数的时间复杂度为O(1)；

- 对于有向图G=(V, E) 记n 为节点数，dijkstra算法在分词最短路径问题中，搜索`d[]`中最小距离需O(n)，内层循环构建新edge，更新参数的循环也需O(n)，算法总体时间复杂度为是O(n^2)。

其他通过调试解决了的问题已在代码中详细注释。

## 5. 用户手册

- 项目在`Window`系统上开发，执行在Qt Creator由项目文件夹构建出的`segmentation.exe`程序即可。

> 请注意确保**window.cpp**中调用`loadDict`函数时所传路径为您的电脑中`dict.txt`文件的绝对路径。

- 程序运行后，您将看到如下交互界面：
  
  <img title="" src="/assets/7.png" alt="" width="350" height="280" data-align="center">

- 界面左侧设为输入框，右侧设为输出框，在输入框输入您待分词的由中文字符构成的文本（长度上限约一万字），单击界面中间的`分词`按钮，即可在右侧的输出框看到分词结果。

## 6. 测试结果

#### (1) 单句分词测试

<img title="" src="/assets/3.png" alt="" width="350" height="280" data-align="center">

#### (2) 长文本分词测试

<img title="" src="/assets/1.png" alt="" width="350" height="280" data-align="center">
<img title="" src="/assets/2.png" alt="" width="350" height="280" data-align="center">
<img title="" src="/assets/4.png" alt="" width="350" height="280" data-align="center">
