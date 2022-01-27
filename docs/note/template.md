# 基础

## 枚举定义模板

```
public enum ExampleEnum {
    /** 枚举相关 */
    ONE(1, "one(1)"),
    TWO(2, "two(2)"),
    THREE(3, "two(3)");

    /** 字段相关 优先使用基础类型 final防止*/
    private final int value;
    private final String desc;

    /** 构造方法 private省略*/
    ExampleEnum(int value, String desc) {
        this.value = value;
        this.desc = desc;
    }

    /** 获取取值 */
    public int getValue() {
        return value;
    }

```

## 集合(数组)常量

普通集合常量即便定义为final也可通过集合方法进行修改, 包括Arrays.asList方法生成的内部ArrayList不能执行add/remove/clear方法,但是可以set方法,也属于可变集合对象

```
 public final class ExampleHelper {
    public static final List<Integer> CONST_VALUE_LIST = Arrays.asList(1, 2, 3);
    public static final Set<Integer> CONST_VALUE_SET = new HashSet<>(Arrays.asList(1, 2, 3));
    public static final Map<Integer, String> CONST_VALUE_MAP;
    static {
        CONST_VALUE_MAP = new HashMap<>(MapHelper.DEFAULT);
        CONST_VALUE_MAP.put(1, "value1");
        CONST_VALUE_MAP.put(2, "value2");
        CONST_VALUE_MAP.put(3, "value3");
    }
}
```

可通过jdk的Collections工具类中提供的一套方法,用于把可变集合变为不可变,修改时UnsupportedOperationException异常

```java
public final class ExampleHelper {
    public static final List<Integer> CONST_VALUE_LIST = Collections.unmodifiableList(Arrays.asList(1, 2, 3));
    public static final Set<Integer> CONST_VALUE_SET = Collections.unmodifiableSet(new HashSet<>(Arrays.asList(1, 2, 3)));
    public static final Map<Integer, String> CONST_VALUE_MAP;
    static {
        Map<Integer, String> valueMap = new HashMap<>(MapHelper.DEFAULT);
        valueMap.put(1, "value1");
        valueMap.put(2, "value2");
        valueMap.put(3, "value3");
        CONST_VALUE_MAP = Collections.unmodifiableMap(valueMap);
    }
}
```

普通的final数组常量也可以通过下标值修改数组值, 可通过Collections包装list来解决

```java
public static final int[] CONST_VALUES = Collections.unmodifiableList(Arrays.asList(1,2,3)).stream().mapToInt(Integer::intValue).toArray();  
```

但是上述方式存在缺点:每一次都会把集合常量转换为数组常量,程序运行效率降低;

最佳方式:”私有数组常量+公有克隆方法”解决方案：先定义一个私有数组常量，保证不会被外部类使用；在定义一个获取数组常量方法，并返回一个数组常量的克隆值。

```
public final class ExampleHelper {
    /** 常量值数组 */
    private static final int[] CONST_VALUES = new int[] {1, 2, 3};
    /** 获取常量值数组方法 */
    public static int[] getConstValues() {
        return CONST_VALUES.clone();
    }
    ...
}
```

由于每次返回的是一个克隆数组，即便修改了克隆数组的常量值，也不会导致原始数组常量值的修改。

## 原生序列化

java序列化二进制流的方式

```
public class SerializeUtils {
    private static Logger logger = LoggerFactory.getLogger(SerializeUtils.class);
    public static Object deserialize(byte[] bytes) {
        Object result = null;
        if (isEmpty(bytes)) {
            return null;
        }
        try {
            ByteArrayInputStream byteStream = new ByteArrayInputStream(bytes);
            try {
                ObjectInputStream objectInputStream = new ObjectInputStream(byteStream);
                try {
                    result = objectInputStream.readObject();
                }
                catch (ClassNotFoundException ex) {
                    throw new Exception("Failed to deserialize object type", ex);
                }
            }
            catch (Throwable ex) {
                throw new Exception("Failed to deserialize", ex);
            }
        } catch (Exception e) {
            logger.error("Failed to deserialize",e);
        }
        return result;
    }

    public static boolean isEmpty(byte[] data) {
        return (data == null || data.length == 0);
    }

    public static byte[] serialize(Object object) {

        byte[] result = null;

        if (object == null) {
            return new byte[0];
        }
        try {
            ByteArrayOutputStream byteStream = new ByteArrayOutputStream(128);
            try  {
                if (!(object instanceof Serializable)) {
                    throw new IllegalArgumentException(SerializeUtils.class.getSimpleName() + " requires a Serializable payload " +
                            "but received an object of type [" + object.getClass().getName() + "]");
                }
                ObjectOutputStream objectOutputStream = new ObjectOutputStream(byteStream);
                objectOutputStream.writeObject(object);
                objectOutputStream.flush();
                result =  byteStream.toByteArray();
            }
            catch (Throwable ex) {
                throw new Exception("Failed to serialize", ex);
            }
        } catch (Exception ex) {
            logger.error("Failed to serialize",ex);
        }
        return result;
    }
}
```







# 日期时间

## 时间格式转换

```
public static String convert(String inDate){
		String formatPattern = "yyyy-MM-dd HH:mm:ss";
		ZonedDateTime zdt  = ZonedDateTime.parse(inDate);
		LocalDateTime localDateTime = zdt.toLocalDateTime();
		DateTimeFormatter formatter = DateTimeFormatter.ofPattern(formatPattern);
		String outDate = formatter.format(localDateTime.plusHours(8));
		return outDate;
	}
```

## 获取最近几天日期

```
public void get(int days){
	for(int i= days-1; i>=0; i--){
		Calendar calendar = Calendar.getInstance();
		calendar.set(Calendar.DAY_OF_YEAR,Calendar.get(Calendar.DAY_OF_YEAR)-i);
		Dcalendar.getTime();
	}
}
```



# 线程

## 查看程序进程信息

```
public static void main(String[] args) {
		// 获取 Java 线程管理 MXBean
	ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
		// 不需要获取同步的 monitor 和 synchronizer 信息，仅获取线程和线程堆栈信息
		ThreadInfo[] threadInfos = threadMXBean.dumpAllThreads(false, false);
		// 遍历线程信息，仅打印线程 ID 和线程名称信息
		for (ThreadInfo threadInfo : threadInfos) {
			System.out.println("[" + threadInfo.getThreadId() + "] " + threadInfo.getThreadName());
		}
	}
```



## 根据线程ID获取线程

```
public static Thread findThread(long threadId) {
    ThreadGroup group = Thread.currentThread().getThreadGroup();
    while(group != null) {
        Thread[] threads = new Thread[(int)(group.activeCount() * 1.2)];
        int count = group.enumerate(threads, true);
        for(int i = 0; i < count; i++) {
            if(threadId == threads[i].getId()) {
                return threads[i];
            }
        }
        group = group.getParent();
    }
    return null;
}
```





