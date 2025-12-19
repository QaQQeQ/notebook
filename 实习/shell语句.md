#条件控制语句 if   case
1） if 语法：
if COMMANDS;   #根据命令的返回值判断命令是否成立
then
   COMMANDS    #返回值为0才执行
[elif COMMANDS; 
then
   COMMANDS]
... ...
[else
   COMMANDS]   #前面条件都不满足才执行
fi

整数计算方式：
expr 1 + 2
echo $((1+2))
a=$((1+2));echo $a

练习：写个支持+-*/整数的计算器
#!/bin/bash
read -e -p "第一个数：" a
read -e -p "运算符：" p
read -e -p "第二个数：" b
if [ "$p" = "+" ]
then
	echo $(($a+$b))
elif [ "$p" = "-" ]
then
	echo $(($a-$b))
elif [ "$p" = "*" ]
then
	echo $(($a*$b))
elif [ "$p" = "/" ]
then
	echo $(($a/$b))
else
   echo "符号错误，只支持+-*/"
fi

单条件判断 && ||
命令1  && 命令2 || 命令3 
解释：命令1的返回值是0就是执行命令2，非0则执行命令3

case分支（单条件），适合写菜单类的脚本
#!/bin/bash
echo "1.查看本机ip"
echo "2.创建用户"
echo "3.关闭防火墙"
echo "4.安装软件"
echo "------------------------"
read -e -p "请选择：" i
case $i in
1)
ip a | grep ens33$
;;

2)
read -e -p "输入创建的用户名：" name
useradd $name
echo "$name 创建成功"
;;

3)
.........
;;

4)
.....
;;

*)
...............   #前面都不匹配就会执行
;;
esac





#循环语句  for   while until
for循环的两种语法风格
1）c风格 （控制循环此次）
for  ((变量初始值;循环条件;变量值变化))
do
     循环体（命令块，每次循环执行）
done
例：
#!bin/bash
for ((i=1;i<11;i++))
do
	echo "第$i次你好"
done


2）遍历单词序列
for 变量 in 单词序列 ; 
do 
     循环体（命令块，每次循环执行） 
done

变形：
a=$(命令)  #单词序列可以来自命令的执行结果
for 变量 in $a ; 
do 
     循环体（命令块，每次循环执行） 
done
例：
```
#!/bin/bash
a=$(ls /tmp/*txt)
for i in $a
do
	mv $i $i.bak
done
```


#随机数变量 $RANDOM
一般用于指定范围随机取值  （使用取余%）
j=$(($RANDOM%100+1))  #随机取值1-100



练习：基于1.log
将登录失败次数超过3次的ip地址过滤出来，放到一个文件中





[[基础命令]]