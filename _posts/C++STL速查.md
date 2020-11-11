# C++STL速查
[TOC]

## 指针
``` C++
int* p = new int[5];      // 两种声明方式等价
int p[5];

string str = new str[5];  // 两种声明方式等价
string str[5];
```

## vector容器

``` C++
#include<vector>

// 声明容器
vector<int> c;               // 创建一个空的vector。
vector<int> c1(c2);          // 复制一个vector
vector<int> c(n);            // 创建一个vector，含有n个数据，数据均已缺省构造产生
vector<int> c(n, elem);      // 创建一个含有n个elem拷贝的vector
vector<int> c(beg,end);      // 创建一个含有n个elem拷贝的vector

// 析构函数
c.~vector();                // 销毁所有数据，释放内存

// 插入删除元素
vec.push_back(a);           // 在尾部插入元素
vec.insert(vec.begin()+i,a);   // 在第i+1个元素前面插入a
vec.erase(vec.begin()+2);      // 删除第3个元素

// 访问元素
cout<<vec[0]<<endl;       // 通过下标访问
vec.front()               // 访问第一个元素
vec.back()                // 访问最后一个元素

// 使用迭代访问元素
vector<int>::iterator it;
for(it=vec.begin();it!=vec.end();it++)
    cout<<*it<<endl;

// 向量大小
vec.size();         // 返回向量大小
vec.resize();       // 改变当前使用数据的大小，如果它比当前使用的大，者填充默认值
vec.empty();        // 返回判断vector是否为空

// 清空
vec.clear();

// 最小值
int min_val = *min_element(vec.begin(), vec.end());

// 求和
int sum = accumulate(vec.begin(), vec.end(), 0);  // 求和再+0
```
  
  
## string
```c++
#include <string>

string str = "cat";
cout << "apple" + "boy" + str; // illegal! 使用重载的运算符 + 时，必须保证前两个操作数至少有一个为 string 类型。
char c = 'c';
string b = str + c;  // string可以 + char类型


//********************************查找*************************************
str.find("ab");//返回字符串 ab 在 str 的位置
str.find("ab", 2);//在 str[2]~str[n-1] 范围内查找并返回字符串 ab 在 str 的位置
str.rfind("ab", 2);//在 str[0]~str[2] 范围内查找并返回字符串 ab 在 str 的位置
//first 系列函数
str.find_first_of("apple");//返回 apple 中任何一个字符首次在 str 中出现的位置
str.find_first_of("apple", 2);//返回 apple 中任何一个字符首次在 str[2]~str[n-1] 范围中出现的位置
str.find_first_not_of("apple");//返回除 apple 以外的任何一个字符在 str 中首次出现的位置
str.find_first_not_of("apple", 2);//返回除 apple 以外的任何一个字符在 str[2]~str[n-1] 范围中首次出现的位置
//last 系列函数
str.find_last_of("apple");//返回 apple 中任何一个字符最后一次在 str 中出现的位置
str.find_last_of("apple", 2);//返回 apple 中任何一个字符最后一次在 str[0]~str[2] 范围中出现的位置
str.find_last_not_of("apple");//返回除 apple 以外的任何一个字符在 str 中最后一次出现的位置
str.find_last_not_of("apple", 2);//返回除 apple 以外的任何一个字符在 str[0]~str[2] 范围中最后一次出现的位置
//以上函数如果没有找到，均返回string::npos
cout << string::npos;


//子串
str.substr(3); //返回 [3] 及以后的子串
str.substr(2, 4); //返回 str[2]~str[2+(4-1)] 子串(即从[2]开始4个字符组成的字符串)


//替换
str.replace(2, 4, "sz");//返回把 [2]~[2+(4-1)] 的内容替换为 "sz" 后的新字符串
str.replace(2, 4, "abcd", 3);//返回把 [2]~[2+(4-1)] 的内容替换为 "abcd" 的前3个字符后的新字符串


//插入
str.insert(2, "sz");//从 [2] 位置开始添加字符串 "sz"，并返回形成的新字符串
str.insert(2, "abcd", 3);//从 [2] 位置开始添加字符串 "abcd" 的前 3 个字符，并返回形成的新字符串
str.insert(2, "abcd", 1, 3);//从 [2] 位置开始添加字符串 "abcd" 的前 [2]~[2+(3-1)] 个字符，并返回形成的新字符串


//追加
str.push_back('a');//在 str 末尾添加字符'a'
str.append("abc");//在 str 末尾添加字符串"abc"


//删除
str.erase(3);//删除 [3] 及以后的字符，并返回新字符串
str.erase(3, 5);//删除从 [3] 开始的 5 个字符，并返回新字符串


//交换
str1.swap(str2);//把 str1 与 str2 交换

//其他
str.size();//返回字符串长度
str.length();//返回字符串长度
str.empty();//检查 str 是否为空，为空返回 1，否则返回 0
str[n];//存取 str 第 n + 1 个字符
str.at(n);//存取 str 第 n + 1 个字符（如果溢出会抛出异常）
```

