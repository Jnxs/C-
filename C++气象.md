##  二、如何开发永不停机的服务程序

### 2.1 开篇语

一、生成测试数据

+ 实现生成测试数据的功能，提升代码能力
+ 逐步掌握开发框架的使用
+ 熟悉csv、xml 和 json 这三种常用的数据格式

二、服务程序的调度

+ 学习Linux信号的基础知识和使用方法
+ 学习Linux多进程的基础知识和使用方法
+ 开发服务程序的调度模块

三、守护进程的实现

+ 学习Linux共享内存的基础知识和使用方法
+ 学习Linux信号量的基础知识和使用方法
+ 开发守护进程模块，与调度模块结合，保证服务程序永不停机

四、两个常用的小工具

+ 开发压缩文件模块
+ 开发清理历史数据文件模块



### 2.2生成测试数据-搭建程序的框架

```c++
#include "_public.h"

CLogFile logfile(10);

int main(int argc, char* argv[]) {

    // inifile outpath logfile
    if (argc != 4) {
        printf("Using: ./crtsurfdata1 inifile outpath logfile\n");
        printf("Example: /project/idc1/bin/crtsurfdata1 /project/idc1/ini/stcode.ini "
            "/tmp/surfdata /log/idc/crtsurfdata1.log \n\n");

        printf("inifile 全国气象站点参数文件名\n");
        printf("outpath 全国气象站点数据文件存放的目录\n");
        printf("logfile 本程序运行时的日志文件名\n\n");

        return -1;
    }

    if (logfile.Open(argv[3], "a+", false) == false) {
        perror("open failed");
        return -1;
    }

    logfile.Write("crtsurfdata1 开始运行。\n");

    // todo

    logfile.Write("crtsurfdata1 结束运行。\n");

    return 0;
}

```



### 2.3 生成测试数据-加载站点参数

```c++
#include "_public.h"

// 全国气象站点参数结构体
struct st_stcode {
    char provname[31]; // 省
    char obtid[11];    // 站号
    char obtname[31];  // 站名
    double lat;        // 纬度
    double lon;        // 经度
    double height;     // 海拔高度
};

// 存放全国气象站点参数的容器
vector<st_stcode> vstcode;

// 把站点参数文件中加载到vstcode容器中
bool LoadSTCode(const char* inifile);

CLogFile logfile;

int main(int argc, char* argv[]) {

    // inifile outpath logfile
    if (argc != 4) {
        printf("Using: ./crtsurfdata2 inifile outpath logfile\n");
        printf("Example: /project/idc1/bin/crtsurfdata2 /project/idc1/ini/stcode.ini "
            "/tmp/surfdata /log/idc/crtsurfdata2.log \n\n");

        printf("inifile 全国气象站点参数文件名\n");
        printf("outpath 全国气象站点数据文件存放的目录\n");
        printf("logfile 本程序运行时的日志文件名\n\n");

        return -1;
    }

    if (logfile.Open(argv[3], "a+", false) == false) {
        perror("open failed");
        return -1;
    }

    logfile.Write("crtsurfdata2 开始运行。\n");

    if (LoadSTCode(argv[1]) == false) {
        return -1;
    }

    logfile.Write("crtsurfdata2 结束运行。\n");

    return 0;
}

bool LoadSTCode(const char* inifile) {
    CFile File;
    // 打开站点参数文件
    if (File.Open(inifile, "r") == false) {
        logfile.Write("File.Open(%s) failed.\n", inifile);
        return -1;
    }

    char strBuffer[301];

    CCmdStr CmdStr;

    st_stcode stcode;

    while (1) {
        // 从站点参数文件中读取一行。如果已经读取完，跳出循环
        if (File.Fgets(strBuffer, 300, true) == false) {
            break;
        }

        // 把读取到的一行拆分
        CmdStr.SplitToCmd(strBuffer, ",", true);

        if ((CmdStr.CmdCount() != 6)) {
            continue; // 丢掉无效的行
        }

        // 把站点参数的每个数据项保存到站点参数结构体中
        CmdStr.GetValue(0, stcode.provname, 30);   // 省
        CmdStr.GetValue(1, stcode.obtid, 10);      // 站号
        CmdStr.GetValue(2, stcode.obtname, 30);    // 站名
        CmdStr.GetValue(3, &stcode.lat);           // 纬度
        CmdStr.GetValue(4, &stcode.lon);           // 经度
        CmdStr.GetValue(5, &stcode.height);        // 海拔高度

        // 把站点参数结构体放入站点参数容器
        vstcode.push_back(stcode);
    }

    for (int i = 0; i < vstcode.size(); ++i) {
        logfile.Write("provname = %s, obtid = %s, obtname = %s, lat = %.2f, lon = %.2f, "
            "height = %.2f\n", vstcode[i].provname, vstcode[i].obtid, vstcode[i].obtname,\
            vstcode[i].lat, vstcode[i].lon, vstcode[i].height);
    }

    return true;
}
```



### 2.4 生成观测数据-模拟观测数据

```c++
#include "_public.h"

// 全国气象站点参数结构体
struct st_stcode {
    char provname[31]; // 省
    char obtid[11];    // 站号
    char obtname[31];  // 站名
    double lat;        // 纬度
    double lon;        // 经度
    double height;     // 海拔高度
};

// 存放全国气象站点参数的容器
vector<st_stcode> vstcode;

// 把站点参数文件中加载到vstcode容器中
bool LoadSTCode(const char* inifile);


// 全国气象站点分钟观测数据结构
struct st_surfdata {
  char obtid[11];      // 站点代码。
  char ddatetime[21];  // 数据时间：格式yyyymmddhh24miss
  int  t;              // 气温：单位，0.1摄氏度。
  int  p;              // 气压：0.1百帕。
  int  u;              // 相对湿度，0-100之间的值。
  int  wd;             // 风向，0-360之间的值。
  int  wf;             // 风速：单位0.1m/s
  int  r;              // 降雨量：0.1mm。
  int  vis;            // 能见度：0.1米。
};

vector<st_surfdata> vsurfdata; // 存放全国气象站点分钟观测数据的容器

// 模拟生成全国气象站点分钟观测数据，存放在vsurfdata容器中
void CrtSurfData();

CLogFile logfile;

int main(int argc, char* argv[]) {

    // inifile outpath logfile
    if (argc != 4) {
        printf("Using: ./crtsurfdata3 inifile outpath logfile\n");
        printf("Example: /project/idc1/bin/crtsurfdata3 /project/idc1/ini/stcode.ini "
            "/tmp/surfdata /log/idc/crtsurfdata3.log \n\n");

        printf("inifile 全国气象站点参数文件名\n");
        printf("outpath 全国气象站点数据文件存放的目录\n");
        printf("logfile 本程序运行时的日志文件名\n\n");

        return -1;
    }

    if (logfile.Open(argv[3], "a+", false) == false) {
        perror("open failed");
        return -1;
    }

    logfile.Write("crtsurfdata3 开始运行。\n");

    if (LoadSTCode(argv[1]) == false) {
        return -1;
    }

    CrtSurfData();

    logfile.Write("crtsurfdata3 结束运行。\n");

    return 0;
}

bool LoadSTCode(const char* inifile) {
    CFile File;
    // 打开站点参数文件
    if (File.Open(inifile, "r") == false) {
        logfile.Write("File.Open(%s) failed.\n", inifile);
        return -1;
    }

    char strBuffer[301];

    CCmdStr CmdStr;

    st_stcode stcode;

    while (1) {
        // 从站点参数文件中读取一行。如果已经读取完，跳出循环
        if (File.Fgets(strBuffer, 300, true) == false) {
            break;
        }

        // 把读取到的一行拆分
        CmdStr.SplitToCmd(strBuffer, ",", true);

        if ((CmdStr.CmdCount() != 6)) {
            continue; // 丢掉无效的行
        }

        // 把站点参数的每个数据项保存到站点参数结构体中
        CmdStr.GetValue(0, stcode.provname, 30);   // 省
        CmdStr.GetValue(1, stcode.obtid, 10);      // 站号
        CmdStr.GetValue(2, stcode.obtname, 30);    // 站名
        CmdStr.GetValue(3, &stcode.lat);           // 纬度
        CmdStr.GetValue(4, &stcode.lon);           // 经度
        CmdStr.GetValue(5, &stcode.height);        // 海拔高度

        // 把站点参数结构体放入站点参数容器
        vstcode.push_back(stcode);
    }

    return true;
}

void CrtSurfData() {
    // 播随机数种子。
    srand(time(0));

    // 获取当前时间，当作观测时间。
    char strddatetime[21];
    memset(strddatetime, 0, sizeof(strddatetime));
    LocalTime(strddatetime, "yyyymmddhh24miss");

    st_surfdata stsurfdata;

    // 遍历气象站点参数的vstcode容器。
    for (int i = 0; i < vstcode.size(); ++i) {
        memset(&stsurfdata, 0, sizeof(st_surfdata));

        // 用随机数填充分钟观测数据的结构体。
        strncpy(stsurfdata.obtid, vstcode[i].obtid, 10); // 站点代码。
        strncpy(stsurfdata.ddatetime, strddatetime,14);  // 数据时间：格式yyyymmddhh24miss
        stsurfdata.t = rand()%351;       // 气温：单位，0.1摄氏度
        stsurfdata.p = rand()%265+10000; // 气压：0.1百帕
        stsurfdata.u = rand()%101;     // 相对湿度，0-100之间的值。
        stsurfdata.wd = rand()%361;      // 风向，0-360之间的值。
        stsurfdata.wf = rand()%150;      // 风速：单位0.1m/s
        stsurfdata.r = rand()%16;        // 降雨量：0.1mm
        stsurfdata.vis = rand()%5001+100000;  // 能见度：0.1米

        // 把观测数据的结构体放入vsurfdata容器。
        vsurfdata.push_back(stsurfdata);
    }
}
```



### 2.5 生成测试数据-生成csv文件

