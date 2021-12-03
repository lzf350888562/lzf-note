# fastjson

自定义转换,控制字段的排序，日期显示格式，序列化标记等

```
//默认情况FastJson库可以序列化 Java bean 实体,但我们可以使用serialize指定字段不序列化。
@JSONField(name="AGE", serialize=false)
private int age;
//使用ordinal参数指定字段的顺序
@JSONField(name="LAST NAME", ordinal = 2)
private String lastName;
@JSONField(name="FIRST NAME", ordinal = 1)
private String firstName;
 //format 参数用于格式化 date 属性。
@JSONField(name="DATE OF BIRTH", format="dd/MM/yyyy", ordinal = 3)
private Date dateOfBirth;
//不进行序列化
@JSONField(serialize = false)
```

BeanToArray序列化

```
String jsonOutput= JSON.toJSONString(bean, SerializerFeature.BeanToArray);
```

创建 JSON 对象非常简单，只需使用 JSONObject（fastJson提供的json对象） 和 JSONArray（fastJson提供json数组对象） 对象即可。

来回转换

```
String jsonObject = JSON.toJSONString(person);
Person newPerson = JSON.parseObject(jsonObject, Person.class);
```

注意

FastJson 在进行操作时，是根据 getter 和 setter 的方法进行的，并不是依据 Field 进行。

若属性是私有的，必须有 set 方法。否则无法反序列化。

注意反序列化时为对象时，必须要有默认无参的构造函数，否则会报异常:

FastJson默认是会将没赋值的属性不进行序列化

















# jackson

来回转换

```
ObjectMapper mapper = new ObjectMapper();
String json = mapper.writeValueAsString(user); 
User user = mapper.readValue(json, User.class);
```

## ObjectMapper

```
private static final ObjectMapper getMapper() {
		ObjectMapper mapper = new ObjectMapper();
		// 设置自定义的 SimpleDateFormat，该对象支持"yyyy-MM-dd HH:mm:ss"格式 
		mapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
		return mapper;
	}
```



## 字段注解@JsonProperty等

```
//序列化email属性为mail  
@JsonProperty("mail")  
private String email;
//格式    还可以用在LocalDateTime上
@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
private Date createTime;
//为空不序列化
@JsonInclude(JsonInclude.Include.NON_EMPTY)
private List<TreeSelect> children;
//不JSON序列化年龄属性  
@JsonIgnore   
private Integer age;
```

JsonInclude.Include.ALWAYS 这个是默认策略，任何情况下都序列化该字段.
JsonInclude.Include.NON_NULL如果字段为null,那么就不序列化这个字段。
JsonInclude.Include.NON_ABSENT  ?
JsonInclude.Include.NON_EMPTY 这个属性包含NON_NULL，NON_ABSENT之后还包含如果字段为空也不序列化。这个也比较常用
JsonInclude.Include.NON_DEFAULT  如果字段是默认值的话就不序列化。
JsonInclude.Include.CUSTOM ?

**类上注解**

### @JsonIgnoreProperties

如果需要转换的json字符串里的字段多余要转换的对象的字段

```
@JsonIgnoreProperties(ignoreUnknown = true) 在目标对象的类级别上加上该注解，并配置ignoreUnknown = true，则Jackson在反序列化的时候，会忽略该目标对象不存在的属性
```

```
@JsonIgnoreProperties(value = {"theme", "port"}, allowSetters = true)
```

也可以全局配置

```
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES,false);配置该objectMapper在反序列化时，忽略目标对象没有的属性。凡是使用该objectMapper反序列化时，都会拥有该特性。
```



### 指定序列化器

