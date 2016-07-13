#Ant构建Android工程学习笔记

>ant 是一个将软件编译、测试、部署等步骤联系在一起加以自动化的一个工具，大多用于Java环境中的软件开发。

##ant脚本的几个基本节点元素

###project

project 元素是 Ant 构件文件的根元素， Ant 构件文件至少应该包含一个 project 元素，否则会发生错误。在每个 project 元素下，
可包含多个 target 元素。接下来向读者展示一下 project 元素的各属性。

- name 属性：用于指定 project 元素的名称。 
- default 属性：用于指定 project 默认执行时所执行的 target 的名称。 
- basedir 属性：用于指定基路径的位置。该属性没有指定时，使用 Ant 的构件文件的附目录作为基准目录。

###target

target为ant的基本执行单元或是任务，它可以包含一个或多个具体的单元/任务。多个target 可以存在相互依赖关系。它有如下属性： 
- name 属性：指定 target 元素的名称，这个属性在一个 project 元素中是唯一的。我们可以通过指定 target 元素的名称来指定某个 target 。 
- depends 属性：用于描述 target 之间的依赖关系，若与多个 target 存在依赖关系时，需要以“,”间隔。 Ant 会依照 depends 属性中 target 出现的顺序依次执行每个 target ，被依赖的target 会先执行。 
- if 属性：用于验证指定的属性是存在，若不存在，所在 target 将不会被执行。 
- unless 属性：该属性的功能与 if 属性的功能正好相反，它也用于验证指定的属性是否存在，若不存在，所在 target 将会被执行。 
- description 属性：该属性是关于 target 功能的简短描述和说明。 

###property

property元素可看作参量或者参数的定义，project 的属性可以通过 property 元素来设定，也可在 Ant 之外设定。若要在外部引入某文件，
例如 build.properties 文件，可以通过如下内容将其引：
    <property file="build.properties"/> 
property 元素可用作 task 的属性值。在 task 中是通过将属性名放在```${属性名}```之间，并放在 task 属性值的位置来实现的。 
Ant 提供了一些内置的属性，它能得到的系统属性的列表与 Java 文档中``` System.getProperties()``` 方法得到的属性一致，这些系统属性可参考 sun 网站的说明。
同时， Ant 还提供了一些它自己的内置属性，如下： 

- basedir： project 基目录的绝对路径；   
- ant.file： buildfile的绝对路径，上例中ant.file值为D:\Workspace\AntExample\build； 
- ant.version： Ant 的版本信息，本文为1.8.1 ； 
- ant.project.name： 当前指定的project的名字，即前文说到的project的name属性值； 
- ant.java.version： Ant 检测到的JDK版本，本文为 1.6 。