```c++
#include "_public.h"

// 全国气象站点参数结构体
struct st_stcode {
    char provname[31]; // 省
    char obtid[11];    // 站号
    char obtname[31];  // 站名
    double lat;        // 纬度
    double lon;        // 经度
    double height;     // 海拔高度
};

// 存放全国气象站点参数的容器
vector<st_stcode> vstcode;

// 把站点参数文件中加载到vstcode容器中
bool LoadSTCode(const char* inifile);


// 全国气象站点分钟观测数据结构
struct st_surfdata {
  char obtid[11];      // 站点代码。
  char ddatetime[21];  // 数据时间：格式yyyymmddhh24miss
  int  t;              // 气温：单位，0.1摄氏度。
  int  p;              // 气压：0.1百帕。
  int  u;              // 相对湿度，0-100之间的值。
  int  wd;             // 风向，0-360之间的值。
  int  wf;             // 风速：单位0.1m/s
  int  r;              // 降雨量：0.1mm。
  int  vis;            // 能见度：0.1米。
};

vector<st_surfdata> vsurfdata; // 存放全国气象站点分钟观测数据的容器


char strddatetime[21]; // 观测数据的时候

// 模拟生成全国气象站点分钟观测数据，存放在vsurfdata容器中
void CrtSurfData();

// 把容器vsurfdata中的全国气象站点分钟观测数据写入文件
bool CrtSurfFile(const char* outpath, const char* datafmt);

CLogFile logfile;

int main(int argc, char* argv[]) {

    // inifile outpath logfile
    if (argc != 5) {
        printf("Using: ./crtsurfdata4 inifile outpath logfile datafmt\n");
        printf("Example: /project/idc1/bin/crtsurfdata4 /project/idc1/ini/stcode.ini "
            "/tmp/surfdata /log/idc/crtsurfdata4.log xml,json,csv\n\n");

        printf("inifile 全国气象站点参数文件名\n");
        printf("outpath 全国气象站点数据文件存放的目录\n");
        printf("logfile 本程序运行时的日志文件名\n");
        printf("datafmt 生成程序文件的格式，支持xml、json和csv三种格式\n\n");

        return -1;
    }

    if (logfile.Open(argv[3], "a+", false) == false) {
        perror("open failed");
        return -1;
    }

    logfile.Write("crtsurfdata4 开始运行。\n");

    // 把站点参数文件中加载到vstcode容器中
    if (LoadSTCode(argv[1]) == false) {
        return -1;
    }

    // 模拟生成全国气象站点分钟观测数据，存放在surfdata容器中
    CrtSurfData();

    // 把容器vsurfdata中全国气象站点分钟观测数据写入文件
    if (strstr(argv[4], "xml") != 0)    CrtSurfFile(argv[2], "xml");
    if (strstr(argv[4], "json") != 0)    CrtSurfFile(argv[2], "json");
    if (strstr(argv[4], "csv") != 0)    CrtSurfFile(argv[2], "csv");

    logfile.Write("crtsurfdata4 结束运行。\n");

    return 0;
}

bool LoadSTCode(const char* inifile) {
    CFile File;
    // 打开站点参数文件
    if (File.Open(inifile, "r") == false) {
        logfile.Write("File.Open(%s) failed.\n", inifile);
        return -1;
    }

    char strBuffer[301];

    CCmdStr CmdStr;

    st_stcode stcode;

    while (1) {
        // 从站点参数文件中读取一行。如果已经读取完，跳出循环
        if (File.Fgets(strBuffer, 300, true) == false) {
            break;
        }

        // 把读取到的一行拆分
        CmdStr.SplitToCmd(strBuffer, ",", true);

        if ((CmdStr.CmdCount() != 6)) {
            continue; // 丢掉无效的行
        }

        // 把站点参数的每个数据项保存到站点参数结构体中
        CmdStr.GetValue(0, stcode.provname, 30);   // 省
        CmdStr.GetValue(1, stcode.obtid, 10);      // 站号
        CmdStr.GetValue(2, stcode.obtname, 30);    // 站名
        CmdStr.GetValue(3, &stcode.lat);           // 纬度
        CmdStr.GetValue(4, &stcode.lon);           // 经度
        CmdStr.GetValue(5, &stcode.height);        // 海拔高度

        // 把站点参数结构体放入站点参数容器
        vstcode.push_back(stcode);
    }

    return true;
}

void CrtSurfData() {
    // 播随机数种子。
    srand(time(0));

    // 获取当前时间，当作观测时间。

    memset(strddatetime, 0, sizeof(strddatetime));
    LocalTime(strddatetime, "yyyymmddhh24miss");

    st_surfdata stsurfdata;

    // 遍历气象站点参数的vstcode容器。
    for (int i = 0; i < vstcode.size(); ++i) {
        memset(&stsurfdata, 0, sizeof(st_surfdata));

        // 用随机数填充分钟观测数据的结构体。
        strncpy(stsurfdata.obtid, vstcode[i].obtid, 10); // 站点代码。
        strncpy(stsurfdata.ddatetime, strddatetime,14);  // 数据时间：格式yyyymmddhh24miss
        stsurfdata.t = rand()%351;       // 气温：单位，0.1摄氏度
        stsurfdata.p = rand()%265+10000; // 气压：0.1百帕
        stsurfdata.u = rand()%101;     // 相对湿度，0-100之间的值。
        stsurfdata.wd = rand()%361;      // 风向，0-360之间的值。
        stsurfdata.wf = rand()%150;      // 风速：单位0.1m/s
        stsurfdata.r = rand()%16;        // 降雨量：0.1mm
        stsurfdata.vis = rand()%5001+100000;  // 能见度：0.1米

        // 把观测数据的结构体放入vsurfdata容器。
        vsurfdata.push_back(stsurfdata);
    }
}

bool CrtSurfFile(const char* outpath, const char* datafmt) {
    CFile File;

    // 拼接生成数据的文件名，例如: SURF_ZH_
    char strFileName[301];
    sprintf(strFileName, "%s/SURF_ZH_%s_%d.%s", outpath, strddatetime, getpid(), datafmt);
    
    // 打开文件
    if (File.OpenForRename(strFileName, "w") == false) {
        logfile.Write("File.OpenForRename(%s) failed.\n", strFileName);
        return -1;
    }

    // 写入第一行标题
    if (strcmp(datafmt, "csv") == 0) {
        File.Fprintf("站点代码, 数据时间, 气温, 气压, 相对湿度, 风向, 风速, 降雨量, 能见度\n");
    }

    // 遍历存放观测数据的vsurfdata容器
    for (int i = 0; i < vsurfdata.size(); ++i) {
        // 写入一条记录
        if (strcmp(datafmt, "csv") == 0) {
            File.Fprintf("%s, %s, %.1f, %.1f, %d, %d, %.1f, %.1f, %.1f\n",\
                vsurfdata[i].obtid,vsurfdata[i].ddatetime,vsurfdata[i].t/10.0,vsurfdata[i].p/10.0,\
                vsurfdata[i].u,vsurfdata[i].wd,vsurfdata[i].wf/10.0,vsurfdata[i].r/10.0,vsurfdata[i].vis/10.0);
        }
    }

    // 关闭文件。
    File.CloseAndRename();

    logfile.Write("生成数据文件%s成功，数据时间%s，记录数%d。\n",strFileName, strddatetime, vsurfdata.size());

    return true;
}
```



### 2.6 生成测试数据-生成xml和json文件

```c++
#include "_public.h"

// 全国气象站点参数结构体
struct st_stcode {
    char provname[31]; // 省
    char obtid[11];    // 站号
    char obtname[31];  // 站名
    double lat;        // 纬度
    double lon;        // 经度
    double height;     // 海拔高度
};

// 存放全国气象站点参数的容器
vector<st_stcode> vstcode;

// 把站点参数文件中加载到vstcode容器中
bool LoadSTCode(const char* inifile);


// 全国气象站点分钟观测数据结构
struct st_surfdata {
  char obtid[11];      // 站点代码。
  char ddatetime[21];  // 数据时间：格式yyyymmddhh24miss
  int  t;              // 气温：单位，0.1摄氏度。
  int  p;              // 气压：0.1百帕。
  int  u;              // 相对湿度，0-100之间的值。
  int  wd;             // 风向，0-360之间的值。
  int  wf;             // 风速：单位0.1m/s
  int  r;              // 降雨量：0.1mm。
  int  vis;            // 能见度：0.1米。
};

vector<st_surfdata> vsurfdata; // 存放全国气象站点分钟观测数据的容器


char strddatetime[21]; // 观测数据的时候

// 模拟生成全国气象站点分钟观测数据，存放在vsurfdata容器中
void CrtSurfData();

// 把容器vsurfdata中的全国气象站点分钟观测数据写入文件
bool CrtSurfFile(const char* outpath, const char* datafmt);

CLogFile logfile;

int main(int argc, char* argv[]) {

    // inifile outpath logfile
    if (argc != 5) {
        printf("Using: ./crtsurfdata5 inifile outpath logfile datafmt\n");
        printf("Example: /project/idc1/bin/crtsurfdata5 /project/idc1/ini/stcode.ini "
            "/tmp/surfdata /log/idc/crtsurfdata5.log xml,json,csv\n\n");

        printf("inifile 全国气象站点参数文件名\n");
        printf("outpath 全国气象站点数据文件存放的目录\n");
        printf("logfile 本程序运行时的日志文件名\n");
        printf("datafmt 生成程序文件的格式，支持xml、json和csv三种格式\n\n");

        return -1;
    }

    if (logfile.Open(argv[3], "a+", false) == false) {
        perror("open failed");
        return -1;
    }

    logfile.Write("crtsurfdata5 开始运行。\n");

    // 把站点参数文件中加载到vstcode容器中
    if (LoadSTCode(argv[1]) == false) {
        return -1;
    }

    // 模拟生成全国气象站点分钟观测数据，存放在surfdata容器中
    CrtSurfData();

    // 把容器vsurfdata中全国气象站点分钟观测数据写入文件
    if (strstr(argv[4], "xml") != 0)    CrtSurfFile(argv[2], "xml");
    if (strstr(argv[4], "json") != 0)    CrtSurfFile(argv[2], "json");
    if (strstr(argv[4], "csv") != 0)    CrtSurfFile(argv[2], "csv");

    logfile.Write("crtsurfdata5 结束运行。\n");

    return 0;
}

bool LoadSTCode(const char* inifile) {
    CFile File;
    // 打开站点参数文件
    if (File.Open(inifile, "r") == false) {
        logfile.Write("File.Open(%s) failed.\n", inifile);
        return -1;
    }

    char strBuffer[301];

    CCmdStr CmdStr;

    st_stcode stcode;

    while (1) {
        // 从站点参数文件中读取一行。如果已经读取完，跳出循环
        if (File.Fgets(strBuffer, 300, true) == false) {
            break;
        }

        // 把读取到的一行拆分
        CmdStr.SplitToCmd(strBuffer, ",", true);

        if ((CmdStr.CmdCount() != 6)) {
            continue; // 丢掉无效的行
        }

        // 把站点参数的每个数据项保存到站点参数结构体中
        CmdStr.GetValue(0, stcode.provname, 30);   // 省
        CmdStr.GetValue(1, stcode.obtid, 10);      // 站号
        CmdStr.GetValue(2, stcode.obtname, 30);    // 站名
        CmdStr.GetValue(3, &stcode.lat);           // 纬度
        CmdStr.GetValue(4, &stcode.lon);           // 经度
        CmdStr.GetValue(5, &stcode.height);        // 海拔高度

        // 把站点参数结构体放入站点参数容器
        vstcode.push_back(stcode);
    }

    return true;
}

void CrtSurfData() {
    // 播随机数种子。
    srand(time(0));

    // 获取当前时间，当作观测时间。

    memset(strddatetime, 0, sizeof(strddatetime));
    LocalTime(strddatetime, "yyyymmddhh24miss");

    st_surfdata stsurfdata;

    // 遍历气象站点参数的vstcode容器。
    for (int i = 0; i < vstcode.size(); ++i) {
        memset(&stsurfdata, 0, sizeof(st_surfdata));

        // 用随机数填充分钟观测数据的结构体。
        strncpy(stsurfdata.obtid, vstcode[i].obtid, 10); // 站点代码。
        strncpy(stsurfdata.ddatetime, strddatetime,14);  // 数据时间：格式yyyymmddhh24miss
        stsurfdata.t = rand()%351;       // 气温：单位，0.1摄氏度
        stsurfdata.p = rand()%265+10000; // 气压：0.1百帕
        stsurfdata.u = rand()%101;     // 相对湿度，0-100之间的值。
        stsurfdata.wd = rand()%361;      // 风向，0-360之间的值。
        stsurfdata.wf = rand()%150;      // 风速：单位0.1m/s
        stsurfdata.r = rand()%16;        // 降雨量：0.1mm
        stsurfdata.vis = rand()%5001+100000;  // 能见度：0.1米

        // 把观测数据的结构体放入vsurfdata容器。
        vsurfdata.push_back(stsurfdata);
    }
}

bool CrtSurfFile(const char* outpath, const char* datafmt) {
    CFile File;

    // 拼接生成数据的文件名，例如: SURF_ZH_20220412122707_11193.xml
    char strFileName[301];
    sprintf(strFileName, "%s/SURF_ZH_%s_%d.%s", outpath, strddatetime, getpid(), datafmt);
    
    // 打开文件
    if (File.OpenForRename(strFileName, "w") == false) {
        logfile.Write("File.OpenForRename(%s) failed.\n", strFileName);
        return -1;
    }

    // 写入第一行标题
    if (strcmp(datafmt, "csv") == 0) {
        File.Fprintf("站点代码, 数据时间, 气温, 气压, 相对湿度, 风向, 风速, 降雨量, 能见度\n");
    }
    if (strcmp(datafmt, "xml") == 0) {
        File.Fprintf("<data>\n");
    }
    if (strcmp(datafmt, "json") == 0) {
        File.Fprintf("{\"data\":[\n");
    }

    // 遍历存放观测数据的vsurfdata容器
    for (int i = 0; i < vsurfdata.size(); ++i) {
        // 写入一条记录
        if (strcmp(datafmt, "csv") == 0) {
            File.Fprintf("%s, %s, %.1f, %.1f, %d, %d, %.1f, %.1f, %.1f\n",\
                vsurfdata[i].obtid,vsurfdata[i].ddatetime,vsurfdata[i].t/10.0,vsurfdata[i].p/10.0,\
                vsurfdata[i].u,vsurfdata[i].wd,vsurfdata[i].wf/10.0,vsurfdata[i].r/10.0,vsurfdata[i].vis/10.0);
        }
        if (strcmp(datafmt, "xml") == 0) {
            File.Fprintf("<obtid>%s</obtid><ddatetime>%s</ddatetime><t>%.1f</t><p>%.1f</p><u>%d</u><wd>%d"
                "</wd><wf>%.1f</wf><r>%.1f</r><vis>%.1f</vis><endl/>\n",\
                vsurfdata[i].obtid,vsurfdata[i].ddatetime,vsurfdata[i].t/10.0,vsurfdata[i].p/10.0,\
                vsurfdata[i].u,vsurfdata[i].wd,vsurfdata[i].wf/10.0,vsurfdata[i].r/10.0,vsurfdata[i].vis/10.0);
        }
        if (strcmp(datafmt, "json") == 0) {
            File.Fprintf("{\"obtid\":\"%s\",\"ddatetime\":\"%s\",\"t\":\"%.1f\",\"p\":\"%.1f\","
                "\"u\":\"%d\",\"wd\":\"%d\",\"wf\":\"%.1f\",\"r\":\"%.1f\",\"vis\":\"%.1f\"}",\
                vsurfdata[i].obtid,vsurfdata[i].ddatetime,vsurfdata[i].t/10.0,vsurfdata[i].p/10.0,\
                vsurfdata[i].u,vsurfdata[i].wd,vsurfdata[i].wf/10.0,vsurfdata[i].r/10.0,vsurfdata[i].vis/10.0);
        }
        if (i < vsurfdata.size() - 1) {
            File.Fprintf(",\n");
        } else {
            File.Fprintf("\n");
        }
            
    }
    if (strcmp(datafmt, "xml") == 0)    File.Fprintf("</data>\n");
    if (strcmp(datafmt, "json") == 0)    File.Fprintf("]}\n");

    // 关闭文件。
    File.CloseAndRename();

    logfile.Write("生成数据文件%s成功，数据时间%s，记录数%d。\n",strFileName, strddatetime, vsurfdata.size());

    return true;
}
```



