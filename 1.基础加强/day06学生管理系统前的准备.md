## 1. 键盘录入问题

- 先使用nextInt()再使用nextLine()会出现问题
- 所以项目中所有的键盘录入都使用**nextLine()**

## 2. 集合存储自定义对象问题

```java
ArrayList<Student> list = new ArrayList<>();

// 键盘录入姓名
String name = sc.nextLine();
// 键盘录入年龄, 年龄以String的形式录入, 再转换成int
int age = Integer.parseInt(sc.nextLine());
// 使用上面的姓名和年龄, 创建学生对象
Student s = new Student(name, 23);
// 将对象添加到集合中
list.add(s);
```

## 3. 工具类问题

- 工具类的特点
  - 类中的成员都是static修饰, 可以直接使用 `类名.`调用
  - 将构造方法使用private修饰, 目的不让别人创建对象

## 4. 共享数据自增问题

```java
public class Config {
    public static int configId = 1;
}
// ==============================================
public class Student {

    private int id;
    private String name;
    private int age;

    public Student() {
    }

    public Student(String name, int age) {
        this.id = Config.configId++;
        this.name = name;
        this.age = age;
    }
}
```



## 5. 模版方法设计模式

- 模版方法: 定义了代码执行的流程, 但是在流程中有一些地方可能会发生改变
- 将可能发生改变的功能, 定义成抽象方法
- 具体的实现由子类来完成