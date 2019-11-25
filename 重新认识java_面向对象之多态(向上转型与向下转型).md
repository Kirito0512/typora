#### 1. 向上转型可以随便用

#### 2. 向下转型注意事项

- 向下转型的前提是，父类引用本身，指向的是子类对象(就是说，使用向下转型前，一定会先有向上转型)

- 向下转型，只能转型为本类对象

  > 来看一个例子：

  ```
  // 基类
  public class Animal {
      public void eat(){
          System.out.println("animal eatting...");
      }
  }
  
  // 子类cat
  public class Cat extends Animal{
  
      public void eat(){
  
          System.out.println("我吃鱼");
      }
  }
  
  // 子类dog
  public class Dog extends Animal{
  
      public void eat(){
  
          System.out.println("我吃骨头");
      }
  
      public void run(){
          System.out.println("我会跑");
      }
  }
  --------------------- 
  
  //还是上面的animal和cat dog
  Animal a = new Cat();
  Cat c = ((Cat) a);
  c.eat();
  //输出  我吃鱼
  Dog d = ((Dog) a);
  d.eat();
  // 报错 ： java.lang.ClassCastException：com.chengfan.animal.Cat cannot be cast to com.chengfan.animal.Dog
  Animal a1 = new Animal();
  Cat c1 = ((Cat) a1);
  c1.eat();
  // 报错 ： java.lang.ClassCastException：com.chengfan.animal.Animal cannot be cast to com.chengfan.animal.Cat
  ```

  

#### 3. 对多态的总结

- 多态，简言而之，就是对同一个行为具有多个不同的表现形式或形态的能力
- 多态分为：运行时多态与编译时多态(即重载与重写)
- 运行时多态的前提：继承或实现，重写，向上转型
- 向上转型与向下转型
- 继承连中，对象方法的调用顺序：`this.show(O)`, `super.show(O)`,`this.show((super) O)`, `super.show((super) O)`

#### 4. 参考

[重新认识java（五） ---- 面向对象之多态（向上转型与向下转型）](https://blog.csdn.net/qq_31655965/article/details/54746235)

[Java向上转型和向下转型](https://blog.csdn.net/qq_25827845/article/details/75530403)