```
public class UserSerializer extends JsonSerializer<User> {
	@Override
	public void serialize(User user, JsonGenerator generator, SerializerProvider provider)
			throws IOException, JsonProcessingException {
		generator.writeStartObject();
		generator.writeStringField("user-name", user.getUserName());
		generator.writeEndObject();
	}
}

public class UserDeserializer extends JsonDeserializer<User> {
	@Override
	public User deserialize(JsonParser parser, DeserializationContext context)
			throws IOException, JsonProcessingException {
		JsonNode node = parser.getCodec().readTree(parser);
		String userName = node.get("user-name").asText();
		User user = new User();
		user.setUserName(userName);
		return user;
	}
}
```

注解

```
@JsonSerialize(using = UserSerializer.class)
@JsonDeserialize (using = UserDeserializer.class)
```

也可以单独对某个属性指定序列化器

### @JsonName

指定json key命名规则

```
@JsonNaming(PropertyNamingStrategy.LowerCaseWithUnderscoresStrategy.class)
```

属性小写单词以下划线分隔

### @JsonUnwrapperd

扁平化转换

```
@Data
public class Account {
	@JsonUnwrapped
    private Location location;
  @Data
  public static class Location {
     private String provinceName;
     private String countyName;
  }
}
```

```
//未使用注解前
{
    "location": {
        "provinceName":"湖北",
        "countyName":"武汉"
    },
}
//使用注解后
{
  "provinceName":"湖北",
  "countyName":"武汉",
}
```



## **Date类型的属性转换**

除了单独加在属性上面

```
#默认情况下json实际格式带有时区并且时世界标准实际,和我们的时间查了8个消失.  设置返回json全局时间格式
spring.jackson.date-format=yyyy-MM-dd HH:mm:ss
spring.jackson.time-zone=GMT+8
```

或者通过配置文件的形式

```
@Configuration
public class JacksonConfig {
	@Bean
	public ObjectMapper getObjectMapper(){
		ObjectMapper mapper = new ObjectMapper();
		mapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
		return mapper;
	}
}
```

## @JsonView

以注解指定规则接口的形式指定哪些字段进行json转换

```
@JsonNaming(PropertyNamingStrategy.LowerCaseWithUnderscoresStrategy.class)
public class User implements Serializable {
	private static final long serialVersionUID = 6222176558369919436L;
	public interface UserNameView {
	};
	//表示AllUserFieldView包括UserNameView
	public interface AllUserFieldView extends UserNameView {
	};
	@JsonView(UserNameView.class)
	private String userName;
	@JsonView(AllUserFieldView.class)
	private int age;
	@JsonView(AllUserFieldView.class)
	private String password;
	@JsonView(AllUserFieldView.class)
	private Date birthday;
}
```

在controller方法上添加注解指定输出规则

```
	@JsonView(User.AllUserFieldView.class)  
	@RequestMapping("getuser")
	@ResponseBody
	public User getUser() {
		User user = new User();
		user.setUserName("mrbird");
		user.setAge(26);
		user.setPassword("123456");
		user.setBirthday(new Date());
		return user;
	}
	
	@JsonView(User.UserNameView.class)  
	@RequestMapping("getuser")
	@ResponseBody
	public User getUser() {
		User user = new User();
		user.setUserName("mrbird");
		//下面的不输出
		user.setAge(26);
		user.setPassword("123456");
		user.setBirthday(new Date());
		return user;
	}
```

## 自定义读取json

读取某个属性

```
String json = "{\"name\":\"mrbird\",\"age\":26}";
JsonNode node = this.mapper.readTree(json);
String name = node.get("name").asText();
int age = node.get("age").asInt();
```

读取数组

```
String jsonStr = "[{\"userName\":\"mrbird\",\"age\":26},{\"userName\":\"scott\",\"age\":27}]";
JavaType type = mapper.getTypeFactory().constructParametricType(List.class, User.class);
List<User> list = mapper.readValue(jsonStr, type);
```

## 配置文件

```
#将所有数字转为string类型返回,避免前端精度
spring.jackson.generator.write-numbers-as-strings=true

spring:
  t
  jackson:
    time-zone: GMT+8
    date-format: yyyy-MM-dd HH:mm:ss

```

# Gson
