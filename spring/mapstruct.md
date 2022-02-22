> <br>
>
> ## **Overview**
>
> - ### [**개요**](#개요)
> - ### [**mapstruct**](#mapstruct)
>   - [**사용하기**](#사용하기)
>     - ##### [**일반적인 객체 매핑**](#일반적인-객체-매핑)
>     - ##### [**객체 속성 무시하기**](#객체-속성-무시하기)
>     - ##### [**다른 이름으로 매핑**](#다른-이름으로-매핑)
>     - ##### [**객체에서 속성 꺼내서 매핑**](#객체에서-속성-꺼내서-매핑)
>     - ##### [**객체 병합하기 1**](#객체-병합하기-1)
>     - ##### [**객체 병합하기 2**](#객체-병합하기-2)
> - [**리스트 매핑하기**](#리스트-매핑하기)
>   - [**qualifiedByName**](#qualifiedByName)
>   - [**default method**](#default-method) <br><br>

<br />

#### [**예제 및 테스트 코드는 github 에서 확인 가능합니다.**](https://github.com/limwoobin/blog-code-example/tree/master/mapstruct-example)

<br />

# **개요**

소스코드를 작성하다보면 Layer를 전환하며 객체를 전환하며 매핑하거나 여러 객체를 합치거나 하는 다양한 경우를 만나게 됩니다.  
흔히 겪는 예시로는 presentation layer 에서는 DTO , service layer , repository layer 에서는 Entity 를 사용하는 예시를 들 수 있습니다.  
이를 매핑하기 위해서는 **model mapper , 정적 팩토리 , object mapping** 등의 방법을 다양한 이용해 모델을 매핑하고 있습니다.  
저는 제가 사용하는 **mapstruct** 에 대해 간략하게 소개하려고 합니다.

<br>

# **mapstruct**

[**mapstruct github page**](#https://github.com/mapstruct/mapstruct#what-is-mapstruct)에서는 mapstrut를 다음과 같이 소개하고 있습니다.

> <br>
> 간략하게 요약하면 다음과 같습니다.  <br>
> 타입에 안전한 Bean Mapping 클래스를 생성하기 위한 Java Annotation Processing
>
> 런타임에 작동하는 Mapping framework 와 비교했을때 MapStruct 는 다음과 같은 이점을 갖는다
>
> - reflection 대신 일반 method 를 사용하기 때문에 빠름
> - 컴파일시 오류를 확인할 수 있음
> - Implementation code 를 제공해 쉽게 디버깅이 가능 , 직접 확인 가능
>
>   <br>

<br />

## **사용하기**

mapstruct 의 사용 예시를 간략하게 소개하겠습니다.  
<br>

User.java

```java
@Entity
@Getter
@Setter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private int age;
}
```

UserDTO

```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
public class UserDTO {
    private Long id;

    private String name;

    private int age;

    private String address;
}
```

User 에서 UserDTO 로 객체매핑을 하는 예시를 들어보겠습니다.

#### **일반적인 객체 매핑**

**UserMpaper.java**

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
	UserDTO toUserDTO(User user);
}
```

위의 코드를 빌드하게되면 아래와 같은 코드가 생성됩니다.

**UserMpaperImpl.java**

```java
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2022-02-20T16:39:40+0900",
    comments = "version: 1.4.2.Final, compiler: IncrementalProcessingEnvironment from gradle-language-java-7.3.3.jar, environment: Java 11.0.10 (Oracle Corporation)"
)
@Component
public class UserMapperImpl implements UserMapper {
	@Override
	public UserDTO toUserDTO(User user) {
		if ( user == null ) {
				return null;
		}

		UserDTO userDTO = new UserDTO();

		userDTO.setId( user.getId() );
		userDTO.setName( user.getName() );
		userDTO.setAge( user.getAge() );

		return userDTO;
	}
}
```

<br>

#### **객체 속성 무시하기**

**UserMpaper.java**

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
	@Mapping(target = "name" , ignore = true)
	UserDTO toUserDTO(User user);
}
```

**UserMpaperImpl.java**

```java
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2022-02-20T16:39:40+0900",
    comments = "version: 1.4.2.Final, compiler: IncrementalProcessingEnvironment from gradle-language-java-7.3.3.jar, environment: Java 11.0.10 (Oracle Corporation)"
)
@Component
public class UserMapperImpl implements UserMapper {
	@Override
	public UserDTO toUserDTO(User user) {
		if ( user == null ) {
				return null;
		}

		UserDTO userDTO = new UserDTO();

		userDTO.setId( user.getId() );
		userDTO.setAge( user.getAge() );

		return userDTO;
	}
}
```

위와 같이 name 속성이 무시된것을 확인할 수 있습니다.

<br>

#### **다른 이름으로 매핑**

**UserMpaper.java**

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
	@Mapping(target = "address" , source = "name")
	UserDTO toUserDTO(User user);
}
```

**UserMpaperImpl.java**

```java
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2022-02-20T16:39:40+0900",
    comments = "version: 1.4.2.Final, compiler: IncrementalProcessingEnvironment from gradle-language-java-7.3.3.jar, environment: Java 11.0.10 (Oracle Corporation)"
)
@Component
public class UserMapperImpl implements UserMapper {
	@Override
	public UserDTO toUserDTO(User user) {
		if ( user == null ) {
				return null;
		}

		UserDTO userDTO = new UserDTO();

		userDTO.setAddress( user.getName() );
		userDTO.setId( user.getId() );
		userDTO.setName( user.getName() );
		userDTO.setAge( user.getAge() );

		return userDTO;
	}
}
```

User 의 name 이라는 필드값이 UserDTO 에 address 라는 필드값에 매핑된것을 확인할 수 있습니다.

<br>

#### **객체에서 속성 꺼내서 매핑**

**Address.java**

```java
@Getter
@Setter
public class Address {
	private String myAddress;
}
```

**UserMpaper.java**

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
	@Mapping(target = "address" , source = "address.myAddress")
	UserDTO toUserDTO(User user , Address address);
}
```

**UserMapperImpl.java**

```java
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2022-02-20T16:39:40+0900",
    comments = "version: 1.4.2.Final, compiler: IncrementalProcessingEnvironment from gradle-language-java-7.3.3.jar, environment: Java 11.0.10 (Oracle Corporation)"
)
@Component
public class UserMapperImpl implements UserMapper {
	@Override
	public UserDTO toUserDTO(User user, Address address) {
		if ( user == null && address == null ) {
			return null;
		}

		UserDTO userDTO = new UserDTO();

		if ( user != null ) {
			userDTO.setId( user.getId() );
			userDTO.setName( user.getName() );
			userDTO.setAge( user.getAge() );
		}
		if ( address != null ) {
			userDTO.setAddress( address.getMyAddress() );
		}

		return userDTO;
	}
}
```

address 내의 myAddress 필드가 UserDTO 의 address 에 매핑된것을 확인할 수 있습니다.

<br>

#### **객체 병합하기 1**

**UserMpaper.java**

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
	UserDTO toUserDTO(User user , String address);
}
```

**UserMpaperImpl.java**

```java
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2022-02-20T16:39:40+0900",
    comments = "version: 1.4.2.Final, compiler: IncrementalProcessingEnvironment from gradle-language-java-7.3.3.jar, environment: Java 11.0.10 (Oracle Corporation)"
)
@Component
public class UserMapperImpl implements UserMapper {
	@Override
	public UserDTO toUserDTO(User user, String address) {
		if ( user == null && address == null ) {
				return null;
		}

		UserDTO userDTO = new UserDTO();

		if ( user != null ) {
				userDTO.setId( user.getId() );
				userDTO.setName( user.getName() );
				userDTO.setAge( user.getAge() );
		}
		if ( address != null ) {
				userDTO.setAddress( address );
		}

		return userDTO;
	}
}
```

<br>

#### **객체 병합하기 2**

```java
public class UserDTO {
	private Long id;