### 2.7 Linux信号





### 2.8 Linux多进程

+ idle进程：系统创建的第一个进程，加载系统
+ systemd进程：系统初始化，是所有其它用户进程的祖先(init)
+ kthreadd进程：负责所有内核线程的调度和管理
+ 

fork函数

`一个现有的进程调用fork()创建一个新的进程。子进程和父进程继续执行fork()函数后的代码。fork（）调用一次，返回两次。`

+ **子进程返回0，父进程返回子进程的进程ID（创建子进程失败，父进程返回-1）**
+ 子进程是父进程的副本
+ **子进程获得了父进程的数据空间、堆和栈的副本，不是共享**
+ 如果父进程先退出，子进程会成为**孤儿进程**，被1号进程收养
+ 如果子进程先退出，内核会向父进程发送SIGCHLD信号。若父进程不处理这个信号，子进程会成为**僵尸进程**

```
如果子进程在父进程之前终止，内核为每个子进程保留了一个数据结构，包括进程编号、终止状态和使用cpu时间等，父进程如果处理了子进程退出的信息，内核就会释放这个数据结构，如果父进程没有处理子进程退出的信息，内核就不会释放这个数据结构，子进程进程编号就会一直被占用，但是系统可用的进程号是有限的，如果大量的产生僵死进程，将因为没有可用的进程号而导致系统不能产生新的进程，这就是僵尸进程的危害。
```



**解决僵尸进程的三种方法**

+ 父进程忽略SIGCHLD信号，让内核处理

```c
signal(SIGCHLD, SIG_IGN);
```

+ 在父进程中调用wait函数等待子进程退出

```c
if (pid > 0) {
  	int sts;
	wait(&sts);  // 但是wait函数会阻塞在这行
    /*
     * 其它操作
    */
}
```

+ 捕获SIGCHLD信号，当信号到达时调用wait函数

```c
void func(int sig) {
    itn sts;
    wait(&sts);
}

signal(SIGCHLD, func);
```



### 2.9 服务程序的调度

```
先执行fork函数，创建一个子进程，让子进程调用execl执行新的程序
新程序将替换子进程，不会影响父进程
在父进程中，可以调用wait函数等待新程序运行的结果，这样就可以实现调度的功能
```





```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>
#include <signal.h>

int main(int argc, char* argv[]) {
    if (argc < 3) {
        printf("Using:./procctl timetvl program argv ...\n");
        printf("Example:/project/tools1/bin/procctl 5 /usr/bin/tar zcvf /tmp/tmp.tgz /usr/include\n\n");

        printf("本程序是服务程序的调度程序，周期性启动服务程序或shell脚本。\n");
        printf("timetvl 运行周期，单位：秒。被调度的程序运行结束后，在timetvl秒后会被procctl重新启动。\n");
        printf("program 被调度的程序名，必须使用全路径。\n");
        printf("argvs   被调度的程序的参数。\n");
        printf("注意，本程序不会被kill杀死，但可以用kill -9强行杀死。\n\n\n");

        return -1;
    }

    // 关闭信号和io，本程序不希望被打扰
    for (int i = 0; i < 64; ++i) {
        signal(i, SIG_IGN);
        close(i);
    }

    // 生成子进程，父进程退出，让子进程运行在后台，由系统1号进程托管
    if (fork() != 0)    exit(0);

    // 启用SIGCHLD信号，让父进程可以wait子进程退出的状态
    signal(SIGCHLD, SIG_DFL);

    // 先执行fork函数，创建一个子进程，让子进程调用execl执行新的程序
    // 新程序将替换子进程，不会影响父进程
    // 在父进程中，可以调用wait函数等待新程序运行的结果，这样就可以实现调度的功能
    // ./procctl 5 /usr/bin/ls -lt /tmp
    char* pargv[argc];
    for (int i = 2; i < argc; ++i) {
        pargv[i - 2] = argv[i];
    }
    pargv[argc - 2] = NULL;

    while (1) {
        if (fork() == 0) {
            //execl("/usr/bin/ls", "/usr/bin/ls", "-lt", "/tmp", (char*)0);
            execv(argv[2], pargv);
            exit(0);    // 当execv函数调用失败时退出程序
        } else {
            int status;
            wait(&status);
            sleep(atoi(argv[1]));
        }
    }
   
    return 0;
}
```



### 2.10 Linux共享内存

1. 调用shmget函数获取或创建共享内存
2. 调用shmat函数把共享内存连接到当前进程的地址空间
3. 调用shmdt函数把共享内存从当前进程中分离
4. 调用shmctl函数删除共享内存

```bash
ipcs -m		// 查看共享内存

ipcrm -m (shmid)	// 删除shmid
```





```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/ipc.h>
#include <sys/shm.h>

struct st_pid {
    int pid;        // 进程编号
    char name[51];  // 进程名称
};

int main(int argc, char* argv[]) {
    // 共享内存的标志
    int shmid;
    // 获取或者创建共享内存，键值位0x5005
    shmid = shmget(0x5005, sizeof(st_pid), 0640|IPC_CREAT);

    if (shmid == -1) {
        perror("shmget(0x5005) failed\n");
        return -1;
    }

    // 用于指向共享内存的结构体变量
    st_pid* stpid;
    // 把共享内存连接到当前进程的地址空间
    stpid = (st_pid*)shmat(shmid, NULL, 0);

    if (stpid == (void*)-1) {
        perror("shmat failed\n");
        return -1;
    }

    printf("pid: %d, name: %s\n", stpid->pid, stpid->name);
    stpid->pid = getpid();
    strcpy(stpid->name, argv[1]);
    printf("pid: %d, name: %s\n", stpid->pid, stpid->name);

    // 把共享内存从当前进程中分离
    shmdt(stpid);

    // 删除共享内存
    if (shmctl(shmid, IPC_RMID, 0) == -1) {
        perror("shmct failed\n");
        return -1;
    }

    return 0;
}
```



### 2.11-1 Linux信号量



### 2.12-2 Linux信号量

```bash
ipcs -s 	// 查看信号量

ipcrm sem (semid)  // 删除信号量
```



```c++
#include "_public.h"

CSEM sem; // 用于给共享内存加锁的信号量

struct st_pid {
    int pid;        // 进程编号
    char name[51];  // 进程名称
};

int main(int argc, char* argv[]) {
    if (argc < 2) {
        printf("Using: ./book procname\n");
        return 0;
    }    
    // 共享内存的标志
    int shmid;
    // 获取或者创建共享内存，键值位0x5005
    shmid = shmget(0x5005, sizeof(st_pid), 0640|IPC_CREAT);

    if (shmid == -1) {
        perror("shmget(0x5005) failed\n");
        return -1;
    }

    // 如果信号量已经存在，获取信号量；
    // 如果信号量不存在，则创建它，并初始化位value
    if (sem.init(0x5005) == false) {
        printf("sem.init(0x5005) failed\n");
        return -1;
    }

    // 用于指向共享内存的结构体变量
    st_pid* stpid;
    // 把共享内存连接到当前进程的地址空间
    stpid = (st_pid*)shmat(shmid, NULL, 0);

    if (stpid == (void*)-1) {
        perror("shmat failed\n");
        return -1;
    }

    printf("aaaa time = %d, val = %d\n", time(0), sem.value());
    sem.P();        // 加锁
    printf("bbbb time = %d, val = %d\n", time(0), sem.value());
    printf("pid: %d, name: %s\n", stpid->pid, stpid->name);
    stpid->pid = getpid();          // 进程编号
    sleep(10);
    strcpy(stpid->name, argv[1]);   // 进程名称
    printf("pid: %d, name: %s\n", stpid->pid, stpid->name);
    printf("cccc time = %d, val = %d\n", time(0), sem.value());
    sem.V();        // 解锁
    printf("dddd time = %d, val = %d\n", time(0), sem.value());
    // 把共享内存从当前进程中分离
    shmdt(stpid);

    // 删除共享内存
    if (shmctl(shmid, IPC_RMID, 0) == -1) {
        perror("shmct failed\n");
        return -1;
    }

    return 0;
}
```



