---
layout: post
title: "Unix like系统中，查找某个目录下包含特定关键字的文件"
date: 2013-09-26 22:39
comments: true
categories: 
- linux
- 技术
--- 
{% img center /images/2013/09/27/2013-09-27-02.jpg %}  

作为一枚程序猿，咋们经常在工作当中会遇到一种场景：		
**查找某个目录中包含某个关键字的文件**，那么我们如何来实现这个需求呢？我个人的习惯是如果某个任务经常要执行，那么我会习惯性的写一个脚本，建立一个alias，然后每次需要的时候，直接调用脚本来完成任务，这才是咋们程序猿应该有的工作态度嘛。  

好了，废话不多了，咋们直接show code啦。

<!-- more -->

```bash find files in a specific dir with a keyword  
 #!/bin/bash 
 #find files in a specific dir with a keyword  
 #write by tiger 
 #2013.09.26
 
echo -e "\nThis script finds files in a specific dir with a keyword.\nOK,Please input a keyword:" 
 
read keyword 
if [ "$keyword" == "" ]; then 
    echo  "keyword can not be null!\n" 
    exit 0 
fi 
      
echo "\nPlease input the dir path:" 
read dirPath 
while [ "$dirPath" == "" ]
do
	echo  "The dir can't be null,pls input it again"
	read  dirPath
done

if [ ! -d "$dirPath" ]; then
  echo "The $dirPath is not exist!\n\n" 
  exit 0 
fi
      
echo  "\n--------------- Find these files ---------------\n" 
  
fileCount=0 
files=`ls -R $dirPath 2> /dev/null | grep -v '^$'` 
for fileName in $files 
do  
    temp=`echo $fileName | sed 's/:.*$//g'` 
    if [ "$fileName" != "$temp" ]; then 
        currentDir=$temp 
    else 
        fileType=`file $currentDir/$fileName | grep "text"` 
        if [ "$fileType" != "" ]; then 
            temp=`grep $keyword $currentDir/$fileName 2> /dev/null` 
            if [ "$temp" != "" ]; then 
                echo $currentDir/$fileName  
                let fileCount++ 
            fi 
        fi 
    fi 
done 
if [ $fileCount -gt 0 ];then
  echo "\n\nFiles Total: $fileCount" 
  echo "\nFind Finished!\n" 
else
	echo "No files found!"      
fi

```