	private String name;

	private int age;

	private AddressDTO addressDTO;
}

public class AddressDTO {
	private String myAddress;
}
```

**UserMpaper.java**

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
	@Mapping(target = "addressDTO" , source = "address")
	UserDTO toUserDTO(User user , Address address);
}
```

**UserMpaperImpl.java**

```java
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2022-02-20T16:39:40+0900",
    comments = "version: 1.4.2.Final, compiler: IncrementalProcessingEnvironment from gradle-language-java-7.3.3.jar, environment: Java 11.0.10 (Oracle Corporation)"
)
@Component
public class UserMapperImpl implements UserMapper {
	@Override
	public UserDTO toUserDTO(User user, Address address) {
		if ( user == null && address == null ) {
				return null;
		}

		UserDTO userDTO = new UserDTO();

		if ( user != null ) {
				userDTO.setId( user.getId() );
				userDTO.setName( user.getName() );
				userDTO.setAge( user.getAge() );
		}
		if ( address != null ) {
				userDTO.setAddressDTO( addressToAddressDTO( address ) );
		}

		return userDTO;
	}
}
```

필드 속성뿐만 아니라 객체도 이름만 동일하다면 위와 같이 매핑되는것을 확인할 수 있습니다.

<br>

# **리스트 매핑하기**

