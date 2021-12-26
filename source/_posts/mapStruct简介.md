

### 1、mapStruct简介

- 官网：[MapStruct – Java bean mappings, the easy way!](https://mapstruct.org/)
- 中文文档：[MapStruct 1.3.0.Final参考指南 (kailing.pub)](http://www.kailing.pub/MapStruct1.3/index.html)
- 使用场景：

![image-20210620163904556](https://s2.loli.net/2021/12/26/RIwpNAk3YWB7zK9.png)

### 2、BeansUtils和MapStruct的对比

#### 2.1 相同之处

- 拷贝简单对象的时候都是深拷贝
- 拷贝复杂对象的时候都是浅拷贝



>复杂对象：类中属性包括对象属性的类为复杂对象

> 深拷贝：是增加了一个指针并且申请了一个新的内存，使这个增加的指针指向这个新的内存
>
> 浅拷贝：只是增加了一个指针指向已存在的内存地址



#### 2.2 不同之处

- BeanUtils在java虚拟机运行期间通过反射实现，效率低下
- BeanUtils的属性拷贝在判断空值和不同类型的属性时有很多障碍 产生Bug
- MapStruct基于JSR269 API实现，在代码编译期间通过getter/setter来实现 对代码运行效率没有性能上的损耗 
- MapStruct有非常灵活的策略和转化方式，自定义性比较强。
- MapStruct支持不同的source和target基本类型自动转换，包括在超类型上声明的属性
- 支持lombok中的model的builder链式调用 应用到ConverterImpl中

>JSR269：插入式注解处理API 是用于处理注解（元数据，[JSR175](https://www.jcp.org/en/jsr/detail?id=269)）的一套API。其API位于`javax.annotation.processing`和`javax.lang.model`包下。
>
>插入式注解处理API可以让你在编译期访问注解元数据，处理和自定义你的编译输出，像反射一样访问类、字段、方法和注解等元素，创建新的源文件等等。可用于减少编写配置文件的劳动量，提高代码可读性等等。



### 3. MapStruct的简单使用

#### 3.1 依赖

```xml
...
<properties>
    <org.mapstruct.version>1.4.2.Final</org.mapstruct.version>
</properties>
...
<dependencies>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${org.mapstruct.version}</version>
    </dependency>
</dependencies>
...
<build>
    <pluginManagement>
      <plugins>
        <plugin>
          <artifactId>maven-compiler-plugin</artifactId>
          <version>3.8.1</version>
          <configuration>
            <source>1.8</source> <!-- depending on your project -->
            <target>1.8</target> <!-- depending on your project -->
            <annotationProcessorPaths>
              <path>
                <groupId>org.mapstruct</groupId>
                <artifactId>mapstruct-processor</artifactId>
                <version>1.4.2.Final</version>
              </path>
              <path>
                  <groupId>org.projectlombok</groupId>
                  <artifactId>lombok</artifactId>
                  <version>1.18.12</version>
              </path>
              <!-- other annotation processors -->
            </annotationProcessorPaths>
          </configuration>
        </plugin>
      </plugins>
    </pluginManagement>
  </build>    
```



### 4. MapStruct自定义配置

#### 4.1 定义映射

##### 4.1.1 基本映射

属性名不同时 指定source和target进行映射

```java
@Mapper
public interface CarMapper {

    @Mapping(source = "make", target = "manufacturer")
    @Mapping(source = "numberOfSeats", target = "seatCount")
    CarDto carToCarDto(Car car);

    @Mapping(source = "name", target = "fullName")
    PersonDto personToPersonDto(Person person);
}
```

##### 4.1.2 添加自定义方法

需要手动实现的特定映射 由MapStruct自动调用

```java
@Mapper
public interface CarMapper {

    @Mapping(...)
    ...
    CarDto carToCarDto(Car car);

    default PersonDto personToPersonDto(Person person) {
        //hand-written mapping logic
    }
}
```

##### 4.1.3 多个数据源的映射

将多个实体组合成一个数据传输对象

```java
@Mapper
public interface AddressMapper {

    @Mapping(source = "person.description", target = "description")
    @Mapping(source = "address.houseNo", target = "houseNumber")
    DeliveryAddressDto personAndAddressToDeliveryAddressDto(Person person, Address address);
}
```

##### 4.1.4 更新现有的实体

使用@MappingTarget

```java
@Mapper
public interface CarMapper {

    void updateCarFromDto(CarDto carDto, @MappingTarget Car car);
}
```

##### 4.1.5 使用构建器

可结合lombok的@Builder构建

```java
public class PersonMapperImpl implements PersonMapper {

    public Person map(PersonDto dto) {
        if (dto == null) {
            return null;
        }

        Person.Builder builder = Person.builder();

        builder.name( dto.getName() );

        return builder.create();
    }
}
```



#### 4.2 检索映射器

##### 4.2.1 使用Mappers工厂

```java
CarMapper INSTANCE = Mappers.getMapper( CarMapper.class );
```

##### 4.2.2 使用依赖注入

Spring

```java
@Mapper(componentModel = "spring")
public interface CarMapper {

    CarDto carToCarDto(Car car);
}
```

```java
...
@Autowired 
private CarMapper mapper;
...
```



#### 4.3 数据类型转换

##### 4.3.1隐式类型转换

对于大多数基本类型或者包装类型之间的互转 MapStruct会自动处理类型转换

再转换时也可以通过@Mapper中的属性指定格式化

示例：

```java
@Mapper
public interface CarMapper {

    @Mapping(source = "price", numberFormat = "#.00")
    CarDto carToCarDto(Car car);

    @IterableMapping(numberFormat = "#.00")
    List<String> prices(List<Integer> prices);
}
```

##### 4.3.2映射对象引用

```java
@Mapper
public interface CarMapper {

    CarDto carToCarDto(Car car);

    PersonDto personToPersonDto(Person person);
}
```



#### 4.4 映射集合

##### 4.4.1 Collection类型

```java
@Mapper
public interface CarMapper {

    Set<String> integerSetToStringSet(Set<Integer> integers);

    List<CarDto> carsToCarDtos(List<Car> cars);

    CarDto carToCarDto(Car car);
}
```

##### 4.4.2 Map类型

```java
public interface SourceTargetMapper {

    @MapMapping(valueDateFormat = "dd.MM.yyyy")
    Map<String, String> longDateMapToStringStringMap(Map<Long, Date> source);
}
```

Impl

```java
@Override
public Map<Long, Date> stringStringMapToLongDateMap(Map<String, String> source) {
    if ( source == null ) {
        return null;
    }

    Map<Long, Date> map = new HashMap<Long, Date>();

    for ( Map.Entry<String, String> entry : source.entrySet() ) {

        Long key = Long.parseLong( entry.getKey() );
        Date value;
        try {
            value = new SimpleDateFormat( "dd.MM.yyyy" ).parse( entry.getValue() );
        }
        catch( ParseException e ) {
            throw new RuntimeException( e );
        }

        map.put( key, value );
    }

    return map;
}
```



#### 4.5 映射Stream

```java
@Mapper
public interface CarMapper {

    Set<String> integerStreamToStringSet(Stream<Integer> integers);

    List<CarDto> carsToCarDtos(Stream<Car> cars);

    CarDto carToCarDto(Car car);
}
```

Impl

```java
@Override
public Set<String> integerStreamToStringSet(Stream<Integer> integers) {
    if ( integers == null ) {
        return null;
    }

    return integers.stream().map( integer -> String.valueOf( integer ) )
        .collect( Collectors.toCollection( HashSet<String>::new ) );
}

@Override
public List<CarDto> carsToCarDtos(Stream<Car> cars) {
    if ( cars == null ) {
        return null;
    }

    return integers.stream().map( car -> carToCarDto( car ) )
        .collect( Collectors.toCollection( ArrayList<CarDto>::new ) );
}
```



#### 4.6 映射配置重用

##### 4.6.1 映射配置继承

@InheritConfiguration

```java
@Mapper
public interface CarMapper {

    @Mapping(target = "numberOfSeats", source = "seatCount")
    Car carDtoToCar(CarDto car);

    @InheritConfiguration
    void carDtoIntoCar(CarDto carDto, @MappingTarget Car car);
}
```

##### 4.6.2 逆映射

@InheritInverseConfiguration

```java
@Mapper
public interface CarMapper {

    @Mapping(source = "numberOfSeats", target = "seatCount")
    CarDto carToDto(Car car);

    @InheritInverseConfiguration
    @Mapping(target = "numberOfSeats", ignore = true)
    Car carDtoToCar(CarDto carDto);
}
```

#### 4.7Demo

Model

```java
@Data
@Builder
public class User {
    private Long id;
    private String username;
    private String password; // 密码
    private Integer sex;  // 性别
    private LocalDate birthday; // 生日
    private LocalDateTime createTime; // 创建时间
    private String config; // 其他扩展信息，以JSON格式存储
}
```

DTO

- 不显示密码；
- 将日期转换；
- config要转成对象的list；

```java
@Data
@Builder
public class UserVo {
    private Long id;
    private String username;
    private String password;
    private Integer gender;
    private LocalDate birthday;
    private String createTime;
    private List<UserConfig> config;
    @Data
    @Builder
    public static class UserConfig {
        private String field1;
        private Integer field2;
    }
}
```

Converter

```java
@Mapper
public interface UserConverter {
    UserConverter INSTANCE = Mappers.getMapper(UserConverter.class);

    /**
      * sex字段映射到gender字段
      * createTime字段格式化
      */
    @Mapping(target = "gender", source = "sex")
    @Mapping(target = "createTime", dateFormat = "yyyy-MM-dd HH:mm:ss")
    UserVo toVo(User var1);

    /**
     * gender字段映射到sex字段
     * createTime字段格式化
     */
    @Mapping(target = "sex", source = "gender")
    //@InheritInverseConfiguration
    @Mapping(target = "password", ignore = true)
    @Mapping(target = "createTime", dateFormat = "yyyy-MM-dd HH:mm:ss")
    User toModel(UserVo var1);

    /**
      * 集合映射
      */
    List<UserVo> toVoList(List<User> userList);

    /**
      * config字段自定义映射
      */
    default List<UserVo.UserConfig> strConfigToListUserConfig(String config) {
        return JSON.parseArray(config, UserVo.UserConfig.class);
    }
    
    /**
      * UserConfig对象集合自定义集合
      */
    default String listUserConfigToStrConfig(List<UserVo.UserConfig> list) {
        return JSON.toJSONString(list);
    }
    
}
```

ConverterImpl

```java
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2021-07-11T15:07:54+0800",
    comments = "version: 1.4.2.Final, compiler: javac, environment: Java 1.8.0_171 (Oracle Corporation)"
)
public class UserConverterImpl implements UserConverter {

    @Override
    public UserVo toVo(User var1) {
        if ( var1 == null ) {
            return null;
        }

        UserVoBuilder userVo = UserVo.builder();

        userVo.gender( var1.getSex() );
        if ( var1.getCreateTime() != null ) {
            userVo.createTime( DateTimeFormatter.ofPattern( "yyyy-MM-dd HH:mm:ss" ).format( var1.getCreateTime() ) );
        }
        userVo.id( var1.getId() );
        userVo.username( var1.getUsername() );
        userVo.password( var1.getPassword() );
        userVo.birthday( var1.getBirthday() );
        userVo.config( strConfigToListUserConfig( var1.getConfig() ) );

        return userVo.build();
    }

    @Override
    public User toModel(UserVo var1) {
        if ( var1 == null ) {
            return null;
        }

        UserBuilder user = User.builder();

        user.sex( var1.getGender() );
        if ( var1.getCreateTime() != null ) {
            user.createTime( LocalDateTime.parse( var1.getCreateTime(), DateTimeFormatter.ofPattern( "yyyy-MM-dd HH:mm:ss" ) ) );
        }
        user.id( var1.getId() );
        user.username( var1.getUsername() );
        user.birthday( var1.getBirthday() );
        user.config( listUserConfigToStrConfig( var1.getConfig() ) );

        return user.build();
    }

    @Override
    public List<UserVo> toVoList(List<User> userList) {
        if ( userList == null ) {
            return null;
        }

        List<UserVo> list = new ArrayList<UserVo>( userList.size() );
        for ( User user : userList ) {
            list.add( toVo( user ) );
        }

        return list;
    }
```

Test

```java
public class AppTest {
    @Test
    public void toVoTest() {
        User user = User.builder()
                .id(1L)
                .username("zhangsan")
                .sex(1)
                .password("abc123")
                .createTime(LocalDateTime.now())
                .birthday(LocalDate.of(1999, 9, 27))
                .config("[{\"field1\":\"Test Field1\",\"field2\":500},{\"field1\":\"Test Field2\",\"field2\":1000}]")
                .build();
        UserVo userVo = UserConverter.INSTANCE.toVo(user);

        System.out.println(user);
        System.out.println(userVo);

//        User(id=1, username=zhangsan, password=abc123, sex=1, birthday=1999-09-27, createTime=2021-07-11T15:09:47.485, config=[{"field1":"Test Field1","field2":500},{"field1":"Test Field2","field2":1000}])
//        UserVo(id=1, username=zhangsan, password=abc123, gender=1, birthday=1999-09-27, createTime=2021-07-11 15:09:47, config=[UserVo.UserConfig(field1=Test Field1, field2=500), UserVo.UserConfig(field1=Test Field2, field2=1000)])

    }

    @Test
    public void toModelTest() {
        ArrayList<UserVo.UserConfig> configs = new ArrayList<>();
        configs.add(UserVo.UserConfig.builder().field1("Test Field1").field2(500).build());
        configs.add(UserVo.UserConfig.builder().field1("Test Field2").field2(1000).build());

        UserVo userVo = UserVo.builder()
                .id(1L)
                .username("zhangsan")
                .gender(2)
                .password("abc123")
                .createTime("2020-01-18 15:32:54")
                .birthday(LocalDate.of(1999, 9, 27))
                .config(configs)
                .build();

        User user = UserConverter.INSTANCE.toModel(userVo);

        System.out.println(userVo);
        System.out.println(user);

//        User(id=1, username=zhangsan, password=null, sex=2, birthday=1999-09-27, createTime=2020-01-18T15:32:54, config=[{"field1":"Test Field1","field2":500},{"field1":"Test Field2","field2":1000}])
//        UserVo(id=1, username=zhangsan, password=abc123, gender=2, birthday=1999-09-27, createTime=2020-01-18 15:32:54, config=[UserVo.UserConfig(field1=Test Field1, field2=500), UserVo.UserConfig(field1=Test Field2, field2=1000)])

    }
```



### 5. 常用注解总结

#### @Mapper(componentModel = "spring")

 该接口作为映射接口，编译时MapStruct处理器的入口

```java
...
@Autowired
private IConverter converter
...
```

#### @Mappings

一组映射关系，值为一个数组，元素为@Mapping

```java
@Mappings{
    @Mapping(..)
    ..
}
```

#### @Mapping

一对映射关系 包含多个属性 支持多数据源进行映射

```java
@Mapping(source="", target="")
```

- source：源属性

- target：目标属性

- dateFormat：日期类型与String类型互转（支持Date、LocalDate、LocalDateTime）

  ```java
  @Mapping(target = "createTime", dateFormat = "yyyy-MM-dd HH:mm:ss")
  ```

- numberFormat：数值类型与String类型之间的转化

  ```java
  @Mapping(target = "price", numberFormat = "#.0")
  ```

- ignore：忽略某个属性的赋值

- defaultValue：默认值

#### @MappingTarget

用在方法参数的前面。使用此注解，源对象同时也会作为目标对象，用于更新。

#### @InheritConfiguration

继承同向映射 解决重复映射

#### @InheritInverseConfiguration

逆向继承映射 
