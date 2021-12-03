# 查看程序进程信息

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



# 根据线程ID获取线程

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



# 时间格式转换

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



# 原生序列化

```
public class SerializeUtils {

    private static Logger logger = LoggerFactory.getLogger(SerializeUtils.class);

    /**
     * 反序列化
     * @param bytes
     * @return
     */
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

    /**
     * 序列化
     * @param object
     * @return
     */
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

















