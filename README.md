### 背景：

1. 最近手痒，整理了下IOS APP逆向工程相关资料，分享出来大家一起看看。
2. 逆向工程可分为四步：砸壳、dump、hook、重签。

### 砸壳：

- 概述：IOS的APP，若上传了App Store会被苹果进行一次加密，所以我们下载下来的安装包都是加密的，若要进行dump需要进行一次解密，即砸壳。我们以微信为例：首先我们需要一台已经越狱了的iPhone手机，然后进入Cydia安装需要的三款工具openSSH、Cycript、iFile。（调试程序时可以方便地查看日志文件）新版本iTunes已经将应用功能去掉了，所以大家只能用手机从App Store下载最新微信。砸壳第一步：获取微信的可执行文件的具***置和沙盒的具***置，我们先把iPhone上的所有程序都关掉，唯独留下微信。连接ssh，打开Mac的bash，用ssh连上iPhone(确保iPhone跟Mac在同一个网段)。openSSH的root密码默认为：alpine

![图一](https://github.com/PengJiang520/Crack/blob/main/图一.jpg?raw=true)

- 然后输入命令 `ps -e | grep WeChat`

![图二](https://github.com/PengJiang520/Crack/blob/main/图二.jpg?raw=true)

- 有时候我们需要一个APP的Bundle id了，这里笔者有一个小技巧，我们可以把iPhone上的所有App都关掉，唯独保留APP如微信，然后输入命令`ps -e`，便可输出当前运行的APP的Bundle id。

- 寻找沙盒Documents具体路径，我们需要用Cycript找出微信的Documents具体路径。输入命令`cycript -p WeChat`

- 寻找沙盒Documents具体路径，我们需要用Cycript找出微信的Documents具体路径。输入命令`cycript -p WeChat`打开微信，进入cy#模式输入`NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, ES)[0]`就能找出Documents的具体路径，或者使用`cy# directory = NSHomeDirectory()，也可以得到沙河位置（缺少/Documents）。`

  ![图三](https://github.com/PengJiang520/Crack/blob/main/图三.jpg?raw=true)

  control+D 可退出模式

### 编译dumpdecrypted

- 进入dumpdecrypted源码的目录，输入指`make`来编译dumpdecrypted。一般`make`后会在当前目录下生成一个dylib文件。

  

### 创建动态库文件

- 在确保Makefile中对动态库的设置和iOS真机环境一致后，在当前目录下输入：make。但是失败了，错误信息如下：

  ```python
  1.`xcrun --sdk iphoneos --find gcc` -Os  -Wimplicit -isysroot `xcrun --sdk iphoneos --show-sdk-path` -F`xcrun --sdk iphoneos --show-sdk-path`/System/Library/Frameworks -F`xcrun --sdk iphoneos --show-sdk-path`/System/Library/PrivateFrameworks -arch armv7 -arch armv7s -arch arm64 -c -o dumpdecrypted.o dumpdecrypted.c  
  2. /bin/sh: /Applications/Xcode: No such file or directory  
  3. make: *** [dumpdecrypted.o] Error 127  
  ```

- 原因是找不到/Applications/Xcode来执行其中的一些脚本。 好吧，我的Mac中有3个Xcode：/Applications/Xcode 5.0.2, /Applications/Xcode 5.1.1, /Applications/Xcode 6 Beta4，就是没有/Applications/Xcode。 没事，将Xcode 5.1.1重命名为Xcode就行了：

  > $ sudo mv Xcode\ 5.1.1.app/ Xcode.app/ 

- 第二次错误

  再make，还是报错，错误信息和上面一样。

  不怕，我们还有xcode-select这个小伙伴，通常Xcode找不到之类的错误都应该找它帮忙：

  > 1. $ xcode-select -p 
  > 2. /Applications/Xcode 5.1.1.app/Contents/Developer 

  原来xcrun查找cmd tool时的路径还是Xcode 5.1.1/，当然什么都找不到了。这时候将它重置就行了（默认是/Applications/Xcode.app/）：

  > 1. $ sudo xcode-select -r 
  > 2. $ xcode-select -p  
  > 3. /Applications/Xcode.app/Contents/Developer 

   

- 成功

  再make，成功，输出如下

  > 1. $ make 
  > 2. `xcrun --sdk iphoneos --find gcc` -Os -Wimplicit -isysroot `xcrun --sdk iphoneos --show-sdk-path` -F`xcrun --sdk iphoneos --show-sdk-path`/System/Library/Frameworks -F`xcrun --sdk iphoneos --show-sdk-path`/System/Library/PrivateFrameworks -arch armv7 -arch armv7s -arch arm64 -c -o dumpdecrypted.o dumpdecrypted.c 
  > 3. `xcrun --sdk iphoneos --find gcc` -Os -Wimplicit -isysroot `xcrun --sdk iphoneos --show-sdk-path` -F`xcrun --sdk iphoneos --show-sdk-path`/System/Library/Frameworks -F`xcrun --sdk iphoneos --show-sdk-path`/System/Library/PrivateFrameworks -arch armv7 -arch armv7s -arch arm64 -dynamiclib -o dumpdecrypted.dylib dumpdecrypted.o 
  > 4.  
  > 5. $ ls 
  > 6. Makefile     dumpdecrypted.c     dumpdecrypted.o 
  > 7. README        dumpdecrypted.dylib 

  可以看到目录中多了两个文件，其中dylib后缀的就是我们要创建的动态库文件，也就是用来砸壳的锤子。

- scp拷贝指令

  使用scp指令把dumpdecrypted.dylib拷贝到iPhone的Documents路径目录下输入指令：`scp 源文件路径:目标文件路径`

  ![图四](https://github.com/PengJiang520/Crack/blob/main/图四.jpg?raw=true)

- 开始砸壳！！！

  回到手机的ssh上，输入dumpdecrypted的指令。
  dumpdecrypted的具体用法：`DYLD_INSERT_LIBRARIES=/PathFrom/dumpdecrypted.dylib /PathTo`

  ![图5](https://github.com/PengJiang520/Crack/blob/main/图5.jpg?raw=true)

  ```python
  *mach-o decryption dumper*
  *DISCLAIMER: This tool is only meant for security research purposes, not for application crackers.*
  *[+] detected 32bit ARM binary in memory.*
  *[+] offset to cryptid found: @0x56a4c(from 0x56000) = a4c*
  *[+] Found encrypted data at address 00004000 of length 38748160 bytes - type 1.*
  *[+] Opening /private/var/mobile/Containers/Bundle/Application/2C920956-E3D6-4313-BD88-66BD24CEBE9B/WeChat.app/WeChat for reading.*
  *[+] Reading header*
  *[+] Detecting header type*
  *[+] Executable is a FAT image - searching for right architecture*
  *[+] Correct arch is at offset 16384 in the file*
  *[+] Opening WeChat.decrypted for writing.*
  *[+] Copying the not encrypted start of the file*
  *[+] Dumping the decrypted data into the file*
  *[+] Copying the not encrypted remainder of the file*
  *[+] Setting the LC_ENCRYPTION_INFO->cryptid to 0 at offset 4a4c*
  *[+] Closing original file*
  *[+] Closing dump file*
  ```

  这样就代表砸壳成功了，当前目录下会生成砸壳后的文件，即WeChat.decrypted。同样用scp命令把WeChat.decrypted文件拷贝到电脑上,接下来我们要正式的dump微信的可执行文件了。

  - scp远程下载到本地

    输入指令：`scp -r root@ip:文件目录/文件名 /目的地`

    ![图六](https://github.com/PengJiang520/Crack/blob/main/图六.jpg?raw=true)

  - dump

    ```python
    针对于debug或者release的包，无需砸壳，可以直接dump。只有从App Store下载下来的包，需要砸壳。
    
    　　dump方法为：安装后使用的命令为：class-dump -H 需要导出的框架路径 -o 导出的头文件存放路径
    　　如：cd到该文件下，用class-dump -H WeChat
    　　便可以得到微信代码的所有方法的声明.h文件。
    ```

  - HOOK

    找到**CMessageMgr.h**和**WCRedEnvelopesLogicMgr.h**这两文件，其中我们注意到有这两个方法：`- (void)AsyncOnAddMsg:(id)arg1 MsgWrap:(id)arg2;` ，`- (void)OpenRedEnvelopesRequest:(id)arg1;`。没错，接下来我们就是要利用这两个方法来实现微信自动抢红包功能。其实现原理是，通过hook微信的新消息函数，我们判断是否为红包消息，如果是，我们就调用微信的打开红包方法。这样就能达到自动抢红包的目的了。哈哈，是不是很简单，我们一起来看看具体是怎么实现的吧。

    - 新建一个dylib工程，因为Xcode默认不支持生成dylib，所以我们需要下载iOSOpenDev，安装完成后(Xcode7环境会提示安装iOSOpenDev失败，请参考[iOSOpenDev安装问题](https://link.jianshu.com/?t=http://www.tqcto.com/article/software/14553.html))，重新打开Xcode，在新建项目的选项中即可看到iOSOpenDev选项了。

      ![图七](https://github.com/PengJiang520/Crack/blob/main/图七.jpg?raw=true)

    - iOSOpenDev

      - dylib代码

        选择Cocoa Touch Library，这样我们就新建了一个dylib工程了，我们命名为autoGetRedEnv。

        删除autoGetRedEnv.h文件，修改autoGetRedEnv.m为autoGetRedEnv.mm，然后在项目中加入[CaptainHook.h](https://link.jianshu.com/?t=https://github.com/rpetrich/CaptainHook)

        因为微信不会主动来加载我们的hook代码，所以我们需要把hook逻辑写到构造函数中。

        ```c
        __attribute__((constructor)) static void entry()
        {
            //具体hook方法
        }
        ```

​                           hook微信的AsyncOnAddMsg: MsgWrap:方法，实现方法如下：

```c++
//声明CMessageMgr类
CHDeclareClass(CMessageMgr);
CHMethod(2, void, CMessageMgr, AsyncOnAddMsg, id, arg1, MsgWrap, id, arg2)
{
    //调用原来的AsyncOnAddMsg:MsgWrap:方法
    CHSuper(2, CMessageMgr, AsyncOnAddMsg, arg1, MsgWrap, arg2);
    //具体抢红包逻辑
    //...
    //调用原生的打开红包的方法
    //注意这里必须为给objc_msgSend的第三个参数声明为NSMutableDictionary,不然调用objc_msgSend时，不会触发打开红包的方法
    ((void (*)(id, SEL, NSMutableDictionary*))objc_msgSend)(logicMgr, @selector(OpenRedEnvelopesRequest:), params);
}
__attribute__((constructor)) static void entry()
{
    //加载CMessageMgr类
    CHLoadLateClass(CMessageMgr);
    //hook AsyncOnAddMsg:MsgWrap:方法
    CHClassHook(2, CMessageMgr, AsyncOnAddMsg, MsgWrap);
}
```

*项目的全部代码，笔者已放入*[Github](https://link.jianshu.com/?t=https://github.com/east520/AutoGetRedEnv)*中。

完成好具体实现逻辑后，就可以顺利生成dylib了。

### 重新打包微信App

- 为微信可执行文件注入dylib

  要想微信应用运行后，能执行我们的代码，首先需要微信加入我们的dylib，这里我们用到一个dylib注入神器:[yololib](https://link.jianshu.com/?t=https://github.com/KJCracks/yololib)，从网上下载源代码，编译后得到yololib。

  使用yololib简单的执行下面一句就可以成功完成注入。注入之前我们先把之前保存的WeChat.decrypted重命名为WeChat，即已砸完壳的可执行文件。
  `./yololib 目标可执行文件 需注入的dylib`
  注入成功后即可见到如下信息：

  ![图八](https://github.com/PengJiang520/Crack/blob/main/图八.jpg?raw=true)

  dylib注入

- 新建Entitlements.plist

  ```
  <?xml version="1.0" encoding="UTF-8"?>
  <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
  <plist version="1.0">
  <dict>
      <key>application-identifier</key>
      <string>123456.com.autogetredenv.demo</string>
      <key>com.apple.developer.team-identifier</key>
      <string>123456</string>
      <key>get-task-allow</key>
      <true/>
      <key>keychain-access-groups</key>
      <array>
          <string>123456.com.autogetredenv.demo</string>
      </array>
  </dict>
  </plist>
  ```

  这里大家也许不清楚自己的证书Teamid及其他信息，没关系，笔者这里有一个小窍门，大家可以找到之前用开发者证书或企业证书打包过的App(例如叫Demo)，然后在终端中输入以下命令即可找到相关信息，命令如下：
  `./ldid -e ./Demo.app/demo`

  接下来把我们生成的**dylib(libautoGetRedEnv.dylib)**、刚刚注入dylib的**WeChat**、以及**embedded.mobileprovision**文件(可以在之前打包过的App中找到)拷贝到**WeChat.app**中。　　

### 重签

- 给微信重新签名

- 命令格式：`codesign -f -s 证书名字 目标文件`

  > PS:证书名字可以在钥匙串中找到

- 分别用codesign命令来为微信中的相关文件签名,具体实现如下：

  ![图九](https://github.com/PengJiang520/Crack/blob/main/图九.jpg?raw=true)

- 打包成ipa

  给微信重新签名后，我们就可以用xcrun来生成ipa了，具体实现如下：
  `xcrun -sdk iphoneos PackageApplication -v WeChat.app -o ~/WeChat.ipa`

### 安装拥有抢红包功能的微信

以上步骤如果都成功实现的话，那么真的就是万事俱备，只欠东风了~~~

我们可以使用iTools工具，来为iPhone(此iPhone Device id需加入证书中)安装改良过的微信了。

![图十](https://github.com/PengJiang520/Crack/blob/main/图十.jpg?raw=true)

### 参考

- https://www.likecs.com/show-253024.html
