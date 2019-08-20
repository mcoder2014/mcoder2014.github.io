---
layout:     post
title:      "Share mouse and keyboard"
subtitle:   "共享鼠标和键盘"
date:       2019-08-20
author:     Mcoder
header-img: img/page_header.jpg
catalog: true
tags:
    - 不务正业
    - Synergy
---

# 键鼠共享
`Synergy` 是一款在不同操作系统间共享键鼠的软件，支持 Windows, Mac OS X, LINUX。
非常适合程序猿在桌上有多台电脑时使用，[配置参考](https://www.cnblogs.com/xzysaber/p/6502986.html)


# 破解
有个老外根据源码，写了个C++ 程序用来激活程序，该程序有个在线执行的地址[C++ Shell](http://cpp.sh/3mjw3)，可以在线执行，获得序列码。

也可以直接复制下方的代码，编译成C++ 程序，经验证，此破解程序在 `1.10`版本依旧有效，尚不确定是否会在后期更换验证方式。

```C++
//2017-June-3 scripted by genBTC, original code from Symless / Synergy (Github)
#include <iostream>
#include <sstream>
#include <iomanip>

static std::string
hexEncode (std::string const& str) {
	std::ostringstream oss;
	for (size_t i = 0; i < str.size(); ++i) {
		int c = str[i];
		oss << std::setfill('0') << std::hex << std::setw(2)
			<< std::uppercase;
		oss << c;
	}
	return oss.str();
}


int main()
{
  std::string name;
  std::string userlimit;
  std::string email;
  std::string business;
  std::cout << "What is your name? ";
  getline (std::cin, name);
  std::cout << "How many userlimit max? be reasonable ";
  getline (std::cin, userlimit);
  std::cout << "What is your E-mail address? ";
  getline (std::cin, email);  
  std::cout << "What is your business/company name? ";
  getline (std::cin, business);
  std::string key;
  key="{v1;pro;" + name + ";" + userlimit + ";" + email + ";" + business + ";0;0}";
  std::cout << "The Key is this: \n";
  std::cout << hexEncode(key);
}
```

# Reference
1. [Synergy](https://symless.com/synergy)
2. [C++ Shell](http://cpp.sh/3mjw3)
3. [配置](https://www.cnblogs.com/xzysaber/p/6502986.html)