---
layout:     post
title:      组合模式
date:       2018-06-02 11:25:00
categories: 设计模式
tags:       设计模式
---

* 文章目录
{:toc}

【学习难度：<font color=ff0000>★★★☆☆</font>，使用频率：<font color=ff0000>★★★★☆</font>】

#### 模式动机

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/composite/composite_filesystem.png)
&emsp;&emsp;我们对于这个图片肯定会非常熟悉，这是mac系统的文件结构，对于这样的结构我们称之为树形结构。在数据结构中我们了解到可以通过调用某个方法来遍历整个树，当我们找到某个叶子节点后，就可以对叶子节点进行相关的操作。我们可以将这颗树理解成一个大的容器，容器里面包含很多的成员对象，这些成员对象即可是容器对象也可以是叶子对象。但是由于容器对象和叶子对象在功能上面的区别，使得我们在使用的过程中必须要区分容器对象和叶子对象，但是这样就会给客户带来不必要的麻烦，作为客户而已，它始终希望能够一致的对待容器对象和叶子对象。这就是组合模式的设计动机：
<font color=#ff0000 size=4 face="黑体">**组合模式定义了如何将容器对象和叶子对象进行递归组合，使得客户在使用的过程中无须进行区分，可以对他们进行一致的处理。**</font>




#### 模式定义

---

&emsp;&emsp;组合模式组合多个对象形成树形结构以表示“整体-部分”的结构层次。

&emsp;&emsp;组合模式对单个对象(叶子对象)和组合对象(组合对象)具有一致性，它将对象组织到树结构中，可以用来描述整体与部分的关系。同时它也模糊了简单元素(叶子对象)和复杂元素(容器对象)的概念，使得客户能够像处理简单元素一样来处理复杂元素，从而使客户程序能够与复杂元素的内部结构解耦。

&emsp;&emsp;上面的图展示了计算机的文件系统，文件系统由文件和目录组成，目录下面也可以包含文件或者目录，计算机的文件系统是用递归结构来进行组织的，对于这样的数据结构是非常适用使用组合模式的。

&emsp;&emsp;在使用组合模式中需要注意一点也是组合模式最关键的地方：叶子对象和组合对象实现相同的接口。这就是组合模式能够将叶子节点和对象节点进行一致处理的原因。
      
#### UML图

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/composite/composite_uml.png)
组合模式中的角色：
1. Component：组合中的对象声明接口，在适当的情况下，实现所有类共有接口的默认行为。声明一个接口用于访问和管理Component子部件。 
2. Leaf：叶子对象。叶子结点没有子结点。 
3. Composite：容器对象，定义有枝节点行为，用来存储子部件，在Component接口中实现与子部件有关操作，如增加(add)和删除(remove)等。

可以想下三个角色分别对应于UML图中的哪个类

#### 具体代码

---

Component角色：

```java
public abstract class File {
    private String name;

    public File(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public abstract void display();
}
```

Composite角色：

```java
public class Folder extends File {

    private List<File> files;

    public Folder(String name) {
        super(name);
        //新创建的文件夹应该是“空”的
        this.files = new ArrayList<File>();
    }

    @Override
    public void display() {
        System.out.println(getName());
        for (File file : files) {
            file.display();
        }
    }

    public void addFile(File file){
        files.add(file);
    }

    public void removeFile(File file) {
        files.remove(file);
    }
}
```

Leaf角色:
```java
public class ImageFile extends File {

    public ImageFile(String name) {
        super(name);
    }

    @Override
    public void display() {
        System.out.println("Image file: " + getName());
    }
}

public class TextFile extends File {

    public TextFile(String name) {
        super(name);
    }

    @Override
    public void display() {
        System.out.println("Text file: " + getName());
    }
}

public class VideoFile extends File {
    public VideoFile(String name) {
        super(name);
    }

    @Override
    public void display() {
        System.out.println("Video file: " + getName());
    }
}
```

测试类Client:

```java
public class Client {
    public static void main(String[] args) {
        /** Step 1.Initial root folder */
        Folder root = new Folder("Root folder");
        root.addFile(new VideoFile("Get started.mp4"));

        Folder imageFolder = new Folder("Images folder");
        imageFolder.addFile(new ImageFile("1.jpg"));
        imageFolder.addFile(new ImageFile("2.jpg"));
        ImageFile image3 = new ImageFile("3.jpg");
        imageFolder.addFile(image3);

        Folder studyFolder = new Folder("Study folder");
        studyFolder.addFile(new TextFile("Python guide.txt"));
        studyFolder.addFile(new TextFile("Thinking in Java.txt.txt"));

        root.addFile(imageFolder);
        root.addFile(studyFolder);

        /** Step 2.Remove image file: 3.jpg, Add text file: Head first php.txt */
//        imageFolder.removeFile(image3);
//        studyFolder.addFile(new TextFile("Head first php.txt"));

        root.display();
    }
}
```

#### 结果

**Step 1:**
![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/composite/composite_result_1.png)

**Step 2:**
![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/composite/composite_result_2.png)