**UserMapper.java**

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
	UserMapper INSTANCE = Mappers.getMapper(UserMapper.class);

	UserDTO toUserDTO(User user);

	@Mapping(target = "name" , ignore = true)
	UserDTO toUserDTO_v2(User user);

	List<UserDTO> toDTOList(List<User> users);
}
```

다음과 같이 User에서 UserDTO 로 전환하는 mapper method가 두개가 있습니다. 그리고 List<User> 에서 List<UserDTO> 로 전환하는 메소드도 존재합니다.  
하지만 위 코드를 build하게 되면 다음과 같은 에러를 뱉습니다.

![mapsturct-image-1](https://user-images.githubusercontent.com/28802545/154836762-5a548f12-7a5f-43a7-9682-65c9cedc85dc.PNG)

이유는 mapstruct 에서 List 내의 요소들을 매핑할때 **toUserDTO** 를 사용할지 **toUserDTO_v2** 를 사용할지 모르기 때문입니다.  
만약 **toUserDTO** 하나만 있거나 아에 없었다면 정상적으로 컴파일 되었을겁니다.

<br>

### **qualifiedByName**

mapstruct 에서는 다음과 같은 경우를 해결하기 위해 qualifiedByName 라는 기능을 지원합니다.  
리스트가 순회시에 어떠한 mapper 를 사용할지 선택할 수 있습니다.

**UserMapper.java**

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
	UserMapper INSTANCE = Mappers.getMapper(UserMapper.class);

	UserDTO toUserDTO(User user);

	@Mapping(target = "name" , ignore = true)
	@Named("v2")
	UserDTO toUserDTO_v2(User user);

	@IterableMapping(qualifiedByName = "v2")
	List<UserDTO> toDTOList(List<User> users);
}
```

**toUserDTO_v2** 라는 메소드에 **v2** 라는 이름을 주었습니다. **toDTOList** 에서도 **v2** 이름의 메소드를 사용하게끔 명시했습니다.

**UserMapperImpl.java**

```java
@Generated(
    value = "org.mapstruct.ap.MappingProcessor",
    date = "2022-02-20T18:45:45+0900",
    comments = "version: 1.4.2.Final, compiler: IncrementalProcessingEnvironment from gradle-language-java-7.3.3.jar, environment: Java 11.0.10 (Oracle Corporation)"
)
@Component
public class UserMapperImpl implements UserMapper {
	@Override
	public UserDTO toUserDTO(User user) {
		if ( user == null ) {
				return null;
		}

		UserDTOBuilder userDTO = UserDTO.builder();

		userDTO.id( user.getId() );
		userDTO.name( user.getName() );
		userDTO.age( user.getAge() );

		return userDTO.build();
	}

	@Override
	public UserDTO toUserDTO_v2(User user) {
		if ( user == null ) {
				return null;
		}

		UserDTOBuilder userDTO = UserDTO.builder();

		userDTO.id( user.getId() );
		userDTO.age( user.getAge() );

		return userDTO.build();
	}

	@Override
	public List<UserDTO> toDTOList(List<User> users) {
		if ( users == null ) {
				return null;
		}

		List<UserDTO> list = new ArrayList<UserDTO>( users.size() );
		for ( User user : users ) {
				list.add( toUserDTO_v2( user ) );
		}

		return list;
	}
}
```

빌드가 정상적으로 실행되어 코드가 생성된걸 확인할 수 있습니다.  
**toDTOList** 를 보시면 v2라는 이름으로 명시했던 toUserDTO_v2 메소드를 사용한걸 확인할 수 있습니다.

<br>

### **default method**

java8 이후부터 지원하는 interface 의 default method 를 이용하는것도 하나의 방법이 될 수 있습니다.

**UserMapper.java**

```java
@Mapper(componentModel = "spring")
public interface UserMapper {
	UserMapper INSTANCE = Mappers.getMapper(UserMapper.class);

	UserDTO toUserDTO(User user);

	@Mapping(target = "name" , ignore = true)
	@Named("v2")
	UserDTO toUserDTO_v2(User user);

	default List<UserDTO> toDTOList(List<User> users) {
		return users.stream()
			.map(this::toUserDTO_v2)
			.collect(Collectors.toList());
}
```

테스트 코드를 이용하여 확인해보겠습니다.

```java
public class MapStructTest {
	private final UserMapper userMapper = Mappers.getMapper(UserMapper.class);

	@DisplayName("List<Entity> to List<DTO> 로 전환하면 DTO 의 name 이 제거되어야 한다")
	@Test
	void mapper_test_6() {
		// given
		User 테스트_유저 = new User(1L , "테스트" , 15);
		User 테스트_유저2 = new User(2L , "테스트2" , 22);

		// when
		List<User> users = List.of(테스트_유저, 테스트_유저2);
		List<UserDTO> userDTOS = userMapper.toDTOList2(users);

		// then
		assertThat(userDTOS.get(0).getName()).isNull();
		assertThat(userDTOS.get(1).getName()).isNull();
	}
}
```

![mapsturct-image-2](https://user-images.githubusercontent.com/28802545/155122896-3c950e6f-59e7-45f3-9b55-319ce9d0350b.PNG)

테스트 코드가 정상적으로 통과된것을 확인할 수 있습니다.

<br>
