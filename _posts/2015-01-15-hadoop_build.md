---
title: Yet Another Hadoop Build Tutorial for Windows
layout: post
---

Building open source software which are targeting *nix system on Windows is painful. Today I spent several hours and a lot efforts on building hadoop-2.6.0 on my windows 8 X64 laptop. To avoid experience the pain again, it's necessary to record the traps and how to workaround.

Dependencies
============

1. [hadoop 2.6.0 Source](http://hadoop.apache.org/releases.html#Download)
2. Windows 8 x64
3. [Visual Studio 2013](http://www.visualstudio.com/news/vs2013-community-vs)
4. [JDK 7](http://www.oracle.com/technetwork/java/javase/downloads/jdk7-downloads-1880260.html)
5. [Cygwin](https://cygwin.com/install.html)
6. [CMake 3.1.0](http://www.cmake.org/)
7. [Maven 3.2.5](http://maven.apache.org/download.cgi)
8. [protobuf 2.5.0](https://code.google.com/p/protobuf/downloads/list)
9. [Zlib](http://www.zlib.net/). I didn't use Zlib to avoid the dll hell of VC's runtime libraries among different versions and different thread/static/dynamic settings.



Steps
=====
<ol>
<li>
Open hadoop-2.6.0-src\hadoop-common-project\hadoop-common\src\main\native\native.sln and hadoop-2.6.0-src\hadoop-common-project\hadoop-common\src\main\winutils\winutils.sln in VS 2013 and upgrade the project files to VS 2013.
</li>
<li>
 Open hadoop-2.6.0-src\hadoop-hdfs-project\hadoop-hdfs\pom.xml and make the following changes
{% highlight xml linenos tabsize=4 %}
<!-- change from VS 2010 to VS 2013, see 'cmake -h'
<arg line="${basedir}/src/ -DGENERATED_JAVAH=${project.build.directory}/native/javah -DJVM_ARCH_DATA_MODEL=${sun.arch.data.model} -DREQUIRE_LIBWEBHDFS=${require.libwebhdfs} -DREQUIRE_FUSE=${require.fuse} -G 'Visual Studio 10 Win64'"/>
-->
<arg line="${basedir}/src/ -DGENERATED_JAVAH=${project.build.directory}/native/javah -DJVM_ARCH_DATA_MODEL=${sun.arch.data.model} -DREQUIRE_LIBWEBHDFS=${require.libwebhdfs} -DREQUIRE_FUSE=${require.fuse} -G 'Visual Studio 12 Win64'"/>
{% endhighlight %}
</li>
<li>
Remove readonly and add full control on hadoop source fodler for all users
</li>
<li>
Add Visual Studio, JDK, Cygwin, CMake, Maven, protobuf's executables' path to PATH env.
</li>
<li>
Launch 'cmd', enter the folder of hadoop source and type
{% highlight bat linenos tabsize=4 %}
    mvn package  -Pdist,native-win -DskipTests -Dtar  -e -X 
{% endhighlight %}
</li>
</ol>

All in one script
================
To avoid the manually editing job in previous section, I created a script to automatically setting the PATH env, edit .pom file and update VS solutions. Feel free to use it and don't forget to change the folder to point to your owns.
{% highlight bat linenos tabsize=4 %}
setlocal
call "C:\Program Files (x86)\Microsoft Visual Studio 12.0\Common7\Tools\VsDevCmd.bat"

set CYGWIN_ROOT=C:\cygwin64
set JAVA_HOME=C:\PROGRA~1\Java\JDK17~1.0_7
set M2_HOME=C:\tools\apache-maven-3.2.5-bin
set MS_BUILD_PATH=C:\Windows\Microsoft.NET\Framework\v4.0.30319
set CMAKE_PATH=C:\Program Files (x86)\CMake
set PROTOBUF_PATH=C:\tools\protoc-2.5.0-win32
set PATH=%PROTOBUF_PATH%;%CMAKE_PATH%\bin;%MS_BUILD_PATH%;%M2_HOME%\bin;%CYGWIN_ROOT%\bin;%PATH%
set HADOOP_SRC=hadoop-2.6.0-src
sed -i -e "s/Visual Studio 10 Win64/Visual Studio 12 Win64/" %HADOOP_SRC%\hadoop-hdfs-project\hadoop-hdfs\pom.xml
for %%i in (%HADOOP_SRC%\hadoop-common-project\hadoop-common\src\main\native\native.sln %HADOOP_SRC%\hadoop-common-project\hadoop-common\src\main\winutils\winutils.sln) do (
    devenv /Upgrade %%i
)
pushd hadoop-2.6.0-src
attrib * -R /S
icacls * /grant Everyone:(OI)(CI)F
set log=..\mvn.log
mvn package  -Pdist,native-win -DskipTests -Dtar  -e -X >> %log% 2>&1
popd

{% endhighlight %}

References
==========
[Building Hadoop on Windows by Dan Sandler](http://www.datadansandler.com/2013/09/building-hadoop-on-windows.html)
