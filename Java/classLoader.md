# ClassLoader

```
class SingleTon {
	
	private static SingleTon singleTon = new SingleTon();
    
	public static int count1;

    public static int count2 = 0;

    public SingleTon() {
		count3=1;
	    count1++;
	    count2++;
	}
    
    {
    	count1 +=1;
    }
    
    public int count3 = count1+count2;
    
    {
    	count3 +=3;
    }
  
	
  
	
    public static SingleTon getInstance() {  
        return singleTon;  
    }  
}  
  
public class Test {
    public static void main(String[] args) {
    	SingleTon singleTon = SingleTon.getInstance();  
        System.out.println("count1=" + singleTon.count1);  
        System.out.println("count2=" + singleTon.count2);
        System.out.println("count3=" + singleTon.count3);
    }
}  
```

结果

```
count1=2
count2=0
count3=1
```

## `<clinit>与<init>`

* `<clinit>`：类的初始化方法，类初始化过程只执行一次，初始化的类会被封装成`class`对象放入永久带（1.8之前）或元数据区（1.8之后）
* `<init>`：实例构造器，实例的初始化方法，类被实例化时调用

### `<clinit>`的作用

1. 父类静态变量初始化和父类的静态语句块
2. 子类的静态变量初始化和子类的静态语句块

### 何时执行类初始化（主动引用）

1. 创建类的实例; （先调用`<clinit>`，再调用`<init>`）
2. 访问类的静态变量
3. 访问类的静态方法
4. Class.forName("XXXXX");（initialize为true时，默认就是true）
5. 当初始化一个类时，发现其父类还未初始化，则先对父类初始化
6. 虚拟机启动时，定义了main()方法的类先初始化

### 被动引用

1. 子类调用父类的静态变量，子类不会被初始化。只有父类被初始化。对于静态字段，只有直接定义这个字段的类才会被初始化
2. 通过数组定义来引用类，不会触发类的初始化
3. 访问类的常量，不会初始化类； （常量在类加载第三阶段准备阶段就被初始化了，会存入调用类的常量池中，本质上并没有直接引用定义常量的类）
4. 通过类名获取Class对象，不会触发类的初始化；  （在类加载第一阶段已经生成Class对象在内存里面了）
5. 通过Class.forName加载指定类时，指定参数initialize为false时，也不会触发类初始化，其实这个参数是告诉虚拟机，是否要对类进行初始化；
6. 通过ClassLoader默认的loadClass方法，也不会触发初始化动作；

### `<init>`的作用

1. 父类变量初始化和父类语句块
2. 父类构造函数
3. 子类变量初始化和子类语句块
4. 子类构造函数

#### 何时执行类的初始化

1. new操作符实例化对象
2. 调用Class或java.lang.reflect.Constructor对象的newInstance()方法
3. 调用已经实例化对象的clone()方法
4. 通过java.io.ObjectInputStream类的getObject()方法反序列化

