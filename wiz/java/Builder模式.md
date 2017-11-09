## 静态类
静态类什么时候被加载到内存的什么区？
静态类可以被实例化吗？
静态类可以被继承吗？
答：首先，静态类，只有内部静态类，它的加载和初始化同普通类，可以被实例化，可以继承其他的类，也可以被继承。

静态内部类和内部类异同：
1. 静态内部类可以有静态成员，内部类则不能有静态成员。
2. 静态内部类只能访问外部类的静态成员，内部类可访问所有。
3. 实例化：
静态内部类：`Inner i = new Outer.Inner();`
内部类：`Outer o = new Outer();` ， `Inner i = 0.new Inner()；`

## Builder模式
```
public class Message {
    private String author;
    private Date sendTime;
    private Message(Builder builder) {
        this.author = builder.author;
        this.sendTime = builder.sendTime;
    }
    public String getAuthor() {return author;}
    public void setAuthor(String author) {this.author = author;}
    public Date getSendTime() {return sendTime;}
    public void setSendTime(Date sendTime) {this.sendTime = sendTime;}
    public static class Builder {
        private String author;
        private Date sendTime;
        public Builder author(String author) {
            this.author = author;
            return this;
        }
        public Builder sendTime(Date sendTime) {
            this.sendTime = sendTime;
            return this;
        }
        
        public Builder fromPrototype(Message prototype) {
            author = prototype.author;
            sendTime = prototype.sendTime;
            return this;
        }
        public Message build() {return new Message(this);}
    }
    public static Builder newBuilder() {return new Builder();}
}
```
调用：`Message.newBuilder().author(c).sendTime(n).content(m).build();`