### 2.13-1 Linux的心跳机制

+ 服务程序在共享内存中维护自己的心跳信息
+ 开发守护程序，终止已经死机的服务程序

```c++
#include "_public.h"

#define MAXNUMP_	1000	// 最大的进程数量
#define SHMKEYP_	0x5095	// 共享内存的key
#define SEMKEYP_	0x5095  // 信号量的key
// 进程心跳信息的结构体
struct st_pinfo
{
	int pid;		// 进程id
	char pname[51]; // 进程名称，可以为空
	int timeout;	// 超时时间，单位:秒
	time_t atime;	// 最后一次心跳的时间，用整数表示
};

int main(int argc, char* argv[])
{
	if (argc < 2) {
		printf("Using ./book procname\n");
		return 0;
	}

	// 创建/获取共享内存，大小为n*sizeof（struct st_pinfo）
	int m_shmid = 0;
	if ((m_shmid = shmget(SHMKEYP_, MAXNUMP_*sizeof(struct st_pinfo), 0640|IPC_CREAT)) == -1) {
		printf("shmget(%x) failed.\n", SHMKEYP_);
		return -1;
	}

	CSEM m_sem;
	if ((m_sem.init(SEMKEYP_)) == false) {
		printf("m_sem.init(%x) failed\n", SEMKEYP_);
		return -1;
	}

	// 将共享内存连接到当前进程的地址空间
	shmat(0x5005, (void*)0, 0);
	struct st_pinfo* m_shm = (struct st_pinfo*) shmat(m_shmid, NULL, 0);

	// 创建当前进程心跳信息结构体变量，把本进程的信息填进去
	struct st_pinfo stpinfo;
	memset(&stpinfo, 0, sizeof(struct st_pinfo));
	stpinfo.pid = getpid();				// 进程id
	STRNCPY(stpinfo.pname, sizeof(stpinfo.pname), argv[1], 50); // 进程名称
	stpinfo.timeout = 30;				// 超时时间，单位:秒
	stpinfo.atime = time(0);			// 最后一次心跳的时间，填当前时间

	int m_pos = -1;
	// 如果共享内存中存在当前进程编号，一定是其他进程残留的数据，当前进程就重用该位置
	for (int i = 0; i < MAXNUMP_; ++i) {
		if (m_shm[i].pid == stpinfo.pid) {
			m_pos = i;
			break;
		}
	}

	m_sem.P();	// 加锁
	// 在共享内存中查找一个空位置，把当前进程的心跳信息存入共享内存中
	if (m_pos == -1) {
		for (int i = 0; i < MAXNUMP_; ++i) {
			//if ((m_shm + i)->pid == 0) {
			if ((m_shm[i].pid == 0)) {
				m_pos = i;
				break;
			}
		}
	}
	if (m_pos == -1) {
		m_sem.V();	// 解锁
		printf("共享内存空间已用完。\n");
		return -1;
	}

	// 把当前进程的心跳信息存入共享内存的进程组中
	memcpy(m_shm + m_pos, &stpinfo, sizeof(struct st_pinfo));

	m_sem.V();	// 解锁

	while (1) {
		// 更新共享内存中本进程的心跳时间
		m_shm[m_pos].atime = time(0);
		sleep(10);
	}
	
	// 把当前进程从共享内存中移去
	m_shm[m_pos].pid = 0;
	memset(m_shm+m_pos, 0, sizeof(struct st_pinfo));
	
	// 把共享内存从当前进程中分离
	shmdt(m_shm);
	return 0;
}
```



### 2.14-2 Linux的心跳机制

```c++
#include "_public.h"

int main(int argc, char* argv[]) {
    if (argc < 2) {
        printf("Using: ./book procname\n");
        return 0;
    }  

    CPActive PActive;   // 进程心跳操作类

    PActive.AddPInfo(30, argv[1]); // 把当前进程的心跳信息加入共享内存进程组中

    while (1) {
        PActive.UptATime();   // 更新共享内存进程组中当前进程的心跳时间

        sleep(10);
    }

    return 0;
}
```



### 2.15-1 守护进程的实现

```c++
#include "_public.h"

// 程序运行的日志
CLogFile logfile;

int main(int argc, char* argv[])
{
    // 程序的帮助
    if (argc != 2) {
        printf("\n");
        printf("Using: ./checkproc logfilename\n");

        printf("Example: /project/tools1/bin/procctl 10 /project/tools1/bin/checkproc /tmp/log/checkproc.log\n\n");

        printf("本程序用于检查后台服务程序是否超时，如果已经超时，就终止它。\n");
        printf("注意:\n");
        printf("  1)本程序由procctl启动，运行周期建议为10秒。\n ");
        printf("  2)为了避免被普通用户误杀，本程序应该由root用户启动。\n\n\n");

        return 0;
    }

    // 打开日志文件
    if (logfile.Open(argv[1], "a+") == false) {
        printf("logfile.Open(%s) failed.\n", argv[1]);
        return -1;
    }

    int shmid = 0;
    // 创建/获取共享内存
    if ((shmid = shmget((key_t)SHMKEYP, MAXNUMP*sizeof(struct st_procinfo), 0666|IPC_CREAT)) == -1) {
        logfile.Write("创建/获取共享内存(%X)失败。\n", SHMKEYP);

        return false;
    }

    // 将共享内存连接到当前进程的地址空间
    struct st_procinfo *shm = (struct st_procinfo*) shmat(shmid, NULL, 0);

    // 遍历共享内存中全部的进程心跳记录
    for (int i = 0; i < MAXNUMP; ++i) {
        // 如果记录的pid==0，表示空记录，continue
        if (shm[i].pid == 0) continue;

        // 如果记录的pid!=0，表示是服务程序的心跳记录
        logfile.Write("i=%d, pid=%d, pname=%s, timeout=%d, atime=%d\n", i, shm[i].pid,
                       shm[i].pname, shm[i].timeout, shm[i].atime);

        // 向进程发送信号0，判断它是否还存在，如果不存在，从共享内存中删除该记录，continue

        // 如果进程未超时，continue

        // 如果已超时

        // 发送信号15，尝试正常终止进程

        // 如果进程仍存在，就发送信号9，强制终止它

        // 从共享内存中删除已超时进程的心跳记录

    }
    // 把共享内存从当前进程中分离


    return 0;
}
```



### 2.16-2 守护进程的实现

+ exit函数不会调用局部对象的析构函数
+ exit函数会调用全局对象的析构函数
+ return会调用局部和全局对象的析构函数

```c++
#include "_public.h"

// 程序运行的日志
CLogFile logfile;

int main(int argc, char* argv[])
{
    // 程序的帮助
    if (argc != 2) {
        printf("\n");
        printf("Using: ./checkproc logfilename\n");

        printf("Example: /project/tools1/bin/procctl 10 /project/tools1/bin/checkproc /tmp/log/checkproc.log\n\n");

        printf("本程序用于检查后台服务程序是否超时，如果已经超时，就终止它。\n");
        printf("注意:\n");
        printf("  1)本程序由procctl启动，运行周期建议为10秒。\n");
        printf("  2)为了避免被普通用户误杀，本程序应该由root用户启动。\n");
        printf("  3)如果要停止本程序，只能用killall -9 终止。\n\n\n");

        return 0;
    }

    // 忽略全部的信号和IO，不希望程序被干扰
    CloseIOAndSignal(true);

    // 打开日志文件
    if (logfile.Open(argv[1], "a+") == false) {
        printf("logfile.Open(%s) failed.\n", argv[1]);
        return -1;
    }

    int shmid = 0;
    // 创建/获取共享内存
    if ((shmid = shmget((key_t)SHMKEYP, MAXNUMP*sizeof(struct st_procinfo), 0666|IPC_CREAT)) == -1) {
        logfile.Write("创建/获取共享内存(%X)失败。\n", SHMKEYP);

        return false;
    }

    // 将共享内存连接到当前进程的地址空间
    struct st_procinfo *shm = (struct st_procinfo*) shmat(shmid, NULL, 0);

    // 遍历共享内存中全部的进程心跳记录
    for (int i = 0; i < MAXNUMP; ++i) {
        // 如果记录的pid==0，表示空记录，continue
        if (shm[i].pid == 0) continue;

        // 如果记录的pid!=0，表示是服务程序的心跳记录
        // logfile.Write("i=%d, pid=%d, pname=%s, timeout=%d, atime=%d\n", i, shm[i].pid,
        //                shm[i].pname, shm[i].timeout, shm[i].atime);

        // 向进程发送信号0，判断它是否还存在，如果不存在，从共享内存中删除该记录，continue
        int iret = kill(shm[i].pid, 0);
        if (iret == -1) {
            logfile.Write("进程pid=%d(%s)已经不存在。\n", (shm+i)->pid, (shm+i)->pname);
            memset(shm+i, 0, sizeof(struct st_procinfo));   // 从共享内存中删除该记录
            continue;
        }
        
        time_t now = time(0);
        // 如果进程未超时，continue
        if (now - shm[i].atime < shm[i].timeout) continue;

        // 如果已超时
        logfile.Write("进程pid=%d(%s)已经超时。\n", (shm+i)->pid, (shm+i)->pname);

        // 发送信号15，尝试正常终止进程
        kill(shm[i].pid, 15);

        // 每隔1秒判断一次进程是否存在，累计5秒，一般来说，5秒的时间足够让进程退出
        for (int j = 0; j < 5; ++j) {
            sleep(1);
            iret= kill(shm[i].pid, 0);  // 向进程发送信号0，判断它是否还存在
            if (iret == -1) break;
        }

        // 如果进程仍存在，就发送信号9，强制终止它
        if (iret == -1)
            logfile.Write("进程pid=%d(%s)已经正常终止。\n", (shm+i)->pid, (shm+i)->pname);
        else {
            kill(shm[i].pid, 9);
            logfile.Write("进程pid=%d(%s)已经强制终止。\n", (shm+i)->pid, (shm+i)->pname);
        }

        // 从共享内存中删除已超时进程的心跳记录
        memset(shm+i, 0, sizeof(struct st_procinfo));   // 从共享内存中删除该记录
    }

    // 把共享内存从当前进程中分离
    shmdt(shm);

    return 0;
}
```



### 2.17 完善生成测试数据程序

