---
title: "bash cheatsheet"
date: 2023-02-21T13:05:39+08:00
categories: [bash]
---
# 管道
``` bash
time ! false && echo "output to stderr" >&2 && echo "output to stdin"  |& cat 
#real	0m0.000s
#user	0m0.000s
#sys	0m0.000s
#output to stderr
#output to stdin
```
# 列表
``` bash
{ sleep 1;echo -en "first one "; } & echo -en "second one " #second one first one
{ sleep 1;echo -en "first one "; } ; echo -en "second one " # first one second one
false && echo 1   #""
! false && echo 1 #1
false || echo 1   #1
! false || echo 1 #"" 
```
# 复杂命令
## (list)
```bash
(read <<<1;echo $REPLY);echo $REPLY 
#1
#
{ read <<<1;echo $REPLY; };echo $REPLY
#1
#1
```
## ((list))
```bash
a=0;(( a++ ));echo $ #1
a=1;(( a>0 ));echo $? #0
a=0;echo $(( a+1 )) #1
```
## [[ list ]]
```bash
[[ "a">"b" ]];echo $? #0
[[ "a">"ab"]];echo $? #1
[[ 'abc' == a* ]];echo $? #0
[[ 'abcd' =~ .* ]]; echo $? #0
[[ ! "a" == "b" ]]; echo $? #0
[[ "a" == "a" && "b" == "c" ]];echo $? #1
[[ "a" == "a" || "b" == "c" ]];echo $? #0
```
##  for _name_ [ [ in [ _word ..._ ] ] ; ] do _list_ ; done
``` bash
for num in {1..3};do
	echo num
done
#1
#2
#3
```
## for _name_;do _list_ ; done
```bash
test(){
	for num;do
		echo $num
	done
}
test 1 2 3
#1
#2
#3
```
## for (( expr1 ; expr2 ; expr3 )) ; do list ; done
``` bash
for ((i=0;i<3;i++));do
	echo $i
done
#0
#1
#2
```
## select name [ in word ] ; do list ; done
``` bash
PS3="choose your favorite distro:"
select distro in archlinux debian;do
	echo "You favorite distro is ${distro}"
	echo "\$REPLY:$REPLY"
	break
done
#1) archlinux
#2) debian
#choose your favorite distro:1
#You favorite distro is archlinux
#$REPLY:1
```
## case word in [ [(] pattern [ | pattern ] ... ) list ;; ] ... esac
``` bash
word="hello"
case ${word/h/H} in
    Hello) echo -n "Hello";;&
    Hello) echo -n "HHH" ;&
    hello) echo -n "hello" ;;
    *) echo -n "world";;
esac
#HelloHHHhello
```
## if list; then list; [ elif list; then list; ] ... [ else list; ] fi
```bash
if { echo "if list"; ((1==0)); };then
echo "then list"
elif { echo "elif list"; ((1==0)); };then
echo "corresponding then list"
else
echo "else then list"
fi
#if list
#elif list
#else then list
```
## while list-1; do list-2; done
```bash
typeset -i a=0
while { echo "list-1";((a<1)); };do
echo "list-2"
((a++))
done
#list-1
#list-2
#list-1
```
## until list-1; do list-2; done
```bash
typeset -i a=0
until { echo "list-1"; ((a==1)); };do
echo "list-2"
((a++))
done
#list-1
#list-2
#list-1
```

