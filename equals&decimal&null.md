## 比较值的内容，除了基本类型只能使用==外，其他引用类型都需要使用equals

说明：注意Integer类型和String类型，使用==有时也能得到貌似正确的结果：

```

Integer a = 127; //Integer.valueOf(127)  缓存机制
Integer b = 127; //Integer.valueOf(127)
log.info("\nInteger a = 127;\n" +
        "Integer b = 127;\n" +
        "a == b ? {}",a == b);    // true   

Integer c = 128; //Integer.valueOf(128)
Integer d = 128; //Integer.valueOf(128)
log.info("\nInteger c = 128;\n" +
        "Integer d = 128;\n" +
        "c == d ? {}", c == d);   //false

Integer e = 127; //Integer.valueOf(127)
Integer f = new Integer(127); //new instance
log.info("\nInteger e = 127;\n" +
        "Integer f = new Integer(127);\n" +
        "e == f ? {}", e == f);   //false

Integer g = new Integer(127); //new instance
Integer h = new Integer(127); //new instance
log.info("\nInteger g = new Integer(127);\n" +
        "Integer h = new Integer(127);\n" +
        "g == h ? {}", g == h);  //false

Integer i = 128; //unbox
int j = 128;
log.info("\nInteger i = 128;\n" +
        "int j = 128;\n" +
        "i == j ? {}", i == j); //true
```

```

String a = "1";
String b = "1";
log.info("\nString a = \"1\";\n" +
        "String b = \"1\";\n" +
        "a == b ? {}", a == b); //true    字符串驻留机制，a,b指向常量池的相同字符串

String c = new String("2");
String d = new String("2");
log.info("\nString c = new String(\"2\");\n" +
        "String d = new String(\"2\");" +
        "c == d ? {}", c == d); //false

String e = new String("3").intern();
String f = new String("3").intern();
log.info("\nString e = new String(\"3\").intern();\n" +
        "String f = new String(\"3\").intern();\n" +
        "e == f ? {}", e == f); //true    使用String提供的intern方法也会走常量池机制

String g = new String("4");
String h = new String("4");
log.info("\nString g = new String(\"4\");\n" +
        "String h = new String(\"4\");\n" +
        "g == h ? {}", g.equals(h)); //true
```



## 对于自定义类型，如果要实现Comparable，请记得equals、hashCode、compareTo三者逻辑一致

说明：Lombok 的 @EqualsAndHashCode 注解实现 equals 和 hashCode 的时候，默认使用类型所有非 static、非 transient 的字段，且不考虑父类。如果希望改变这种默认行为，可以使用 @EqualsAndHashCode.Exclude 排除一些字段，并设置 callSuper = true 来让子类的 equals 和 hashCode 调用父类的相应方法。



## 使用BigDecimal表示和计算浮点数，且务必使用字符串的构造方法来初始化BigDecimal



## 浮点数的字符串格式化也要通过BigDecimal进行

```
BigDecimal num1 = new BigDecimal("3.35");
BigDecimal num2 = num1.setScale(1, BigDecimal.ROUND_DOWN);
System.out.println(num2);
BigDecimal num3 = num1.setScale(1, BigDecimal.ROUND_HALF_UP);
System.out.println(num3);
```



## 只比较BigDecimal的value，使用compareTo方法，不要使用equals

```
System.out.println(new BigDecimal("1.0").equals(new BigDecimal("1"))) //false
System.out.println(new BigDecimal("1.0").compareTo(new BigDecimal("1"))==0)//true
```



## 进行数值运算时要小心溢出问题，使用Math.xxxExat方法进行运算，可抛出溢出异常



## NullPointerException处理规范：

- 针对参数值是Integer等包装类型，使用 Optional.ofNullable 来构造一个 Optional，避免自动拆箱出现异常
- 针对字符串比较，可以把字面量放在前面，对于两个可能为null的字符串变量的比较，可以使用Objects.equals，它会做判空处理
- HashMap 的 Key 和 Value 可以存入 null，而 ConcurrentHashMap 看似是 HashMap 的线程安全版本，却不支持 null 值的 Key 和 Value
- 对于级联调用，使用Optional.ofNullable包装
- 对于返回值，使用Optional.ofNullable 包装

```
反例：
private List<String> wrongMethod(FooService fooService, Integer i, String s, String t) {
    log.info("result {} {} {} {}", i + 1, s.equals("OK"), s.equals(t),
            new ConcurrentHashMap<String, String>().put(null, null));
    if (fooService.getBarService().bar().equals("OK"))
        log.info("OK");
    return null;
}

@GetMapping("wrong")
public int wrong(@RequestParam(value = "test", defaultValue = "1111") String test) {
    return wrongMethod(test.charAt(0) == '1' ? null : new FooService(),
            test.charAt(1) == '1' ? null : 1,
            test.charAt(2) == '1' ? null : "OK",
            test.charAt(3) == '1' ? null : "OK").size();
}

class FooService {
    @Getter
    private BarService barService;

}

class BarService {
    String bar() {
        return "OK";
    }
}
```

```
正例：
private List<String> wrongMethod(FooService fooService, Integer i, String s, String t) {
    log.info("result {} {} {} {}", i + 1, s.equals("OK"), s.equals(t),
            new ConcurrentHashMap<String, String>().put(null, null));
    if (fooService.getBarService().bar().equals("OK"))
        log.info("OK");
    return null;
}

@GetMapping("wrong")
public int wrong(@RequestParam(value = "test", defaultValue = "1111") String test) {
    return wrongMethod(test.charAt(0) == '1' ? null : new FooService(),
            test.charAt(1) == '1' ? null : 1,
            test.charAt(2) == '1' ? null : "OK",
            test.charAt(3) == '1' ? null : "OK").size();
}

class FooService {
    @Getter
    private BarService barService;

}

class BarService {
    String bar() {
        return "OK";
    }
}
```

说明：使用判空方式或Optional方式来避免出现空指针异常，不一定是解决问题的最好方式，空指针没有出现可能隐藏了更深的Bug



## Mysql中有关NULL的三个坑

- MySQL 中 sum 函数没统计到任何记录时，会返回 null 而不是 0，可以使用 IFNULL 函数把 null 转换为 0；
- MySQL 中 count 字段不统计 null 值，COUNT(*) 才是统计所有记录数量的正确方式。
- MySQL 中使用诸如 =、<、> 这样的算数比较操作符比较 NULL 的结果总是 NULL，这种比较就显得没有任何意义，需要使用 IS NULL、IS NOT NULL 或 ISNULL() 函数来比较。



## 避免判等、数值计算、空值处理等低级代码错误的最有效方式——代码静态检查

[P3C](https://github.com/alibaba/p3c)

CheckStyle

FindBugs

SonarQube



## 定位NullPointerException的神器推荐

[Arthas（阿尔萨斯）](https://arthas.aliyun.com/doc/)

当你遇到以下类似问题而束手无策时，`Arthas`可以帮助你解决：

1. 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
2. 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
3. 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
4. 线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
5. 是否有一个全局视角来查看系统的运行状况？
6. 有什么办法可以监控到JVM的实时运行状态？
7. 怎么快速定位应用的热点，生成火焰图？