```c++
// 增加生成历史数据文件的功能，为压缩文件和清理文件模块准备历史数据文件
// 增加信号处理函数，处理2和15的信号
// 解决调用exit函数退出时局部对象没有调用析构函数的问题
// 把心跳信息写入共享内存

#include "_public.h"

CPActive PActive;   // 进程心跳

// 全国气象站点参数结构体
struct st_stcode {
    char provname[31]; // 省
    char obtid[11];    // 站号
    char obtname[31];  // 站名
    double lat;        // 纬度
    double lon;        // 经度
    double height;     // 海拔高度
};

// 存放全国气象站点参数的容器
vector<st_stcode> vstcode;

// 把站点参数文件中加载到vstcode容器中
bool LoadSTCode(const char* inifile);


// 全国气象站点分钟观测数据结构
struct st_surfdata {
  char obtid[11];      // 站点代码。
  char ddatetime[21];  // 数据时间：格式yyyymmddhh24miss
  int  t;              // 气温：单位，0.1摄氏度。
  int  p;              // 气压：0.1百帕。
  int  u;              // 相对湿度，0-100之间的值。
  int  wd;             // 风向，0-360之间的值。
  int  wf;             // 风速：单位0.1m/s
  int  r;              // 降雨量：0.1mm。
  int  vis;            // 能见度：0.1米。
};

vector<st_surfdata> vsurfdata; // 存放全国气象站点分钟观测数据的容器

char strddatetime[21]; // 观测数据的时候

// 模拟生成全国气象站点分钟观测数据，存放在vsurfdata容器中
void CrtSurfData();

CFile File;

// 把容器vsurfdata中的全国气象站点分钟观测数据写入文件
bool CrtSurfFile(const char* outpath, const char* datafmt);

CLogFile logfile;   // 日志类

void EXIT(int sig); // 程序退出和信号2、15的处理函数

int main(int argc, char* argv[]) {

    // inifile outpath logfile
    if ((argc != 5) && (argc != 6)) {
        printf("Using: ./crtsurfdata inifile outpath logfile datafmt [datetime]\n");
        printf("Example: /project/idc1/bin/crtsurfdata /project/idc1/ini/stcode.ini "
            "/tmp/surfdata /log/idc/crtsurfdata.log xml,json,csv\n");
        printf("         /project/idc1/bin/crtsurfdata /project/idc1/ini/stcode.ini "
            "/tmp/surfdata /log/idc/crtsurfdata.log xml,json,csv 20210710123000\n");
        printf("         /project/tools1/bin/procctl 60 /project/idc1/bin/crtsurfdata /project/idc1/ini/stcode.ini "
            "/tmp/surfdata /log/idc/crtsurfdata.log xml,json,csv\n\n");
        printf("inifile 全国气象站点参数文件名\n");
        printf("outpath 全国气象站点数据文件存放的目录\n");
        printf("logfile 本程序运行时的日志文件名\n");
        printf("datafmt 生成程序文件的格式，支持xml、json和csv三种格式，中间用逗号分隔\n");
        printf("datetime 这是一个可选参数，表示生产指定时间的数据和文件。\n\n\n");

        return -1;
    }

    // 关闭全部的信号和输入输出
    // 设置信号，在shell状态下可用“kill + 进程号”正常终止进程
    // 但请不要用“kill -9 + 进程号”强行终止
    CloseIOAndSignal(true);
    signal(SIGINT, EXIT);
    signal(SIGTERM, EXIT);

    if (logfile.Open(argv[3], "a+", false) == false) {
        perror("open failed");
        return -1;
    }

    logfile.Write("crtsurfdata 开始运行。\n");

    PActive.AddPInfo(20, "crtsurfdata");

    // 把站点参数文件中加载到vstcode容器中
    if (LoadSTCode(argv[1]) == false) {
        return -1;
    }

    // 获取当前时间，当作观测时间。
    memset(strddatetime, 0, sizeof(strddatetime));
    if (argc == 5)
        LocalTime(strddatetime, "yyyymmddhh24miss");
    else
        STRCPY(strddatetime, sizeof(strddatetime), argv[5]);

    // 模拟生成全国气象站点分钟观测数据，存放在surfdata容器中
    CrtSurfData();

    // 把容器vsurfdata中全国气象站点分钟观测数据写入文件
    if (strstr(argv[4], "xml") != 0)    CrtSurfFile(argv[2], "xml");
    if (strstr(argv[4], "json") != 0)    CrtSurfFile(argv[2], "json");
    if (strstr(argv[4], "csv") != 0)    CrtSurfFile(argv[2], "csv");

    logfile.Write("crtsurfdata 结束运行。\n");

    return 0;
}

bool LoadSTCode(const char* inifile) {
    // 打开站点参数文件
    if (File.Open(inifile, "r") == false) {
        logfile.Write("File.Open(%s) failed.\n", inifile);
        return -1;
    }

    char strBuffer[301];

    CCmdStr CmdStr;

    st_stcode stcode;

    while (1) {
        // 从站点参数文件中读取一行。如果已经读取完，跳出循环
        if (File.Fgets(strBuffer, 300, true) == false) {
            break;
        }

        // 把读取到的一行拆分
        CmdStr.SplitToCmd(strBuffer, ",", true);

        if ((CmdStr.CmdCount() != 6)) {
            continue; // 丢掉无效的行
        }

        // 把站点参数的每个数据项保存到站点参数结构体中
        CmdStr.GetValue(0, stcode.provname, 30);   // 省
        CmdStr.GetValue(1, stcode.obtid, 10);      // 站号
        CmdStr.GetValue(2, stcode.obtname, 30);    // 站名
        CmdStr.GetValue(3, &stcode.lat);           // 纬度
        CmdStr.GetValue(4, &stcode.lon);           // 经度
        CmdStr.GetValue(5, &stcode.height);        // 海拔高度

        // 把站点参数结构体放入站点参数容器
        vstcode.push_back(stcode);
    }

    return true;
}

void CrtSurfData() {
    // 播随机数种子。
    srand(time(0));

    st_surfdata stsurfdata;

    // 遍历气象站点参数的vstcode容器。
    for (int i = 0; i < vstcode.size(); ++i) {
        memset(&stsurfdata, 0, sizeof(st_surfdata));

        // 用随机数填充分钟观测数据的结构体。
        strncpy(stsurfdata.obtid, vstcode[i].obtid, 10); // 站点代码。
        strncpy(stsurfdata.ddatetime, strddatetime,14);  // 数据时间：格式yyyymmddhh24miss
        stsurfdata.t = rand()%351;       // 气温：单位，0.1摄氏度
        stsurfdata.p = rand()%265+10000; // 气压：0.1百帕
        stsurfdata.u = rand()%101;     // 相对湿度，0-100之间的值。
        stsurfdata.wd = rand()%361;      // 风向，0-360之间的值。
        stsurfdata.wf = rand()%150;      // 风速：单位0.1m/s
        stsurfdata.r = rand()%16;        // 降雨量：0.1mm
        stsurfdata.vis = rand()%5001+100000;  // 能见度：0.1米

        // 把观测数据的结构体放入vsurfdata容器。
        vsurfdata.push_back(stsurfdata);
    }
}

bool CrtSurfFile(const char* outpath, const char* datafmt) {
    CFile File;

    // 拼接生成数据的文件名，例如: SURF_ZH_20220412122707_11193.xml
    char strFileName[301];
    sprintf(strFileName, "%s/SURF_ZH_%s_%d.%s", outpath, strddatetime, getpid(), datafmt);
    
    // 打开文件
    if (File.OpenForRename(strFileName, "w") == false) {
        logfile.Write("File.OpenForRename(%s) failed.\n", strFileName);
        return -1;
    }

    // 写入第一行标题
    if (strcmp(datafmt, "csv") == 0) {
        File.Fprintf("站点代码, 数据时间, 气温, 气压, 相对湿度, 风向, 风速, 降雨量, 能见度\n");
    }
    if (strcmp(datafmt, "xml") == 0) {
        File.Fprintf("<data>\n");
    }
    if (strcmp(datafmt, "json") == 0) {
        File.Fprintf("{\"data\":[\n");
    }

    // 遍历存放观测数据的vsurfdata容器
    for (int i = 0; i < vsurfdata.size(); ++i) {
        // 写入一条记录
        if (strcmp(datafmt, "csv") == 0) {
            File.Fprintf("%s, %s, %.1f, %.1f, %d, %d, %.1f, %.1f, %.1f\n",\
                vsurfdata[i].obtid,vsurfdata[i].ddatetime,vsurfdata[i].t/10.0,vsurfdata[i].p/10.0,\
                vsurfdata[i].u,vsurfdata[i].wd,vsurfdata[i].wf/10.0,vsurfdata[i].r/10.0,vsurfdata[i].vis/10.0);
        }
        if (strcmp(datafmt, "xml") == 0) {
            File.Fprintf("<obtid>%s</obtid><ddatetime>%s</ddatetime><t>%.1f</t><p>%.1f</p><u>%d</u><wd>%d"
                "</wd><wf>%.1f</wf><r>%.1f</r><vis>%.1f</vis><endl/>\n",\
                vsurfdata[i].obtid,vsurfdata[i].ddatetime,vsurfdata[i].t/10.0,vsurfdata[i].p/10.0,\
                vsurfdata[i].u,vsurfdata[i].wd,vsurfdata[i].wf/10.0,vsurfdata[i].r/10.0,vsurfdata[i].vis/10.0);
        }
        if (strcmp(datafmt, "json") == 0) {
            File.Fprintf("{\"obtid\":\"%s\",\"ddatetime\":\"%s\",\"t\":\"%.1f\",\"p\":\"%.1f\","
                "\"u\":\"%d\",\"wd\":\"%d\",\"wf\":\"%.1f\",\"r\":\"%.1f\",\"vis\":\"%.1f\"}",\
                vsurfdata[i].obtid,vsurfdata[i].ddatetime,vsurfdata[i].t/10.0,vsurfdata[i].p/10.0,\
                vsurfdata[i].u,vsurfdata[i].wd,vsurfdata[i].wf/10.0,vsurfdata[i].r/10.0,vsurfdata[i].vis/10.0);
        
            if (i < vsurfdata.size() - 1) {
                File.Fprintf(",\n");
            } else {
                File.Fprintf("\n");
            }
        }
            
    }
    if (strcmp(datafmt, "xml") == 0)    File.Fprintf("</data>\n");
    if (strcmp(datafmt, "json") == 0)    File.Fprintf("]}\n");

    // 关闭文件。
    File.CloseAndRename();

    UTime(strFileName, strddatetime);   // 修改文件的时间属性

    logfile.Write("生成数据文件%s成功，数据时间%s，记录数%d\n",strFileName, strddatetime, vsurfdata.size());

    return true;
}

void EXIT(int sig)
{
    logfile.Write("程序退出，sig=%d\n\n", sig);
    exit(0);
}
```



### 2.18 压缩文件

