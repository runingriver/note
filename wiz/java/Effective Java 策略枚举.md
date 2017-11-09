1. 优先使用基本类型而不是装箱类型，且要当心无意识的自动装箱。但是小对象的创建和销毁代价非常廉价，创建附加对象使得程序更加清晰简洁是好事。
2. 只要类自己管理内存，我们就应该警惕内存泄露的问题：
eg：
```
Object result = elements[--size];    //栈的pop操作，就是弹出一个对象。
elements[size] =null;
//...
return result;
```
分析：elements是一个数组我们自己管理，但是如果不指向null，就会有一个引用一直存在，不会被GC。

3. 使用for-each读取集合！
4. 用enum代替int常量
`int`常量是编译时的常量，它会被编译到使用它的客户端中，如果`int`常量发生变化，客户端就必须重新编译！
枚举：是通过公有的静态final域为每个枚举常量导出实例的类（每个枚举都可以看做一个实例），枚举才是真正的final。
枚举类型中的抽象方法必须被它所有常量中的具体方法所覆盖！称作：特定于常量的方法实现！

###　策略枚举
```
/**
 * 策略枚举：拿计算员工工资为例
 * 平时加班和周末加班计算方法不一样！
 */
enum NormalEnum {
    MONDAY,TUESDAY,WEDNESDAY, THURSDAY,FRIDAY,SATURDAY,SUNDAY;
    private static final int HOURS_PER_SHIFT = 8;
    double pay(double hoursWorked, double payRate) {
        double basePay = hoursWorked * payRate;
        double overtimePay;
        switch (this) {
            case SATURDAY:case SUNDAY:  //周末加班时间计算
                overtimePay = hoursWorked*payRate/2;
                default:    //平时的加班时间计算
                    overtimePay = hoursWorked <= HOURS_PER_SHIFT ?
                            0 : (hoursWorked - HOURS_PER_SHIFT) * payRate / 2;
                    break;
        }
        return basePay + overtimePay;
    }
/**
 * switch特定常量的计算
 * 这段代码如果添加一个元素到该枚举中，如果忘记给枚举switch值，那就会很麻烦了。
 */
}
/**
 * 策略枚举：更加安全，灵活
 * 舍弃了switch对特定常量的依赖！解耦了常量和计算方法
 */
public enum StrategyEnum{
    MONDAY(PayType.WEEKDAY),TUESDAY(PayType.WEEKDAY),
    WEDNESDAY(PayType.WEEKDAY), THURSDAY(PayType.WEEKDAY),
    FRIDAY(PayType.WEEKDAY),
    SATURDAY(PayType.WEEKEND),SUNDAY(PayType.WEEKEND);
    private final PayType payType;
    StrategyEnum(PayType payType) {
        this.payType = payType;
    }
    double pay(double hoursWorked, double payRate) {
        return payType.pay(hoursWorked, payRate);
    }
    //策略枚举类型
    private enum PayType{
        WEEKDAY{
            double overtimePay(double hours, double payRate) {
                return hours <= HOURS_PER_SHIFT ? 0 :
                        (hours - HOURS_PER_SHIFT) * payRate / 2;
            }
        },
        WEEKEND{
            double overtimePay(double hours, double payRate) {
                return hours * payRate / 2;
            }
        };
        private static final int HOURS_PER_SHIFT = 8;
        abstract double overtimePay(double hrs, double payRate);
        double pay(double hoursWorked, double payRate) {
            double basePay = hoursWorked * payRate;
            return basePay + overtimePay(hoursWorked, payRate);
        }
    }
}
```