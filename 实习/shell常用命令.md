#shell辅助命令 wc、sort、uniq、cut、read、sleep
1、wc统计  常用选项-l
例：cat /etc/passwd | wc -l

2、sort排序 常用选项 -g  -r
 排序规则（默认是字典顺序）
例：sort -g 1.txt   # 按数值大小排序
      sort -r -g 1.txt   # 倒序排

3、uniq去重  常用选项 -c
     一般情况下先排序后去重
例：sort 2.txt | uniq -c    #分组去重，并统计重复的次数

4、cut文本截取 常用选项 -c  -d  -f
例：cut -c 2 2.txt  #按字符去截取  每行第2个字符
                  2-5  #每行第2到5个字符
                   -5   #每行前5个字符
                   5-  #每行第5个字符到结尾

-d  指定分割符，-f取分割后的第几段
cat 1.txt | cut -d ":" -f 1,7  # 按：分割，取第1和第7段
                                -f 1-3  #第1段到第3段

5、sleep 睡眠 可以用来控制命令执行时间
  例： sleep 5  #睡眠5秒


#文本处理工具，脚本三剑客grep、sed、awk
1、grep #文本过滤   常用选项  -i -E -o -v
 -i 忽略大小写  例：cat 1.txt | grep -i "root"
-E 启用扩展正则支持  ？ + |  （ ）{ }
-o  只输出匹配内容（默认输出满足条件的整行内容）
-v 取反
  例：处理配置文件
cat /usr/local/nginx/conf/nginx.conf | grep -vE "^\s*#|^$"
\s ：表示空格  
^$：表示空行

2、sed命令  编辑器
语法： sed [选项]... {脚本(如果没有其他脚本)} [输入文件]...
sed  选项  ‘地址定位+处理动作’ 文件
                    哪些行需要处理+怎么处理

#选项：
-r #启动扩展正则
-n  #抑制标准输出，一般和p动作一起使用
-i   #回源（修改源文件）

#地址定位
1）行号
例：sed -n '3p' 1.txt  #第3行
     sed -n '1,3p' 1.txt   #第1行到第3行
    sed -n '1~2p' 1.txt   #从1行开始，每2行输出一次

2）正则定位     /正则表达式/
例：sed -n -r '/^s/p' 1.txt  #输出以s开头的行
      sed -n -r '/^s/,/^m/p' 1.txt  #从s开头的行开始到m开头的行结束


#处理动作
p：输出
d：删除   cat -n 1.txt | sed '3d'     -i 才会删原文件
a：指定位置后面新增一行
i：指定位置前面新增一行
c：替换指定行
    例：cat 1.txt |sed '1a\hello'
          cat 1.txt |sed '1i\hello'
          cat 1.txt |sed '1c\hello'

s：局部替换
   '地址地位s/原内容/新内容/修饰符'
  sed '1s/bin/BIN/g' 1.txt  #第1行所有的bin全部替换
  sed '1s/bin/BIN/gi' 1.txt  #替换且不区分大小写
  sed '1s/bin/BIN/2' 1.txt   #替换第2个出现的关键字

练习：
head /etc/passwd > 1.txt
1、输出1.txt的第5行到结尾的内容
2、输出1.txt文件的偶数行内容
3、删除1.txt文件以nologin结尾的行
4、输出用户名只有3个字母的行
5、输出用户编号是2位数的行
6、这文件第一行插入内容“这是用户信息：”
7、将文件中的所有的root替换成大写ROOT
8、将文件中前5行，每行出现的第2个root换成大写
9、关闭selinux，修改配置文件/etc/selinux/config
10、修改时间同步服务配置文件/etc/chrony.conf，配置成服务端。 

3、awk
语法：awk  选项 ‘地址定位{动作}’ 文件名
选项：-F
动作：print
基本用法
 ip a | awk '/ens33$/{print $2}' | awk -F / '{print $1}'

地址定位：
1）行号（不直接支持，需要借助变量NR）
cat 1.txt | awk 'NR==3{print}'  #输出第3行


2)正则定位（类似sed）
cat 1.txt | awk '/^a/{print}'  #输出以a开头的行
cat 1.txt | awk -F : '/^a/{print $3}' 

3)关系表达式定位
例：cat 3.txt | awk '$2<60{print $1}'  #满足第2列小于60就输出第1列





[[shell语句]]