```c++
#include "_public.h"

// 程序退出和信号2、15的处理函数
void EXIT(int sig);

int main(int argc, char* argv[])
{
    // 程序的帮助
    if (argc != 4) {
        printf("\n");
        printf("Using: /project/tools1/bin/gzipfiles pathname matchstr timeout\n\n");

        printf("Example: /project/tools1/bin/gzipfiles /log/idc \"*.log.20*\" 0.02\n");
        printf("         /project/tools1/bin/gzipfiles /tmp/idc/surfdata \"*.xml,*.json\" 0.01\n");
        printf("         /project/tools1/bin/procctl 300 /project/tools1/bin/gzipfiles /log/idc \"*.log.20*\" 0.02\n");
        printf("         /project/tools1/bin/procctl 300 /project/tools1/bin/gzipfiles /tmp/idc/surfdata \"*.xml,*.json\" 0.01\n");
    
        printf("这是一个工具程序，用于压缩历史的数据文件或日志文件。\n");
        printf("本程序把pathname目录及子目录中timeout天之前的匹配matchstr文件全部压缩，timeout可以是小数。\n");
        printf("本程序不写日志文件，也不会在控制台输出任何信息。\n");
        printf("本程序调用/usr/bin/gzip命令压缩文件。\n\n\n");

        return -1;
    }

    // 关闭全部的信号和输入输出
   // CloseIOAndSignal(true);
    signal(SIGINT, EXIT);
    signal(SIGTERM, EXIT);

    // 获取文件超时的时间点
    char strTimeOut[21];
    LocalTime(strTimeOut, "yyyy-mm-dd hh24:mi:ss", 0 - (int)(atof(argv[3])*24*60*60));

    CDir Dir;
    // 打开目录
    if (Dir.OpenDir(argv[1], argv[2], 10000, true) == false) {
        printf("Dir.Open(%s) failed.\n", argv[1]);
        return -1;
    }
    
    char strCmd[1024];  // 存放gzip压缩文件的命令

    // 遍历目录中的文件名
    while (1) {
        // 得到一个文件的信息
        if (Dir.ReadDir() == false) break;

        //printf("FullFileName=%s\n", Dir.m_FullFileName);

        // 与超时的时间点比较，如果更早，就需要压缩
        if ((strcmp(Dir.m_ModifyTime, strTimeOut) < 0) && (MatchStr(Dir.m_FileName, "*.gz") == false)) {
            // 压缩文件，调用操作系统的gzip命令
            SNPRINTF(strCmd, sizeof(strCmd), 1000, "/usr/bin/gziop -f %s 1>/dev/nul 2>/dev/null", Dir.m_FullFileName);
            if (system(strCmd) == 0)
                printf("gzip %s ok.\n", Dir.m_FullFileName);
            else
                printf("gzip %s failed.\n", Dir.m_FullFileName);
        }
    }
    return 0;
}

void EXIT(int sig)
{
    printf("程序退出，sig=%d\n\n", sig);
    exit(0);
}
```



### 2.19 清理文件

```c++
#include "_public.h"

// 程序退出和信号2、15的处理函数
void EXIT(int sig);

int main(int argc, char* argv[])
{
    // 程序的帮助
    if (argc != 4) {
        printf("\n");
        printf("Using: /project/tools1/bin/deletefiles pathname matchstr timeout\n\n");

        printf("Example: /project/tools1/bin/deletefiles /log/idc \"*.log.20*\" 0.02\n");
        printf("         /project/tools1/bin/deletefiles /tmp/idc/surfdata \"*.xml,*.json\" 0.01\n");
        printf("         /project/tools1/bin/procctl 300 /project/tools1/bin/deletefiles /log/idc \"*.log.20*\" 0.02\n");
        printf("         /project/tools1/bin/procctl 300 /project/tools1/bin/deletefiles /tmp/idc/surfdata \"*.xml,*.json\" 0.01\n");
    
        printf("这是一个工具程序，用于删除历史的数据文件或日志文件。\n");
        printf("本程序把pathname目录及子目录中timeout天之前的匹配matchstr文件全部删除，timeout可以是小数。\n");
        printf("本程序不写日志文件，也不会在控制台输出任何信息。\n\n\n");

        return -1;
    }

    // 关闭全部的信号和输入输出
   // CloseIOAndSignal(true);
    signal(SIGINT, EXIT);
    signal(SIGTERM, EXIT);

    // 获取文件超时的时间点
    char strTimeOut[21];
    LocalTime(strTimeOut, "yyyy-mm-dd hh24:mi:ss", 0 - (int)(atof(argv[3])*24*60*60));

    CDir Dir;
    // 打开目录
    if (Dir.OpenDir(argv[1], argv[2], 10000, true) == false) {
        printf("Dir.Open(%s) failed.\n", argv[1]);
        return -1;
    }

    // 遍历目录中的文件名
    while (1) {
        // 得到一个文件的信息
        if (Dir.ReadDir() == false) break;

        //printf("FullFileName=%s\n", Dir.m_FullFileName);

        // 与超时的时间点比较，如果更早，就需要删除
        if (strcmp(Dir.m_ModifyTime, strTimeOut) < 0) {
            if (REMOVE(Dir.m_FullFileName) == true)
                printf("REMOVE %s ok.\n", Dir.m_FullFileName);
            else
                printf("REMOVE %s failed.\n", Dir.m_FullFileName);
        }
    }
    return 0;
}

void EXIT(int sig)
{
    printf("程序退出，sig=%d\n\n", sig);
    exit(0);
}
```



### 2.20 服务程序的运行策略

`vim /etc/rc.local`

```c++

```



### 2.21 本章总结

开发任务

+ 生成测试数据
+ 调度程序
+ 守护程序
+ 文件的压缩和清理



## 三、开发基于ftp协议的文件传输子系统

### 3-1 开篇语

#### 一、ftp的基础知识

#### 二、ftp客户端的封装

#### 三、文件下载功能的实现

#### 四、文件上传功能的实现



### 3-2 课件预习

### 3-3 ftp客户端的封装

### 3-4 ftp下载文件-搭建程序的框架

```c++
//#include "_public.h"
#include  "_ftp.h"

struct st_arg {
  char host[31];           // 远程服务器的IP和端口。
  int  mode;               // 传输模式，1-被动模式，2-主动模式，缺省采用被动模式。
  char username[31];       // 远程服务器ftp的用户名。
  char password[31];       // 远程服务器ftp的密码。
  char remotepath[301];    // 远程服务器存放文件的目录。
  char localpath[301];     // 本地文件存放的目录。
  char matchname[101];     // 待下载文件匹配的规则。
} starg;

CLogFile logfile;

Cftp ftp;

// 程序退出和信号2、15的处理函数
void EXIT(int sig);

void _help();

// 把xml解析到参数starg结构中。
bool _xmltoarg(char *strxmlbuffer);

int main(int argc, char* argv[])
{
    // 小目标，把ftp服务器上某目录中的文件下载到本地的目录中
    if (argc != 3) {
       _help();
       return -1;
    }
    // 处理程序的退出信号
    // 关闭全部的信号和输入输出。
    // 设置信号,在shell状态下可用 "kill + 进程号" 正常终止些进程。
    // 但请不要用 "kill -9 +进程号" 强行终止。
    // CloseIOAndSignal(); 
    signal(SIGINT,EXIT); signal(SIGTERM,EXIT);
    
    // 打开日志文件
    if (logfile.Open(argv[1], "a+") == false) {
        printf("打开日志文件失败（%s）。\n", argv[1]);
        return -1;
    }

    // 解析xml，得到程序运行的参数
    if (_xmltoarg(argv[2]) == false) return -1;

    // 登录ftp服务器

    // 进入ftp服务器存放文件的目录

    // 调用ftp.nlist()方法列出服务器目录中的文件，结果存放到本地文件中

    // 把ftp.nlis()方法获取到的list文件加载到容器vfilelist中

    // 遍历容器vfilelist
    //for (int i = 0; i < vfilelist.size(); ++i) {
        // 调用ftp.get()方法从服务器下载文件
    //}

    // ftp.logout();

    return 0;
}

void EXIT(int sig)
{
    printf("程序退出, sig=%d\n\n", sig);
    exit(0);
}

void _help()
{
    printf("\n");
    printf("Using:/project/tools1/bin/ftpgetfiles logfilename xmlbuffer\n\n");

    printf("Example:/project/tools1/bin/procctl 30 /project/tools1/bin/ftpgetfiles /log/idc/ftpgetfiles_surfdata.log \"<host>127.0.0.1:21</host><mode>1</mode><username>wucz</username><password>wuczpwd</password><localpath>/idcdata/surfdata</localpath><remotepath>/tmp/idc/surfdata</remotepath><matchname>SURF_ZH*.XML,SURF_ZH*.CSV</matchname>\"\n\n\n");

    printf("本程序是通用的功能模块，用于把远程ftp服务器的文件下载到本地目录。\n");
    printf("logfilename是本程序运行的日志文件。\n");
    printf("xmlbuffer为文件下载的参数，如下：\n");
    printf("<host>127.0.0.1:21</host> 远程服务器的IP和端口。\n");
    printf("<mode>1</mode> 传输模式，1-被动模式，2-主动模式，缺省采用被动模式。\n");
    printf("<username>wucz</username> 远程服务器ftp的用户名。\n");
    printf("<password>wuczpwd</password> 远程服务器ftp的密码。\n");
    printf("<remotepath>/tmp/idc/surfdata</remotepath> 远程服务器存放文件的目录。\n");
    printf("<localpath>/idcdata/surfdata</localpath> 本地文件存放的目录。\n");
    printf("<matchname>SURF_ZH*.XML,SURF_ZH*.CSV</matchname> 待下载文件匹配的规则。"\
        "不匹配的文件不会被下载，本字段尽可能设置精确，不建议用*匹配全部的文件。\n\n\n");

}

// 把xml解析到参数starg结构中。
bool _xmltoarg(char *strxmlbuffer)
{
  memset(&starg,0,sizeof(struct st_arg));

  GetXMLBuffer(strxmlbuffer,"host",starg.host,30);   // 远程服务器的IP和端口。
  if (strlen(starg.host)==0)
  { logfile.Write("host is null.\n");  return false; }

  GetXMLBuffer(strxmlbuffer,"mode",&starg.mode);   // 传输模式，1-被动模式，2-主动模式，缺省采用被动模式。
  if (starg.mode!=2)  starg.mode=1;

  GetXMLBuffer(strxmlbuffer,"username",starg.username,30);   // 远程服务器ftp的用户名。
  if (strlen(starg.username)==0)
  { logfile.Write("username is null.\n");  return false; }

  GetXMLBuffer(strxmlbuffer,"password",starg.password,30);   // 远程服务器ftp的密码。
  if (strlen(starg.password)==0)
  { logfile.Write("password is null.\n");  return false; }

  GetXMLBuffer(strxmlbuffer,"remotepath",starg.remotepath,300);   // 远程服务器存放文件的目录。
  if (strlen(starg.remotepath)==0)
  { logfile.Write("remotepath is null.\n");  return false; }

  GetXMLBuffer(strxmlbuffer,"localpath",starg.localpath,300);   // 本地文件存放的目录。
  if (strlen(starg.localpath)==0)
  { logfile.Write("localpath is null.\n");  return false; }

  GetXMLBuffer(strxmlbuffer,"matchname",starg.matchname,100);   // 待下载文件匹配的规则。
  if (strlen(starg.matchname)==0)
  { logfile.Write("matchname is null.\n");  return false; }

  return true;
}
```



### 3-5 ftp下载文件-下载全部的文件



### 3-6 ftp下载文件-清理和转存文件



### 3-7  ftp下载文件-下载新增的文件



### 3-8  ftp下载文件-下载修改的文件



### 3-9  ftp上传文件

