所有的Editor相关代码，必须存在于Assets目录下的任意路径的名为Editor文件夹内。U3D会自动为这些文件生成编辑器工程。

# [c#反射进行编辑器扩展](https://zhuanlan.zhihu.com/p/25891306)

 c#中的反射指的是在运行时可以获取类型等信息的能力。[参见这里](http://www.cnblogs.com/wangshenhe/p/3256657.html)
 
 https://zhuanlan.zhihu.com/p/25891306
 
 文中提到了一个反编译过的Unity源码库，Editor是用c#写的，可以看到一些私有的API，从而通过反射调用）
 
km上的文章摘抄：
当Project窗口中的某个代码被拖到一个场景物体上时，Unity编辑器会根据代码名称在项目的程序集中查找对应的类型，并使用反射方法判断此类型是否继承于MonoBehaviour。如果是，则通过反射方法创建这个类型的实例并加入到该场景物体的组件列表中。Inspector窗口上的属性的显示也是同样道理——使用反射方法扫描这个类型的所有成员，得到其中“可编辑的成员”，根据成员的类型在Inspector窗口上显示编辑控件，并通过反射方法将编辑后的值设置到组件实例上。
 