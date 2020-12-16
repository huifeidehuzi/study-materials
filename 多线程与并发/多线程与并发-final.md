# 多线程与并发-final

## 如何理解private所修饰的方法是隐式的final?

final类中的所有方法都隐式为final，因为无法覆盖他们，所以在final类中给任何方法添加final关键字是没有任何意义的



## final类型的类如何拓展?

组合模式，装饰器模式都可以

```java
// 拿String举例
public class FinalExtExample{
  
  private String extStr;
  
  public FinalExtExample(String str){
    this.extStr = str;
  }
  
  // 使用String原有的方法
  public int indexOf(String pr){
    return extStr.indexOf(pr);
  }
  // 扩展自己的方法
  public void show(){
    System.out.println("我是扩展方法");
  }
}
```



## final方法可以被重载吗?

可以

```java
public class FinalExampleParent {
    public final void test() {
    }

    public final void test(String str) {
    }
}
```



## 所有的final修饰的字段都是编译期常量吗?

不是，看例子

```java
public void test01(){
    // 编译期常量
    final int i = 0;

    // 非编译期常量
    // k值只有当程序运行随机出来后才能确定
    // 被随机出来初始化后，k值无法被更改
    final int k = new Random().nextInt();
  	
  	// 注意:被static final 修饰的变量必须赋初始值
		// 一旦复制后，无论调用多少次，它的值都不会变
    // 因为一个既是static又是final的字段只占据一段不能改变的存储空间，被初始化后就不能被修改
    // 因为static关键字所修饰的字段并不属于一个对象，而是属于这个类的
    static final int j = new Random().nextInt();
  	
    // 非编译期常量
    // 允许在使用前赋值
    final int d;
  	
    // d在此处赋值
  	public test01(int d){
      this.d = d;
		}
}
```