```c++
//#include "_public.h"
#include  "_ftp.h"

struct st_arg {
    char host[31];           // 远程服务器的IP和端口。
    int  mode;               // 传输模式，1-被动模式，2-主动模式，缺省采用被动模式。
    char username[31];       // 远程服务器ftp的用户名。
    char password[31];       // 远程服务器ftp的密码。
    char remotepath[301];    // 远程服务器存放文件的目录。
    char localpath[301];     // 本地文件存放的目录。
    char matchname[101];     // 待上传文件匹配的规则。
    int ptype;               // 上传后客户端文件的处理方式：1-什么也不做；2-删除；3-备份，如果为3，还需要指定备份的目录
    char localpathbak[301];  // 上传后客户端文件的备份目录
    char okfilename[301];    // 已上传成功文件名清单
    int timeout;             // 进程心跳的超时时间
    char pname[51];          // 进程名，建议用"ftpputfles_后缀"的方式

} starg;

struct st_fileinfo {
    char filename[301];    // 文件名
    char mtime[21];        // 文件时间
};

vector<struct st_fileinfo> vlistfile1;   // 已上传成功文件名的容器，从okfilename中加载
vector<struct st_fileinfo> vlistfile2;   // 上传前列出服务器文件名的容器，从nlist文件中加载
vector<struct st_fileinfo> vlistfile3;   // 本次不需要上传的文件的容器
vector<struct st_fileinfo> vlistfile4;   // 本次需要上传的文件的容器


// 加载okfilename文件中的内容到容器vlistfile1中
bool LoadOKFile();

// 比较vlistfile2和vlistfile1，得到vlistfile3和vlistfile4
bool CompVecotr();

// 把容器vlistfile3中的内容写入okfilename文件，覆盖之前的旧okfilename文件
bool WriteToOKFile();

// 如果ptype==1，把上传成功的文件记录追加到okfilename文件中
bool AppendToOKFile(struct st_fileinfo* st_fileinfo);

// 把localpath目录下的文件加载到容器vlistfile2中
bool LoadLocalFile();

CLogFile logfile;

Cftp ftp;

// 程序退出和信号2、15的处理函数
void EXIT(int sig);

void _help();

// 把xml解析到参数starg结构中。
bool _xmltoarg(char *strxmlbuffer);

// 上传文件功能的主函数
bool _ftpputfiles();

CPActive PActive;   // 进程心跳

int main(int argc, char* argv[])
{
    if (argc != 3) {
       _help();
       return -1;
    }
    // 处理程序的退出信号
    // 关闭全部的信号和输入输出。
    // 设置信号,在shell状态下可用 "kill + 进程号" 正常终止些进程。
    // 但请不要用 "kill -9 +进程号" 强行终止。
    //CloseIOAndSignal(); 
    signal(SIGINT,EXIT); signal(SIGTERM,EXIT);
    
    // 打开日志文件
    if (logfile.Open(argv[1], "a+") == false) {
        printf("打开日志文件失败（%s）。\n", argv[1]);
        return -1;
    }

    // 解析xml，得到程序运行的参数
    if (_xmltoarg(argv[2]) == false) return -1;

    PActive.AddPInfo(starg.timeout, starg.pname);   // 把进程的心跳信息写入共享内存

    // 登录ftp服务器
    if (ftp.login(starg.host, starg.username, starg.password, starg.mode) == false) {
        logfile.Write("ftp.login(%s,%s,%s) failed.\n", starg.host, starg.username, starg.password);
        return -1;
    }

    // logfile.Write("ftp.login ok.\n");  // 正式运行后，可以注释这行代码。

    _ftpputfiles();

    ftp.logout();

    return 0;
}

void EXIT(int sig)
{
    printf("程序退出, sig=%d\n\n", sig);
    exit(0);
}

void _help()
{
    printf("\n");
    printf("Using:/project/tools1/bin/ftpputfiles logfilename xmlbuffer\n\n");

    printf("Example:/project/tools1/bin/procctl 30 /project/tools1/bin/ftpputfiles "
        "/log/idc/ftpputfiles_surfdata.log \"<host>127.0.0.1:21</host><mode>1</mode>"
        "<username>yh</username><password>111111</password><localpath>/tmp/idc/surfdata</localpath>"
        "<remotepath>/tmp/ftpputtest</remotepath><matchname>SURF_ZH*.JSON</matchname><ptype>1</ptype>"
        "<localpathbak>/tmp/idc/surfdatabak</localpathbak><okfilename>/idcdata/ftplist/ftpputfiles_surfdata.xml</okfilename>"
        "<timeout>80</timeout><pname>ftpputfiles_surfdata</pname>\"\n\n\n");

    printf("本程序是通用的功能模块，用于把远程ftp服务器的文件上传到本地目录。\n");
    printf("logfilename是本程序运行的日志文件。\n");
    printf("xmlbuffer为文件上传的参数，如下：\n");
    printf("<host>127.0.0.1:21</host> 远程服务器的IP和端口。\n");
    printf("<mode>1</mode> 传输模式，1-被动模式，2-主动模式，缺省采用被动模式。\n");
    printf("<username>yh</username> 远程服务器ftp的用户名。\n");
    printf("<password>111111</password> 远程服务器ftp的密码。\n");
    printf("<remotepath>/tmp/ftpputtest</remotepath> 远程服务器存放文件的目录。\n");
    printf("<localpath>/tmp/idc/surfdata</localpath> 本地文件存放的目录。\n");
    printf("<matchname>SURF_ZH*.JSON</matchname> 待上传文件匹配的规则。"\
        "不匹配的文件不会被上传，本字段尽可能设置精确，不建议用*匹配全部的文件。\n");
    printf("<ptype>1</ptype> 文件上传成功后，远程服务器文件的处理方式："
        "1-什么也不做；2-删除；3-备份，如果为3，还需要指定备份的目录。\n");
    printf("<localpathbak>/tmp/idc/surfdatabak</localpathbak> 文件上传完成后，服务器文的备份目录，"
        "此参数只有当ptype=3时才有效。\n");
    printf("<okfilename>/idcdata/ftplist/ftpputfiles_surfdata.xml</okfilename> 已上传成功过的文件清单,"
        "此参数只有当ptype=1时才有效。\n");
    printf("<timeout>80</timeout> 上传文件超时时间，单位：秒，视文件大小和网络带宽而定。\n");
    printf("<pname>ftpputfiles_surfdata</pname> 进程名，尽可能采用易懂的、与其他进程不同的名称，方便故障排查。\n\n\n");

}

// 把xml解析到参数starg结构中。
bool _xmltoarg(char *strxmlbuffer)
{
  memset(&starg,0,sizeof(struct st_arg));

  GetXMLBuffer(strxmlbuffer,"host",starg.host,30);   // 远程服务器的IP和端口。
  if (strlen(starg.host)==0)
  { logfile.Write("host is null.\n");  return false; }

  GetXMLBuffer(strxmlbuffer,"mode",&starg.mode);   // 传输模式，1-被动模式，2-主动模式，缺省采用被动模式。
  if (starg.mode!=2)  starg.mode=1;

  GetXMLBuffer(strxmlbuffer,"username",starg.username,30);   // 远程服务器ftp的用户名。
  if (strlen(starg.username)==0)
  { logfile.Write("username is null.\n");  return false; }

  GetXMLBuffer(strxmlbuffer,"password",starg.password,30);   // 远程服务器ftp的密码。
  if (strlen(starg.password)==0)
  { logfile.Write("password is null.\n");  return false; }

  GetXMLBuffer(strxmlbuffer,"remotepath",starg.remotepath,300);   // 远程服务器存放文件的目录。
  if (strlen(starg.remotepath)==0)
  { logfile.Write("remotepath is null.\n");  return false; }

  GetXMLBuffer(strxmlbuffer,"localpath",starg.localpath,300);   // 本地文件存放的目录。
  if (strlen(starg.localpath)==0)
  { logfile.Write("localpath is null.\n");  return false; }

 GetXMLBuffer(strxmlbuffer,"matchname",starg.matchname,100);   // 待上传文件匹配的规则。
  if (strlen(starg.matchname)==0)
  { logfile.Write("matchname is null.\n");  return false; }

  GetXMLBuffer(strxmlbuffer,"ptype",&starg.ptype);
  if ((starg.ptype != 1) && (starg.ptype != 2) && (starg.ptype != 3))
  { logfile.Write("ptype is error.\n"); return false;   }

  GetXMLBuffer(strxmlbuffer,"localpathbak",starg.localpathbak,300);  // 上传后服务器文件的备份目录
  if ((starg.ptype == 3) && (strlen(starg.localpathbak) ==0))
  { logfile.Write("localpathbak is null.\n");  return false; }

  GetXMLBuffer(strxmlbuffer, "okfilename", starg.okfilename, 300);  // 已上传成功的文件名清单
  if ((starg.ptype == 1) && (strlen(starg.okfilename) ==0))
  { logfile.Write("okfilename is null.\n");  return false; }

  GetXMLBuffer(strxmlbuffer,"timeout",&starg.timeout);   // 进程的心跳时间
  if (starg.timeout==0)
  { logfile.Write("timeout is null.\n");  return false; }

  GetXMLBuffer(strxmlbuffer,"pname",starg.pname,50);   // 进程名
  if (strlen(starg.pname)==0)
  { logfile.Write("pname is null.\n");  return false; }

  return true;
}

bool _ftpputfiles()
{
    // 把localpath目录下的文件加载到容器vlistfile2中
    if (LoadLocalFile() == false) {
        logfile.Write("LoadLocalFile() failed.\n");
        return false;
    }

    PActive.UptATime(); // 更新进程的心跳

    if (starg.ptype == 1) {
        // 加载okfilename文件中的内容到容器vlistfile1中
        LoadOKFile();

        // 比较vlistfile2和vlistfile1，得到vlistfile3和vlistfile4
        CompVecotr();

        // 把容器vlistfile3中的内容写入okfilename文件，覆盖之前的旧okfilename文件
        WriteToOKFile();

        // 把vlistfile4中的内容复制到vlistfile2中
        vlistfile2.clear();
        vlistfile2.swap(vlistfile4);
    }

    PActive.UptATime(); // 更新进程的心跳    

    char strremotefilename[301], strlocalfilename[301];
    // 遍历容器vlistfile2
    for (int i = 0; i < vlistfile2.size(); ++i) {
        SNPRINTF(strremotefilename, sizeof(strremotefilename), 300, "%s/%s", starg.remotepath, vlistfile2[i].filename);
        SNPRINTF(strlocalfilename, sizeof(strlocalfilename), 300, "%s/%s", starg.localpath, vlistfile2[i].filename);
        
        logfile.Write("put %s ...", strlocalfilename);
        // 调用ftp.put()方法把文件上传到服务端
        if (ftp.put(strlocalfilename, strremotefilename, true) == false) {
            logfile.WriteEx("failed.\n");
            return false;
        }

        logfile.WriteEx("ok.\n");

        PActive.UptATime(); // 更新进程的心跳

        // 如果ptype==1，把上传成功的文件记录追加到okfilename文件中
        if (starg.ptype == 1)   AppendToOKFile(&vlistfile2[i]);

        // 删除文件
        if (starg.ptype == 2) {
            if (REMOVE(strlocalfilename) == false) {
                logfile.Write("REMOVE(%s) failed.\n", strlocalfilename);
                return false;
            }
        }

        // 转存到备份目录
        if (starg.ptype == 3) {
            char strlocalfilenamebak[301];
            SNPRINTF(strlocalfilenamebak, sizeof(strlocalfilenamebak), 300, "%s/%s", starg.localpathbak, vlistfile2[i].filename);
            if (RENAME(strlocalfilename, strlocalfilenamebak) == false) {
                logfile.Write("RENAME(%s,%s) faijled.\n", strlocalfilename, strlocalfilenamebak);
                return false;
            }
        }   

    }
    return true;
}

bool LoadLocalFile()
{
    vlistfile2.clear();

    CDir Dir;

    Dir.SetDateFMT("yyyymmddhh24miss");
    
    if (Dir.OpenDir(starg.localpath, starg.matchname) == false) {
        logfile.Write("Dir.Open(%s) 失败。\n", starg.localpath);
        return false;
    }

    struct st_fileinfo stfileinfo;

    while (true) {
        memset(&stfileinfo, 0, sizeof(struct st_fileinfo));

        if (Dir.ReadDir() == false) break;
        strcpy(stfileinfo.filename, Dir.m_FileName);
        strcpy(stfileinfo.mtime, Dir.m_ModifyTime);

        vlistfile2.push_back(stfileinfo);
    }
    // for (int i = 0; i < vlistfile2.size(); ++i) {
    //     logfile.Write("filename=%s=\n", vlistfile2[i].filename);
    // }
    return true;
}

// 加载okfilename文件中的内容到容器vlistfile1中
bool LoadOKFile()
{
    vlistfile1.clear();

    CFile File;

    // 注意：如果程序是第一次上传，okfilename是不存在的，并不是错误，若有也返回true
    if (File.Open(starg.okfilename, "r") == false) {
        return true;
    }

    char strbuffer[501];

    struct st_fileinfo stfileinfo;
    while (true) {
        memset(&stfileinfo, 0, sizeof(struct st_fileinfo));

        if (File.Fgets(strbuffer, 300, true) == false) break;

        GetXMLBuffer(strbuffer, "filename", stfileinfo.filename);
        GetXMLBuffer(strbuffer, "mtime", stfileinfo.mtime);

        vlistfile1.push_back(stfileinfo);
    }

    return true;
}

// 比较vlistfile2和vlistfile1，得到vlistfile3和vlistfile4
bool CompVecotr()
{
    vlistfile3.clear();
    vlistfile4.clear();

    int i, j;
    // 遍历vlistfile2
    for (i = 0; i < vlistfile2.size(); i++) {
        // 在vlistfile1中查找vlistfile2[i]的记录
        for (j = 0; j < vlistfile1.size(); j++) {
            // 如果找到了，把记录放入vlistfile3
            if ((strcmp(vlistfile2[i].filename, vlistfile1[j].filename) == 0)  &&
                (strcmp(vlistfile2[i].mtime, vlistfile1[j].mtime) == 0)) {
                vlistfile3.push_back(vlistfile2[i]);
                break;
            }
        }
        // 如果没有找到，把记录放入vlistfile4
        if (j == vlistfile1.size())  vlistfile4.push_back(vlistfile2[i]);
    }

    return true;
}

// 把容器vlistfile3中的内容写入okfilename文件，覆盖之前的旧okfilename文件
bool WriteToOKFile()
{
    CFile File;

    if (File.Open(starg.okfilename, "w") == false) {
        logfile.Write("File.Open(%s) failed.\n", starg.okfilename);
        return false;
    }

    for (int i = 0; i < vlistfile3.size(); i++) {
        File.Fprintf("<filename>%s</filename><mtime>%s</mtime>\n", vlistfile3[i].filename, vlistfile3[i].mtime);
    }

    return true;
}

// 如果ptype==1，把上传成功的文件记录追加到okfilename文件中
bool AppendToOKFile(struct st_fileinfo* stfileinfo)
{
    CFile File;

    if (File.Open(starg.okfilename, "a") == false) {
        logfile.Write("File.Open(%s) failed.\n", starg.okfilename);
        return false;
    }

    File.Fprintf("<filename>%s</filename><mtime>%s</mtime>\n", stfileinfo->filename, stfileinfo->mtime);
    return true;
}
```