## map & unordered_map
存储时是根据key的hash值判断元素是否相同，即unordered_map内部元素是无序的，而map中的元素是按照二叉搜索树存储，进行中序遍历会得到有序遍历。
```c++
// 声明
map<string, int> map;
unordered_map<string, int> map;

// 插入元素
map["one"] = 1;
map.insert(map<string, int> :: value_type("two", 2));
map.insert(pair<string, int>("five", 5)); 
map.insert(make_pair("three", 3)); 

// 查找
int num = map["two"];
if (map.find("four") == map.end())  // 没找到
if (map.count("one") != 0)          // 找到了
map.lower_bound("n")                // 返回键值>=给定元素的第一个位置
upper_bound()                       // 返回键值>给定元素的第一个位置

// 删除
map<string, int>::iterator erase(iterator it);                    // 通过一个条目对象删除 
map<string, int>::iterator erase(iterator first, iterator last);  // 删除一个范围 
map<string, int>::size_type erase(const Key& key);                // 通过key删除 
map.clear()                                     // 就相当于下面
map.erase(enumMap.begin(), enumMap.end());

// 容量
map.max_size()       // 返回可以容纳的最大元素个数
map.size()           // 返回map中元素的个数
map.empty()          // 返回map是否为空

// 遍历
for (auto m : map)
    cout << m.first;
for (auto iter = map.begin(); iter != map.end; ++iter)
    cout << iter -> second;

// 声明map的operator < (对于unordered_map，须声明 ==)
struct person  
{  
    string name;  
    int age;
    person(string name, int age)  
    {  
        this->name =  name;  
        this->age = age;  
    }
    bool operator < (const person& p) const  
    {  
        return this->age < p.age;     // age相同的person将会覆盖
    }  
}; 
map<person,int> m;
```
  
  
## set& unordered_set

```c++
// 对于键值为自定义类型的情况，声明容器时必须提供一个比较的函数，传入函数指针时不必些为&compare，因为函数名会自动转换为函数指针。
multiset<data, decltype(compare)*> st(compare);
bool compare(const data &lhs, const data &rhs) {
    return lhs.num() < rhs.num();
}
```
multiset和multimap不能进行下标操作，访问时可以使用find()和count(),或lower_bound()和upper_bound()，或equal_range()  
lower_bound()和upper_bound()如果没有找到对应元素，会返回小于和大于该元素的元素。  
equal_range()返回pair，分别是头和尾。

<br/>

## sort
可以用自定义函数，lambda表达式，函数对象
```c++
bool compare(int a,int b)
{
  return a<b; //升序排列，如果改为return a>b，则为降序
}
int a[20]={2,4,1,23,5,76,0,43,24,65};
sort(a,a+20,compare);

sort(a,a+10,less<int>());
```


<br/>

## priority_queue

```c++
// 声明1
struct compare {
    bool operator()(const ListNode* l1, const ListNode* l2) {  // 类似lambda表达式，const是必须的
        return l1 -> val > l2 -> val;            // 注意这里是大于
    }
};
priority_queue<ListNode*, vector<ListNode*>, compare> pq;

// 声明2
struct Node{
     int x, y;
     Node( int a= 0, int b= 0 ): x(a), y(b) {}
 };
bool operator<( Node a, Node b ){
     if( a.x== b.x ) return a.y> b.y;
     return a.x> b.x;
 }
priority_queue<Node> q;

// 声明3
struct Node{
     int x, y;
     Node( int a= 0, int b= 0 ):
         x(a), y(b) {}
    friend operator<( Node a, Node b ){
        if( a.x== b.x ) return a.y> b.y;
        return a.x> b.x;
    }
 };
priority_queue<Node> q;

// 操作
q.top();              // 返回第一个元素
q.pop();              // 删除第一个元素
q.size();             // 返回大小
q.empty();            // 返回是否为空
q.push();             // 插入一个元素
```