还有其他节点和属性请看[这里](https://ant.apache.org/manual/)
下面结合项目来具体分析一下构建过程。

##具体分析

###项目介绍

该项目是一个app工场，要求一套代码生成多个不同的app。这个不同是指包名，app名称，appicon，一些资源图片，一些第三方sdk的license等等。

先看project节点
```<project name="John" basedir=".">```
并未指定default属性，所以ant命令中要指定

>ant 
在当前目录下的build.xml运行Ant，执行缺省的target。
ant -buildfile build-test.xml 
在当前目录下的build-test.xml运行Ant，执行缺省的target。
ant -buildfile build-test.xml clean 
在当前目录下的build-test.xml运行Ant，执行一个叫做clean的target。
ant -buildfile build-test.xml -Dbuild=build/classes clean 
在当前目录下的build-test.xml运行Ant，执行一个叫做clean的target，并设定build属性的值为build/classes。

已知指定的target如下：
```
    <target name="nofify_success" depends="deploy">
        <echo>curl ${project.success}</echo>
        <exec executable="/usr/bin/curl" failonerror="true">
            <arg value="${project.success}"/>
        </exec>
    </target>
```

cURL是一个利用URL语法在命令行下工作的文件传输工具，这个target实际上是将编好的包上传到```${project.success}```指定的服务器上去。
这不是我们关注的重点。我们看```depends="deploy"```这是这个target之前要做的事。

```
    <target name="deploy" depends="rebuild, create_remote_dir">
        <echo>Deploy to ${deploy.file}</echo>
        <scp file="${last-package}"
             todir="${deploy.file}" trust="true"/>
    </target>
```

>scp节点
Copies a file or FileSet to or from a (remote) machine running an SSH daemon. FileSet only works for copying files from the local machine to a remote machine.
复制一个文件或文件集，或从运行SSH守护进程（远程）机器。文件集仅适用于复制从本地机文件到远程计算机。

这个target貌似是把生成的apk复制到了服务器上。再追溯它之前的target```rebuild```和```create_remote_dir```


```
    <target name="create_remote_dir">
        <sshexec host= "${deploy.server}" username="${deploy.user}" password="${deploy.password}"
                 command="rm -rf ${deploy.todir}; mkdir -p ${deploy.todir}" trust="true"/>
    </target>
```

>sshexec
Runs a command on a remote machine running SSH daemon.
运行在远程机器上运行SSH守护进程的命令。
host
The hostname or IP address of the remote host to which you wish to connect.
command
The command to run on the remote host.


```
    <target name="rebuild" depends="zipalign">
        <echo>del unneed apk...</echo>
        <delete file="${outdir}/${project.name}-unsigned.apk"/>
        <delete file="${outdir}/${project.name}_${DSTAMP}_build_signed.apk"/>
        <delete file="${resources-package}"/>
        <rename src="${outdir}/${project.name}_${project.version}_${DSTAMP}_build_signed_qa.apk"
                dest="${last-package}"/>
    </target>
```
这个target只是将apk重命名了。再看看```depends="zipalign"```

```
    <target name="zipalign" depends="jarsigner">
        <exec executable="${android-zipalign}" failonerror="true">
            <arg value="-v"/>
            <arg value="-f"/>
            <arg value="4"/>
            <arg value="${outdir}/${project.name}_${DSTAMP}_build_signed.apk"/>
            <arg value="${outdir}/${project.name}_${project.version}_${DSTAMP}_build_signed_qa.apk"/>
        </exec>
    </target>
```

貌似有点熟悉了，这不是Android打包的优化过程吗！再看```jarsigner```是签名过程,再往上呢？

![pic1](C:\Users\sks\Desktop\图片1.png)

我们通过这张图来回顾一下Android的打包过程吧。

target"zipalign"实际所做的工作就是
```
zipalign -v -f 4 ${outdir}/${project.name}_${DSTAMP}_build_signed.apk ${outdir}/${project.name}_${project.version}_${DSTAMP}_build_signed_qa.apk
```

>Usage: zipalign [-f] [-v] [-z] <align> infile.zip outfile.zip
 zipalign -c [-v] <align> infile.zip
  <align>: alignment in bytes, e.g. '4' provides 32-bit alignment
  -c: check alignment only (does not modify file)
  -f: overwrite existing outfile.zip
  -v: verbose output
  -z: recompress using Zopfli
这是zipalign用法，参数可一一对照

target"jarsigner"
```
    <target name="jarsigner" depends="add-seconde-dex">
        <exec executable="${jarsigner}" failonerror="true">
```
这是签名，参数就不贴了。

```
    <target name="add-seconde-dex" depends="release">
        <echo>Adding seconde dex ...</echo>
        <exec executable="${android-aapt}" dir="${outdir}" failonerror="true">
            <arg line='add -v "${outdir}/${project.name}-unsigned.apk" classes2.dex'/>
        </exec> 
    </target>
```
还原成命令行是
```
aapt add -v "${outdir}/${project.name}-unsigned.apk" classes2.dex
```

> aapt a[dd] [-v] file.{zip,jar,apk} file1 [file2 ...]
   Add specified files to Zip-compatible archive.
   
对照用法，可解读成将未签名的apk添加到了class2.dex里。
那这个class2.dex是什么，又是怎么来的？
我们再往上看
```
<target name="release" depends="package-res-and-assets">
        <echo>Packaging for release...</echo>
        <exec executable="${android-builder}" failonerror="true">
            <arg value="${outdir}/${project.name}-unsigned.apk"/>
            <arg value="-u"/>
            <arg value="-z"/>
            <arg value="${resources-package}"/>
            <arg value="-f"/>
            <arg value="${intermediate-dex-location}"/>
            <arg value="-rf"/>
            <arg value="${project.home}/src"/>
            <arg value="-rj"/>
            <arg value="${external-libs}"/>
            <arg value="-nf"/>
            <arg value="${external-libs}"/>
            <arg value="-nf"/>
            <arg value="${external-im-libs}"/>
        </exec>
        <echo>It will need to be signed with jarsigner before being published.</echo>
    </target>
```
翻译成命令行如下：
```
apkbuilder ${outdir}/${project.name}-unsigned.apk -u -z ${resources-package} -f ${intermediate-dex-location} -rf ${project.home}/src 
-rj ${external-libs} -nf ${external-libs} -nf ${external-im-libs}
```

>参数含义看[这里](http://blog.csdn.net/electricity/article/details/6543299)

 这里会遇到一个坑，apkbuilder按道理来说应该在${ANDROID_SDK_HOME}/tools这个目录下，但如果你的sdk升级到22以上后，你会发现，这个批处理文件不见了。[这里](http://www.devdiv.com/forum.php?mod=viewthread&tid=204076)有解决办法。下面是
 ```
 set jarfile=sdklib.jar
set frameworkdir=

if exist %frameworkdir%%jarfile% goto JarFileOk
    set frameworkdir=lib\

if exist %frameworkdir%%jarfile% goto JarFileOk
    set frameworkdir=..\framework\

:JarFileOk

set jarpath=%frameworkdir%%jarfile%

call %java_exe% -classpath %jarpath% com.android.sdklib.build.ApkBuilderMain %*
```
 apkbuilder是一个脚本工具，实际调用的是android-sdk\tools\lib\sdklib.jar文件中的com.android.sdklib.build.ApkBuilderMain类。它的代码实现位于android系统源码的sdk\sdkmanager\libs\sdklib\src\com\android\sdklib\build\ApkBuilderMain.java文件，代码构建了一个ApkBuilder类，然后 以包含resources.arsc的文件为基础生成apk文件，这个文件一般为ap_结尾，接着调用addSourceFolder()函数添加工程资源，addSourceFolder()会调用processFileForResource（）函数往apk文件中添加资源，处理的内容包括res目录与asserts目录中的文件，添加完资源后调用addResourceFromJar（）函数往apk文件中写入依赖库，接着调用addNativeLibraries()函数添加工程libs目录下的Native库（通过android NDK编译生成的so或bin文件），最后调用sealApk（）关闭apk文件。









