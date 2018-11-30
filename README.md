#Learn golang to generate c dynamic library to let java call

reference https://studygolang.com/topics/6025#commentForm

##Environmental preparation
##环境准备


`GCC Golang JAVA`

Enter in the console
在控制台中输入

     GCC -v
     go version
     java -version
 
##Write the go program
##编写go程序
#####We are just writing a simple calculation decryption program that accepts two strings, then computes the decryption and returns. Here we name the file libaes128.go
#####我们这里只是编写一个简单的计算解密的程序，接受两个字符串，然后计算解密，并返回。 在这里，我们将文件命名为libaes128.go
```
package main

import (
	"C"
	"encoding/hex"
	"crypto/aes"
	"crypto/cipher"
	"fmt"
	"log"
)

//export Aes128Des
func Aes128Des(k string, data string) *C.char {
	key := []byte(k)
	ciphertext, _ := hex.DecodeString(data)

	block, err := aes.NewCipher(key)
	if err != nil {
		log.Println(err)
		return C.CString(err.Error())
	}

	// The IV needs to be unique, but not secure. Therefore it's common to
	// include it at the beginning of the ciphertext.
	if len(ciphertext) < aes.BlockSize {
		log.Println("ciphertext too short")
		return C.CString("ciphertext too short")
	}
	ciphertext = []byte("iu3riubbku0dsia0"+string(ciphertext))
	iv := ciphertext[:aes.BlockSize]
	ciphertext = ciphertext[aes.BlockSize:]

	// CBC mode always works in whole blocks.
	if len(ciphertext)%aes.BlockSize != 0 {
		log.Println("ciphertext is not a multiple of the block size")
		return C.CString("ciphertext is not a multiple of the block size")
	}

	mode := cipher.NewCBCDecrypter(block, iv)

	// CryptBlocks can work in-place if the two arguments are the same.
	mode.CryptBlocks(ciphertext, ciphertext)

	// If the original plaintext lengths are not a multiple of the block
	// size, padding would have to be added when encrypting, which would be
	// removed at this point. For an example, see
	// https://tools.ietf.org/html/rfc5246#section-6.2.3.2. However, it's
	// critical to note that ciphertexts must be authenticated (i.e. by
	// using crypto/hmac) before being decrypted in order to avoid creating
	// a padding oracle.

	log.Printf("%s\n", ciphertext)
	// Output: exampleplaintext
	return C.CString(fmt.Sprintf("%s\n", ciphertext))
}

func main() {
	
}

```
`Note that even if you want to compile into a dynamic library, you must have a main function. The above import "C" must have and must have a comment.
注意，即使是要编译成动态库，也要有main函数，上面的import "C"一定要有 而且一定要有注释`

	//export Aes128Des
`After testing, if there is no corresponding function found in the exported DLL library
经测试，如果没有这个导出的DLL库中找不到对应的函数`


##Compile go program
##编译go程序

First, switch the directory where the console is located to the directory where the go program is located, that is, the directory where libaes128.go is located.
首先，将控制台的所在目录切换到go程序的所在目录，即libaes128.go所在目录

A. Windows dynamic library
Execute the following command to generate a DLL dynamic link library:
A. Windows动态库
执行如下命令生成DLL动态链接库：

	go build -buildmode=c-shared -o libaes128.dll .\libaes128.go

`If the console does not report an error, the libhello.dll file will be generated in the current path.如果控制台没有报错，那么会在当前路径下生成libhello.dll文件`


B. Linux/Unix/macOS dynamic library
Execute the following command to generate the SO dynamic library:
B. Linux/Unix/macOS动态库
执行如下命令生成SO动态库：

	go build -buildmode=c-shared -o libaes128.so .\libaes128.go

##Called in java
##在java中调用

####A. Reference to JNA
There are two ways for Java to call Native's dynamic library. JNI and JNA, JNA is the latest way for Oracle to interact with Native. I will not mention more about it. I quote Baidu Encyclopedia's connection: https://baike.baidu. Com/item/JNA/8637274?fr=aladdin, friends in need can go and see. Here, we use the JNA approach, JNI's approach is basically abandoned, unless there are special needs, not much to say here, you can contact me for discussion. New Java project, I am using Maven for package management, so I directly reference JNA's dependencies:

####A. JNA的引用
Java调用Native的动态库有两种方式，JNI和JNA，JNA是Oracle最新推出的与Native交互的方式，具体介绍我就不多说了，引用百度百科的连接：https://baike.baidu.com/item/JNA/8637274?fr=aladdin，有需要的朋友可以去看看。 在这里，我们使用JNA的方式，JNI的方式基本废弃，除非有特殊需要，在这里不多说，有需要可以联系我讨论。 新建Java工程，我使用的是Maven做包管理，所以直接引用JNA的依赖：

	<dependency>
	      <groupId>net.java.dev.jna</groupId>
	      <artifactId>jna</artifactId>
	      <version>4.5.2</version>
	</dependency>

If you are not using the package management tool, you can directly download the Jar file, and download the address, which is also version 4.5.2: http://central.maven.org/maven2/net/java/dev/jna/jna /4.5.2/jna-4.5.2.jar
如果你没有使用包管理工具，可以直接下载Jar文件引入，下载地址也贴一下吧，也是4.5.2版本的： http://central.maven.org/maven2/net/java/dev/jna/jna/4.5.2/jna-4.5.2.jar


####B. Create an interface
We need to create an interface to map the functions in the DLL, then we can access the functions in the DLL through the instance of the interface.
####B. 创建接口
我们需要创建一个interface来映射DLL中的函数，之后我们可以通过interface的实例来访问DLL中的函数。

