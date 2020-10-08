思路点
1、StringBuilder和StringBuffer都是集成AbstractStringBuilder类
2、对于内置char数组，String不可变（final），StringBuilder、StringBuffer可变性
2、对于线程安全，String、StringBuffer是安全的，StringBuilder不安全
3、对于性能（大量数据操作），StringBuilder最优，StringBuffer其次，String最差
4、与JVM中OOP对象的关系（String改变、AbstractStringBuilder改变char[]、字符串常量池）
5、关于String、StringBuilder、StringBuffer多线程环境下性能测试的代码例子

参考
JAVA理论知识文档/JAVA基础常见知识点.md#12-string-stringbuffer-和-stringbuilder-的区别是什么-string-为什么是不可变的
@peteryuanpan peteryuanpan added JAVA memory labels 20 days ago
@peteryuanpan
 
Owner
Author
peteryuanpan commented 20 days ago
StringBuilder和StringBuffer都是集成AbstractStringBuilder类
StringBuilder部分源码

package java.lang;

public final class StringBuilder
    extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence
{

    /** use serialVersionUID for interoperability */
    static final long serialVersionUID = 4383685877147921099L;

    /**
     * Constructs a string builder with no characters in it and an
     * initial capacity of 16 characters.
     */
    public StringBuilder() {
        super(16);
    }
StringBuffer部分源码

package java.lang;

 public final class StringBuffer
    extends AbstractStringBuilder
    implements java.io.Serializable, CharSequence
{

    /**
     * A cache of the last value returned by toString. Cleared
     * whenever the StringBuffer is modified.
     */
    private transient char[] toStringCache;

    /** use serialVersionUID from JDK 1.0.2 for interoperability */
    static final long serialVersionUID = 3388685877147921107L;

    /**
     * Constructs a string buffer with no characters in it and an
     * initial capacity of 16 characters.
     */
    public StringBuffer() {
        super(16);
    }
@peteryuanpan
 
Owner
Author
peteryuanpan commented 20 days ago • 
对于内置char数组，String不可变（final），StringBuilder、StringBuffer可变性
String部分源码

package java.lang;

public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];

    /** Cache the hash code for the string */
    private int hash; // Default to 0
StringBuilder和StringBuffer的char数组都在AbstractStringBuilder中

AbstractStringBuilder部分源码

package java.lang;

abstract class AbstractStringBuilder implements Appendable, CharSequence {
    /**
     * The value is used for character storage.
     */
    char[] value;

    /**
     * The count is the number of characters used.
     */
    int count;
可以看出来，String中的char value[]数组式final修饰的，不可改变，String也是final修饰的

每次对 String 类型进行改变的时候，都会生成一个新的 String 对象，然后将指针指向新的 String 对象

而AbstractStringBuilder中的char value[]数组是可变的，每次进行修改操作（append、insert等）时，StringBuilder和StringBuilder对象不变，会生成新的char[]数组

下面来看一下 AbstractStringBuilder 的 append 函数源码

    public AbstractStringBuilder append(String str) {
        if (str == null)
            return appendNull();
        int len = str.length();
        ensureCapacityInternal(count + len);
        str.getChars(0, len, value, count);
        count += len;
        return this;
    }
其中有两个函数，一个是 ensureCapacityInternal，一个是 str.getChars

关于 ensureCapacityInternal 源码

    private void ensureCapacityInternal(int minimumCapacity) {
        // overflow-conscious code
        if (minimumCapacity - value.length > 0) {
            value = Arrays.copyOf(value,
                    newCapacity(minimumCapacity));
        }
    }

    private int newCapacity(int minCapacity) {
        // overflow-conscious code
        int newCapacity = (value.length << 1) + 2;
        if (newCapacity - minCapacity < 0) {
            newCapacity = minCapacity;
        }
        return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
            ? hugeCapacity(minCapacity)
            : newCapacity;
    }

    private int hugeCapacity(int minCapacity) {
        if (Integer.MAX_VALUE - minCapacity < 0) { // overflow
            throw new OutOfMemoryError();
        }
        return (minCapacity > MAX_ARRAY_SIZE)
            ? minCapacity : MAX_ARRAY_SIZE;
    }
可以看出来，ensureCapacityInternal是为了保证char[] value数组大小合适的，因为 str.getChars 即将对它进行改变

再来看一下 Arrays.copyOf 源码

    public static char[] copyOf(char[] original, int newLength) {
        char[] copy = new char[newLength];
        System.arraycopy(original, 0, copy, 0,
                         Math.min(original.length, newLength));
        return copy;
    }
可以看出来，这里有个 char[] copy = new char[newLength] 操作，并 return copy，这样当ensureCapacityInternal起作用（minimumCapacity - value.length > 0）时，会生成一个新的char数组赋值给char[] value

关于 str.getChars 源码

    public void getChars(int srcBegin, int srcEnd, char dst[], int dstBegin) {
        if (srcBegin < 0) {
            throw new StringIndexOutOfBoundsException(srcBegin);
        }
        if (srcEnd > value.length) {
            throw new StringIndexOutOfBoundsException(srcEnd);
        }
        if (srcBegin > srcEnd) {
            throw new StringIndexOutOfBoundsException(srcEnd - srcBegin);
        }
        System.arraycopy(value, srcBegin, dst, dstBegin, srcEnd - srcBegin);
    }
关于 System.arraycopy 源码

    public static native void arraycopy(Object src,  int  srcPos,
                                        Object dest, int destPos,
                                        int length);
可以看出来，str.getChar目的是，给char[] dst从dstBegin位置开始往后赋值，长度为strEnd - strBegin，原始数组是str中的 char[] value

System.arraycopy是native修饰的，要走到JVM源码中去查询了

总之，通过append函数可以看出来，StringBuilder及StringBuffer在进行字符串修改操作时，会修改char[] value的值，若数组长度需要改变，还会赋值新的char[]数组对象

@peteryuanpan
 
Owner
Author
peteryuanpan commented 20 days ago • 
对于线程安全，String、StringBuffer是安全的，StringBuilder不安全
1、String 中的对象是不可变的，也就可以理解为常量，线程安全。
2、AbstractStringBuilder 是 StringBuilder 与 StringBuffer 的公共父类，定义了一些字符串的基本操作，如 expandCapacity、append、insert、indexOf 等公共方法。StringBuffer 对方法加了同步锁或者对调用的方法加了同步锁，所以是线程安全的。StringBuilder 并没有对方法进行加同步锁，所以是非线程安全的。

StringBuffer部分源码

    @Override
    public synchronized StringBuffer append(Object obj) {
        toStringCache = null;
        super.append(String.valueOf(obj));
        return this;
    }

    @Override
    public synchronized StringBuffer insert(int index, char[] str, int offset,
                                            int len)
    {
        toStringCache = null;
        super.insert(index, str, offset, len);
        return this;
    }

    @Override
    public synchronized StringBuffer reverse() {
        toStringCache = null;
        super.reverse();
        return this;
    }

    /**
     * @since      1.4
     */
    @Override
    public synchronized int indexOf(String str, int fromIndex) {
        return super.indexOf(str, fromIndex);
    }
StringBuilder部分源码

    @Override
    public StringBuilder append(String str) {
        super.append(str);
        return this;
    }

    @Override
    public StringBuilder delete(int start, int end) {
        super.delete(start, end);
        return this;
    }

    @Override
    public StringBuilder replace(int start, int end, String str) {
        super.replace(start, end, str);
        return this;
    }

    @Override
    public StringBuilder insert(int index, char[] str, int offset,
                                int len)
    {
        super.insert(index, str, offset, len);
        return this;
    }
@peteryuanpan
 
Owner
Author
peteryuanpan commented 20 days ago • 
对于性能（大量数据操作），StringBuilder最优，StringBuffer其次，String最差
1、操作少量的数据，适用String
2、单线程操作字符串缓冲区下操作大量数据，适用StringBuilder
3、多线程操作字符串缓冲区下操作大量数据，适用StringBuffer

这里在习惯上要注意，以后处理字符串问题时，先考虑用String、StringBuilder，还是StringBuffer。

@peteryuanpan
 
Owner
Author
peteryuanpan commented 20 days ago • 
与JVM中OOP对象的关系（String改变、AbstractStringBuilder改变char[]、字符串常量池）
OOP是JAVA对象在JVM中的存在形式

例子1：String a = "1"; a = "2";
例子2：String a = new String("1"); a = new String("2");
例子3：StringBuilder a = new StringBuilder("1"); a.append("2");
例子4：StringBuffer a = new StringBuffer("1"); a.append("2");

JVM底层OOP对象如何改变？
有点难，先放着