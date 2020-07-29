# junit
## 1. 依赖管理
~~~
<dependency>
    <groupId>org.jmockit</groupId>
    <artifactId>jmockit</artifactId>
    <version>1.36</version>
    <scope>test</scope>
  </dependency>
~~~

## 2. JUnit4.x及以下用户特别注意事项

如果你是通过mvn test来运行你的测试程序 , 请确保JMockit的依赖定义出现在JUnit的依赖之前

~~~
<!-- 先声明jmockit的依赖 -->
   <dependency>
    <groupId>org.jmockit</groupId>
    <artifactId>jmockit</artifactId>
    <version>1.36</version>
    <scope>test</scope>
  </dependency>
<!-- 再声明junit的依赖 -->
   <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>4.9</version>
    <scope>test</scope>
  </dependency>
~~~

## 3. JMockit Coverage配置
覆盖率功能
~~~
 <plugin>
     <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
    <argLine>-javaagent:"${settings.localRepository}/org/jmockit/jmockit/1.36/jmockit-1.36.jar=coverage"</argLine>
    <disableXmlReport>false</disableXmlReport>
    <systemPropertyVariables>
    <coverage-output>html</coverage-output>
    <coverage-outputDir>${project.build.directory}/codecoverage-output</coverage-outputDir>
    <coverage-metrics>all</coverage-metrics>
    </systemPropertyVariables>
    </configuration>
 </plugin>
~~~

## 4. 程序结构
（1）测试属性、参数

a)测试属性：测试类的一个属性，用途测试类的**所有测试方法**。

b)测试参数：仅作用于当前测试方法。

测试方法：

step1:录制（record）：录制某个方法的调用，传入某个参数，返回某个结果；

step2:重放（replay）：重放测试逻辑。

step3:验证（Verification)：重放后的验证，验证是否被调用，调用多少次。

## 5. API介绍

（1）@Mocked

@Mocked修饰的类/接口，是告诉JMockit，帮我生成一个Mocked对象，这个对象方法（包含静态方法)返回默认值。
即如果返回类型为原始类型(short,int,float,double,long)就返回0，如果返回类型为String就返回null，如果返回类型是其它引用类型，则返回这个引用类型的Mocked对象

（2）@Injectable

只针对修饰的实例；@Mocked是针对修饰的类的所有实例；

对类的静态方法、构造函数没有影响

（3）@Tested

~~~
   //@Tested与@Injectable搭配使用
public class TestedAndInjectable {
   //@Tested修饰的类，表示是我们要测试对象,在这里表示，我想测试订单服务类。JMockit也会帮我们实例化这个测试对象
   @Tested
   OrderService orderService;
   //测试用户ID
   long testUserId = 123456l;
   //测试商品id
   long testItemId = 456789l;
 
   // 测试注入方式
   @Test
   public void testSubmitOrder(@Injectable MailService mailService, 
     @Injectable UserCheckService userCheckService) {
    new Expectations() {
       {
         // 当向testUserId发邮件时，假设都发成功了
         mailService.sendMail(testUserId, anyString);
         result = true;
        // 当检验testUserId的身份时，假设该用户都是合法的
         userCheckService.check(testUserId);
        result = true;
         }
         };
    // JMockit帮我们实例化了mailService了，并通过OrderService的构造函数，注入到orderService对象中。 
    //JMockit帮我们实例化了userCheckService了，并通过OrderService的属性，注入到orderService对象中。 
    Assert.assertTrue(orderService.submitOrder(testUserId, testItemId));
    }
}
~~~

@Tested表示被测试对象。如果该对象没有赋值，JMockit会去实例化它，若@Tested的构造函数有参数，
则JMockit通过在测试属性&测试参数中查找@Injectable修饰的Mocked对象注入@Tested对象的构造函数来实例化，
不然，则用无参构造函数来实例化。除了构造函数的注入，JMockit还会通过属性查找的方式，把@Injectable对象注入到@Tested对象中。

（4）@Capturing

（5）Expectations

~~~
new Expectations() {
    // 这是一个Expectations匿名内部类
    {
          // 这是这个内部类的初始化代码块，我们在这里写录制脚本，脚本的格式要遵循下面的约定
        //方法调用(可是类的静态方法调用，也可以是对象的非静态方法调用)
        //result赋值要紧跟在方法调用后面
        //...其它准备录制脚本的代码
        //方法调用
        //result赋值
    }
};
~~~

案例：

a)引用外部类的Mock对象
~~~
public class ExpectationsTest {
    @Mocked
    Calendar cal;
 
    @Test
    public void testRecordOutside() {
        new Expectations() {
            {
                // 对cal.get方法进行录制，并匹配参数 Calendar.YEAR
                cal.get(Calendar.YEAR);
                result = 2016;// 年份不再返回当前小时。而是返回2016年
                // 对cal.get方法进行录制，并匹配参数 Calendar.HOUR_OF_DAY
                cal.get(Calendar.HOUR_OF_DAY);
                result = 7;// 小时不再返回当前小时。而是返回早上7点钟
            }
        };
        Assert.assertTrue(cal.get(Calendar.YEAR) == 2016);
        Assert.assertTrue(cal.get(Calendar.HOUR_OF_DAY) == 7);
        // 因为没有录制过，所以这里月份返回默认值 0
        Assert.assertTrue(cal.get(Calendar.DAY_OF_MONTH) == 0);
    }
 
}
~~~
这种方法会mock掉这个类的所有方法。