```
/*

package com.mycompany.mavenproject1;

import com.sun.jna.Library;
import com.sun.jna.Native;

public interface Ase128 extends Library {

    Ase128 INSTANCE = (Ase128) Native.loadLibrary("E:/testweb/protoctest/libaes128.dll", Ase128.class);

    String Aes128Des(GoString.ByValue key,GoString.ByValue data);
}

```

`Note that Aes128Des is the name of the function, it must be consistent with the name of the function written in Go. The first parameter of Native.loadLibrary() is a string, the name of the dynamic library to be loaded or the full path, which does not need to be added later. The .dll or .so suffix. The second parameter is the class name of the interface.注意，Aes128Des是函数名，一定要与Go中事先写好的函数名保持一致 Native.loadLibrary()的第一个参数是一个字符串，要加载的动态库的名称或全路径，后面不需要加.dll或者.so的后缀。第二个参数为interface的类名称。`


##C. Call
We create a new App class, as the entry class of the main method, do not need extra operations in the main method, just need to call, here we call the Sum method, at the same time pass iu3riubbku0dsia0, 890795015b67204ab42d16f9be3c7f19563f0e82dc992ff74a52e118d36e58ad394d87ea2a9ace75d9446eb91f6b1d2144ac343a6e6db973d41c7808ad26e6f2, you can see the console output: {"code": 449, "message": "The name is mismatching."}
##C. 调用
我们新建一个App类，作为main方法的入口类，在main方法中不需要多余的操作，只需要调用即可，在这里我们调用Aes128Des 方法，同时传如iu3riubbku0dsia0， 890795015b67204ab42d16f9be3c7f19563f0e82dc992ff74a52e118d36e58ad394d87ea2a9ace75d9446eb91f6b1d2144ac343a6e6db973d41c7808ad26e6f2，可以看到控制台输出：{"code": 449, "message": "The name is mismatching."}

```
/*

package com.mycompany.mavenproject1;

public class App {
    public static void main(String[] args) {
        String key ="iu3riubbku0dsia0";
        String data ="890795015b67204ab42d16f9be3c7f19563f0e82dc992ff74a52e118d36e58ad394d87ea2a9ace75d9446eb91f6b1d2144ac343a6e6db973d41c7808ad26e6f2";
        System.out.println(Ase128.INSTANCE.Aes128Des(new GoString.ByValue(key),new GoString.ByValue(data)));
    }
}

```

##D.Create a GoString!
##D.创建GoString！
We first use JNA to build a C structure type, then the problem comes, JNA char can be directly replaced by java String, then ptrdiff_t this stuff ... a little speechless, this is what? After a bit of operation Baidu and Google, I finally realized that this type is actually the value of the distance between two memory addresses. The data type is actually the long int in C. Here he represents the length of the string char. That is, the length of the string 呗~, knowing this is easy, we use the long type instead of it in Java. We create a new GoString class to correspond to the GoString structure in C, which is the string in the Go program. This piece has to be said, some people may not have used JNA. If you want to define a structure in JNA, you need to create a class. Inherited from com.sun.jna.Structure, people familiar with C should know (do not know it does not matter), there are usually two types of values ​​passed to C, one is a reference (that is, the type of pointer), one is fax Values, if done in JNA, we usually create two static inner classes in this struct class. These two inner classes inherit from this struct class and implement the Structure.ByValue and Structure.ByReference interfaces, where ByValue is passed. When the real value is used, ByReference is used when passing the reference. In summary, our GoString class should grow like this:
我们首先用JNA构建一个C的结构体类型，那么问题来了，JNA中char 可以直接用java的String来代替，那么ptrdiff_t这个玩意……有点无语，这是啥啊？经过一顿操作百度和谷歌，终于知道了，这个类型实际上是两个内存地址之间的距离的值，数据类型实际上就是C中的long int，在这里他表示的是字符串char 的长度，也就是字符串的长度呗~，知道这个就好办了，我们在Java中直接用long类型来代替它。 我们新建一个GoString类来对应C中的GoString结构体，也就是Go程序中的string，这块得说一下，有些人可能没有用过JNA，在JNA中若想定义一个结构体，需要创建一个类继承自com.sun.jna.Structure，熟悉C的人应该知道（不知道也没关系），向C中传值通常有两种，一种是传引用（就是传指针类型），一种是传真实值，在JNA里面做的话我们通常在这个结构体类中创建两个静态的内部类，这两个内部类继承自这个结构体类，并实现Structure.ByValue和Structure.ByReference接口，其中ByValue就是传真实值时候用的，ByReference就是传引用的时候用的，综上所述，我们的GoString类就应该长成这个样子：


```
/*

package com.mycompany.mavenproject1;

import com.sun.jna.Structure;
import java.util.ArrayList;
import java.util.List;


public class GoString extends Structure {

    public String str;
    public long length;

    public GoString() {
    }

    public GoString(String str) {
        this.str = str;
        this.length = str.length();
    }

    @Override
    protected List<String> getFieldOrder() {
        List<String> fields = new ArrayList<>();
        fields.add("str");
        fields.add("length");
        return fields;
    }

    public static class ByValue extends GoString implements Structure.ByValue {

        public ByValue() {
        }

        public ByValue(String str) {
            super(str);
        }
    }

    public static class ByReference extends GoString implements Structure.ByReference {

        public ByReference() {
        }

        public ByReference(String str) {
            super(str);
        }
    }

}

```
Ok, well, run:
好了好了好了，运行：

	{"code": 449, "message": "The name is mismatching."}