## pair和tuple
```
构造
pair<int ,double> p1;          // 默认构造函数
pair<int ,double> p2(1, 2.4);  // 用给定值初始化
pair<int ,double> p3(p2);      // 拷贝构造函数
pair<int ,double> p4 = make_pair(1, 1.2);   // 根据参数推断类型
auto item = make_tuple("123", 3, 20.00);   // tuple<const char*, int, double>

p1.first = 1;
p1.second = 2.5;
```
tuple可以有任意数量的成员，类似结构体
```
tuple<int, char, float> threeD(1, 2, 3);  // 直接构造
tuple<int, char, float> threeD{1, 2, 3};  // 列表构造?(不确定类型参数不同时是否可行)
tuple<int, char, float> threeD = {1, 2, 3};  // 错误：explicit禁止隐式转换
tuple<const char*, int>tp = make_tuple("as",2); //通过参数推断类型。

auto book = get<0>(item);    // 获取第0个元素，返回引用

typedef decltype(item) trans;   // trans是item的类型
size_t sz = tuple_size<trans>::value;   // tuple_size获取item的元素个数3
tuple_element<1, trans>::type cnt = get<1>(item);  // tuple_element获取元素类型
```

## bitset
```
#include <bitset>
std::bitset<8> bs;      

// 模板参数是一个size_t类型的数值（value），而非一个类型
// numeric_limits<size_t>::min() == 0
// std::bitset<8> 表示的二进制位为8位，
// 默认的构造函数将其初始为全0

std::bitset<13> bitvec1(0xbeef);    // 高位舍弃
std::bitset<20> bitvec1(0xbeef);    // 高位补0
bitset<n> b(s,bpos, m, zero, one);  // 把string("1010")从bpos,到bpos+m转换为b
bitset<numeric_limits<unsigned short>::digits> bs1(267);  // 16位 
bitset<numeric_limits<unsigned long>::digits> bs2(267);   // 32位

bs[0] = 1;    // 0000 0001 与vector的索引相反
bs[7] = 1;    // 1000 0000 与vector的索引相反
cout << bs.to_ulong() << endl;      // 0
cout << bs.to_string() << endl;     // "00000000"
```

成员函数 | 函数功能
|:-----:|:-----:|
bs.any()|是否存在值为1的二进制位
bs.none() |  是否不存在值为1的二进制位,或者说是否全部位为0
bs.size() |  位长，也即是非模板参数值
bs.count() | 值为1的个数
bs.test(pos) | 测试pos处的二进制位是否为1,与0做或运算
bs.set() | 全部位置1
bs.set(pos) | pos位处的二进制位置1,与1做或运算
bs.reset() | 全部位置0
bs.reset(pos) |  pos位处的二进制位置0,与0做或运算
bs.flip() |  全部位逐位取反
bs.flip(pos) | pos处的二进制位取反
bs.to_ulong() |  将二进制转换为unsigned long输出
bs.to_string() | 将二进制转换为字符串输出
~bs | 按位取反,效果等效为bs.flip()
os << b | 将二进制位输出到os流,小值在右，大值在左

## 正则表达式
[正则表达式组件](https://www.cnblogs.com/szn409/p/7471427.html):
函数/类| 功能
|:--:|:-:|
regex|用于表示一个正则表达式的类
regex_match|将一个字符序列与一个正则表达式匹配
regex_search|寻找第一个与正则表达式匹配的子序列
regex_replace|使用给定格式替换一个正则表达式
sregex_iterator|迭代器适配器，调用regex_search来遍历一个string中所有匹配的字串
smatch|容器类，保存在string中搜索的结果
ssub_match|string中匹配的子表达式的结果

regex_search, regex_match的参数(返回值均为bool类型):  
seq, m, r, mft  
seq, r, mft  
seq:字符序列  
m:与seq兼容的match对象  
r:一个正则表达式  
mtf:可选值，用来保存匹配结果的相关细节  

## 随机数

```
// 引擎：主要确定种子
default_random_engine e1;     // 创建随机数引擎
default_random_engine e2(10); // 设置种子为10
e1.seed(10);   // 设置新的种子为10
e2.seed(time(0));     // 把系统时间作为种子
e1.min();  e1.max();  // 引擎可以生成的最小、最大值

// 分布：主要确定分布的类型和参数
uniform_int_distribution<unsigned> u(0, 9); // [0,9]整数均匀分布
uniform_real_distribution<float> u(0, 9); // [0,9]小数均匀分布
normal_distribution<> n(m, s); //正态分布，默认类型为double，均值为m(=0.0)，标准差为s(=1.0)
u.min();  u.max();  // 分布可以生成的最小、最大值

for (int i = 0; i < 9; ++i) {
    cout << e() << endl;
    cout << u(e) << endl;
}
```