b)通过构建函数注入类/对象来录制

~~~
//通过Expectations对其构造函数mock对象进行录制
public class ExpectationsConstructorTest2 {
 
    // 把类传入Expectations的构造函数
    @Test
    public void testRecordConstrutctor1() {
        Calendar cal = Calendar.getInstance();
        // 把待Mock的类传入Expectations的构造函数，可以达到只mock类的部分行为的目的
        new Expectations(Calendar.class) {
            {
                // 只对get方法并且参数为Calendar.HOUR_OF_DAY进行录制
                cal.get(Calendar.HOUR_OF_DAY);
                result = 7;// 小时永远返回早上7点钟
            }
        };
        Calendar now = Calendar.getInstance();
        // 因为下面的调用mock过了，小时永远返回7点钟了
        Assert.assertTrue(now.get(Calendar.HOUR_OF_DAY) == 7);
        // 因为下面的调用没有mock过，所以方法的行为不受mock影响，
        Assert.assertTrue(now.get(Calendar.DAY_OF_MONTH) == (new Date()).getDate());
    }
 
    // 把对象传入Expectations的构造函数
    @Test
    public void testRecordConstrutctor2() {
        Calendar cal = Calendar.getInstance();
        // 把待Mock的对象传入Expectations的构造函数，可以达到只mock类的部分行为的目的，但只对这个对象影响
        new Expectations(cal) {
            {
                // 只对get方法并且参数为Calendar.HOUR_OF_DAY进行录制
                cal.get(Calendar.HOUR_OF_DAY);
                result = 7;// 小时永远返回早上7点钟
            }
        };
 
        // 因为下面的调用mock过了，小时永远返回7点钟了
        Assert.assertTrue(cal.get(Calendar.HOUR_OF_DAY) == 7);
        // 因为下面的调用没有mock过，所以方法的行为不受mock影响，
        Assert.assertTrue(cal.get(Calendar.DAY_OF_MONTH) == (new Date()).getDate());
 
        // now是另一个对象，上面录制只对cal对象的影响，所以now的方法行为没有任何变化
        Calendar now = Calendar.getInstance();
        // 不受mock影响
        Assert.assertTrue(now.get(Calendar.HOUR_OF_DAY) == (new Date()).getHours());
        // 不受mock影响
        Assert.assertTrue(now.get(Calendar.DAY_OF_MONTH) == (new Date()).getDate());
    }
}
~~~

（6）Mockup&@Mock

~~~
public class MockUpTest {
 
    @Test
    public void testMockUp() {
        // 对Java自带类Calendar的get方法进行定制
        // 只需要把Calendar类传入MockUp类的构造函数即可
        new MockUp<Calendar>(Calendar.class) {
            // 想Mock哪个方法，就给哪个方法加上@Mock， 没有@Mock的方法，不受影响
            @Mock
            public int get(int unit) {
                if (unit == Calendar.YEAR) {
                    return 2017;
                }
                if (unit == Calendar.MONDAY) {
                    return 12;
                }
                if (unit == Calendar.DAY_OF_MONTH) {
                    return 25;
                }
                if (unit == Calendar.HOUR_OF_DAY) {
                    return 7;
                }
                return 0;
            }
        };
        // 从此Calendar的get方法，就沿用你定制过的逻辑，而不是它原先的逻辑。
        Calendar cal = Calendar.getInstance(Locale.FRANCE);
        Assert.assertTrue(cal.get(Calendar.YEAR) == 2017);
        Assert.assertTrue(cal.get(Calendar.MONDAY) == 12);
        Assert.assertTrue(cal.get(Calendar.DAY_OF_MONTH) == 25);
        Assert.assertTrue(cal.get(Calendar.HOUR_OF_DAY) == 7);
        // Calendar的其它方法，不受影响
        Assert.assertTrue((cal.getFirstDayOfWeek() == Calendar.MONDAY));
 
    }
}
~~~
缺点

a)一个类有多个实例。只对其中某1个实例进行mock。 
 最新版的JMockit已经让MockUp不再支持对实例的Mock了。1.19之前的老版本仍支持。

b)AOP动态生成类的Mock。

c)对类的所有方法都需要Mock时，书写MockUp的代码量太大。

（7）Verification

~~~
new Verifications() {
    // 这是一个Verifications匿名内部类
    {
          // 这个是内部类的初始化代码块，我们在这里写验证脚本，脚本的格式要遵循下面的约定
        //方法调用(可是类的静态方法调用，也可以是对象的非静态方法调用)
        //times/minTimes/maxTimes 表示调用次数的限定要求。赋值要紧跟在方法调用后面，也可以不写（表示只要调用过就行，不限次数）
        //...其它准备验证脚本的代码
        //方法调用
        //times/minTimes/maxTimes 赋值
    }
};
  
还可以再写new一个Verifications，只要出现在重放阶段之后均有效。
new Verifications() {
       
    {
         //...验证脚本
    }
};
~~~