### 3-10 本章总结



## 四、开发基于tcp协议的文件传输子系统

### 4.1 解决tcp粘包和分包的文件

```
在项目开发中，采用自定义的报文格式
```

+ 报文长度+报文内容



#### inet_函数

` int inet_aton（const char *strptr, struct in_addr *addrptr）;`

将一个字符串表示的点分十进制IP地址IP转换为32位网络字节序存储在addrptr中，并且返回该网络字节序表示的无符号整数。

将strptr所指C字符串转换成一个32位的网络字节序二进制值，并通过指针addrptr来存储。若成功则返回1，否则返回0。

`char *inet_ntoa(struct in_addr inaddr);`

将一个网络字节序的IP地址（也就是结构体in_addr类型变量）转化为点分十进制的IP地址（字符串）。该函数以一个结构而不是以指向该结构的一个指针作为其参数。返回：指向一个点分十进制数串的指针

```c++
int main(int argc, char* argv[])
{
    in_addr ia, ib;
    ia.s_addr = 16777343;

    std::cout << ntohs(36115) << std::endl;     // 5005
    std::cout << inet_ntoa(ia) << std::endl;    // "127.0.0.1"
    inet_aton("127.0.0.1", &ib);
    std::cout << ib.s_addr << std::endl;        // 167777343

    return 0;
}
```





### 4-7 TCP长连接心跳机制的实现

+ 在同一网段内部，网络设备不会断开空闲的连接
+ 在不同的网段之间，网络设备肯定会断开空闲的连接，超时时间1-5分钟
+ 网络服务程序心跳的超时时间一般设置在50-120秒之间





## 五、轻松搞定MySQL数据库的开发

### 5-1 开篇语

1. 基础知识的学习
   + 掌握MySQL数据库及客户端软件的安装、配置和使用
   + 掌握SQL语言（增删改查）和MySQL的常用函数
   + 理解MySQL高可用方案的原理

2. 封装MySQL数据库开发API
   + 学习connection和sqlstatement类的使用方法
   + 在C++程序中操作MySQL数据库
   + MySQL数据库开发注意事项
3. 学习数据库设计工具
   + 学习PowerDesigner软件的使用方法
   + 生成数据库设计文档和SQL语句

4. 把测试数据文件入库
   + 站点参数入库
   + 观测数据入库
   + 数据库版本是MySQL 5.7.34，字符集是utf8

```
mysql配置文件存放目录：/etc/my.cnf

通过如下命令可以在日志文件中找出密码：grep "password" /var/log/mysqld.log

修改mysql密码为：123456
ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';
```



### 5-2 创建信息表

```c++
#include "_mysql.h"

int main(int argc, char* argv[]) {
  connection conn;  // 数据库连接类

  // 登录数据库，返回值：0-成功，其他是失败，存放了MySQL的错误代码
  // 失败代码存放在conn.m_cda.rc中，失败描述在conn.m_cda.message中
  if (conn.connecttodb("127.0.0.1,root,99759346,mysql,3306", "utf8") != 0) {
    printf("connect database failed.\n%s\n", conn.m_cda.message);
    return -1;
  }

  // sqlstatement stmt;
  // stmt.connect(&conn);
  sqlstatement stmt(&conn);  // 操作SQL语句的对象

  // 准备创建表的SQL语句。
  // 超女表girls，超女编号id，超女姓名name，体重weight，报名时间btime，超女说明memo，超女图片pic。
  stmt.prepare(
      "create table girls(id     bigint(10),\
                   name     varchar(30),\
                   weight  decimal(8,2),\
                   btime   datetime,\
                   memo    longtext,\
                   pic     longblob,\
                   primary key (id))");
  /*
  1、int prepare(const char *fmt,...)，SQL语句可以多行书写。
  2、SQL语句最后的分号可有可无，建议不要写（兼容性考虑）。
  3、SQL语句中不能有说明文字。
  4、可以不用判断stmt.prepare()的返回值，stmt.execute()时再判断。
  */

  // 执行SQL语句，一定要判断返回值。0-成功，其他-失败
  // 失败代码存放在conn.m_cda.rc中，失败描述在conn.m_cda.message中
  if (stmt.execute() != 0) {
    printf("stmt.execute() failed.\n%s\n%d\n%s\n", stmt.m_sql, stmt.m_cda.rc,
           stmt.m_cda.message);
    return -1;
  }

  return 0;
}

/*
-- 超女基本信息表。
create table girls(id      bigint(10),    -- 超女编号。
                   name    varchar(30),   -- 超女姓名。
                   weight  decimal(8,2),  -- 超女体重。
                   btime   datetime,      -- 报名时间。
                   memo    longtext,      -- 备注。
                   pic     longblob,      -- 照片。
                   primary key (id));
*/

```



### 插入数据

```c++
#include "_mysql.h"

int main(int argc, char* argv[]) {
  connection conn;  // 数据库连接类

  // 登录数据库，返回值：0-成功，其他是失败，存放了MySQL的错误代码
  // 失败代码存放在conn.m_cda.rc中，失败描述在conn.m_cda.message中
  if (conn.connecttodb("127.0.0.1,root,99759346,mysql,3306", "utf8") != 0) {
    printf("connect database failed.\n%s\n", conn.m_cda.message);
    return -1;
  }

  // 定义用于超女信息的结构，与表中的字段对应
  struct st_girls {
    long id;         // 超女编号
    char name[31];   // 超女姓名
    double weight;   // 超女体重
    char btime[20];  // 报名时间
  } stgirls;

  // sqlstatement stmt;
  // stmt.connect(&conn);
  sqlstatement stmt(&conn);  // 操作SQL语句的对象

  // 准备插入表的SQL语句
  stmt.prepare(
      "insert into girls(id, name, weight, btime) values(:1+1, :2, :3+45.25, "
      "str_to_date(:4, '%%Y-%%m-%%d %%H:%%i:%%s'))");
  /*
    注意事项：
    1、参数的序号从1开始，连续、递增，参数也可以用问号表示，但是，问号的兼容性不好，不建议；
    2、SQL语句中的右值才能作为参数，表名、字段名、关键字、函数名等都不能作为参数；
    3、参数可以参与运算或用于函数的参数；
    4、如果SQL语句的主体没有改变，只需要prepare()一次就可以了；
    5、SQL语句中的每个参数，必须调用bindin()绑定变量的地址；
    6、如果SQL语句的主体已改变，prepare()后，需重新用bindin()绑定变量；
    7、prepare()方法有返回值，一般不检查，如果SQL语句有问题，调用execute()方法时能发现；
    8、bindin()方法的返回值固定为0，不用判断返回值；
    9、prepare()和bindin()之后，每调用一次execute()，就执行一次SQL语句，SQL语句的数据来自被绑定变量的值。
  */
  stmt.bindin(1, &stgirls.id);
  stmt.bindin(2, stgirls.name, 30);
  stmt.bindin(3, &stgirls.weight);
  stmt.bindin(4, stgirls.btime, 19);

  // 模式超女数据，向表中插入5条测试数据
  for (int i = 0; i < 5; i++) {
    memset(&stgirls, 0, sizeof(st_girls));

    // 为结构体变量的成员赋值
    stgirls.id = i;                                      // 超女编号
    sprintf(stgirls.name, "西施%05dgirl", i + 1);        // 超女姓名
    stgirls.weight = i;                                  // 超女体重
    sprintf(stgirls.btime, "2022-08-25 10:33:%02d", i);  // 报名时间

    if (stmt.execute() != 0) {
      printf("stmt.execute() failed.\n%s\n%s\n", stmt.m_sql,
             stmt.m_cda.message);
      return -1;
    }
    printf("成功插入了%d条记录。\n",
           stmt.m_cda.rpc);  // 本次执行SQL影响的记录数
  }

  printf("insert table girls ok.\n");

  conn.commit();  // 提交数据库事务

  return 0;
}

/*
-- 超女基本信息表。
create table girls(id      bigint(10),    -- 超女编号。
                   name    varchar(30),   -- 超女姓名。
                   weight  decimal(8,2),  -- 超女体重。
                   btime   datetime,      -- 报名时间。
                   memo    longtext,      -- 备注。
                   pic     longblob,      -- 照片。
                   primary key (id));
*/

```

