---
layout:     post
title:      责任链模式
date:       2018-06-02 17:15:00
categories: 设计模式
tags:       设计模式
---

* 文章目录
{:toc}

【学习难度：★★★☆☆，使用频率：★★☆☆☆】

>职责链上的处理者负责处理请求，客户只需要将请求发送到职责链上即可，无须关心请求的处理细节和请求的传递，所以职责链**将请求的发送者和请求的处理者解耦**了，这就是职责链的设计动机。



#### 模式定义
&emsp;&emsp;**避免请求发送者与接收者耦合在一起，让多个对象都有可能接收请求，将这些对象连接成一条链，并且沿着这条链传递请求，直到有对象处理它为止，这就是职责链模式。**

&emsp;&emsp;在职责链模式中最关键的一点就是客户提交请求后，请求沿着链往下传递直到有一个处理者处理它，在这里客户无需关心它的请求是哪个处理者来处理，反正总有一个处理者会处理它的请求。

&emsp;&emsp;在这里客户端和处理者都没有对方明确的信息，同时处理者也不知道职责链中的结构。所以职责链可以简化对象的相互连接，他们只需要保存一个指向其后续者的引用，而不需要保存所有候选者的引用。

&emsp;&emsp;在职责链模式中我们可以随时随地的增加或者更改一个处理者，甚至可以更改处理者的顺序，增加了系统的灵活性。处理灵活性是增加了，但是有时候可能会导致一个请求无论如何也得不到处理，它会被放置在链末端，这个既是职责链的优点也是缺点。

#### UML图

---
在公司报销时，涉及到金额比较大的时候，往往不能由老大直接审批，而是要视金额逐级往上（请假也是）【万恶的资本主义】。下面我们看一个具体的例子：

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/chain/chain_uml.png)
在责任链模式中，有一个链式的处理流，在我们这个例子中是上下级关系形成的链式结构，当下级处理不了报销申请时，需要提交给上级继续审批，以此形成的责任链。

#### 具体代码

---

Leader角色：

```java
public abstract class Leader {
    private Leader successor;

    public Leader(Leader successor) {
        this.successor = successor;
    }

    public Leader getSuccessor() {
        return successor;
    }

    public abstract void handleRequest(Reimbursement reimbursement);

    public void approve(String role, Reimbursement reimbursement) {
        System.out.println(String.format("%s has approved %s's reimbursement, money: $%s", role,
                reimbursement.getUsername(), reimbursement.getMoney()));
    }
}
```

不同的角色：

```java
public class ProjectManager extends Leader {

    public ProjectManager(Leader successor) {
        super(successor);
    }

    @Override
    public void handleRequest(Reimbursement reimbursement) {
        int money = reimbursement.getMoney();
        if (money <= 500) {
            approve("Project manager", reimbursement);
        }else {
            Leader successor = getSuccessor();
            successor.handleRequest(reimbursement);
        }
    }
}

public class DeptManager extends Leader {

    public DeptManager(Leader successor) {
        super(successor);
    }

    @Override
    public void handleRequest(Reimbursement reimbursement) {
        int money = reimbursement.getMoney();
        if (money <= 2000) {
            approve("Dept manager", reimbursement);
        }else {
            Leader successor = getSuccessor();
            successor.handleRequest(reimbursement);
        }
    }
}

public class CTO extends Leader {
    public CTO(Leader successor) {
        super(successor);
    }

    @Override
    public void handleRequest(Reimbursement reimbursement) {
        int money = reimbursement.getMoney();
        if (money <= 10000) {
            approve("CTO", reimbursement);
        }else {
            Leader successor = getSuccessor();
            successor.handleRequest(reimbursement);
        }
    }
}

public class CEO extends Leader{

    public CEO() {
        //CEO没有上级！
        super(null);
    }

    @Override
    public void handleRequest(Reimbursement reimbursement) {
        int money = reimbursement.getMoney();
        if (money > 10000) {
            //只有婷婷可以报销1万元以上，其他人想都别想，你懂的~
            if (reimbursement.getUsername().equals("tingting")) {
                approve("CEO", reimbursement);
            }else {
                System.out.println(String.format("CEO has rejected %s's reimbursement, money: $%s",
                        reimbursement.getUsername(), reimbursement.getMoney()));
            }
        }
    }
}
```

报销单Reimbursement:
```java
public class Reimbursement {
    //Reimbursement: 报销单
    private String username;
    private int money;

    public Reimbursement(String username, int money) {
        this.username = username;
        this.money = money;
    }

    public String getUsername() {
        return username;
    }

    public int getMoney() {
        return money;
    }
}
```

测试类Client:

```java
public class Client {
    public static void main(String[] args) {
        //首先构造责任链
        CEO ceo = new CEO();
        CTO cto = new CTO(ceo);
        DeptManager deptManager = new DeptManager(cto);
        ProjectManager projectManager = new ProjectManager(deptManager);

        Reimbursement zhangsan380 = new Reimbursement("zhangsan", 380);
        projectManager.handleRequest(zhangsan380);

        Reimbursement lishi1220 = new Reimbursement("lishi", 1220);
        projectManager.handleRequest(lishi1220);

        Reimbursement wangwu4400 = new Reimbursement("wangwu", 4400);
        projectManager.handleRequest(wangwu4400);

        Reimbursement zhangsan8880 = new Reimbursement("zhangsan", 8880);
        projectManager.handleRequest(zhangsan8880);

        Reimbursement lishi14900 = new Reimbursement("lishi", 14900);
        projectManager.handleRequest(lishi14900);

        Reimbursement tingting88888 = new Reimbursement("tingting", 88888);
        projectManager.handleRequest(tingting88888);
    }
}
```

#### 结果

---

![image](http://oc26wuqdw.bkt.clouddn.com/2018/6/pattern/chain/chain_result.png)

不得不感叹下，tingting果然还是有魅力啊……
