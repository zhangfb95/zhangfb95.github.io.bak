---
layout: post
title:  Hibernate validator基础篇
date:   2016-07-12 09:40:35 +0800
categories: HibernateValidator
tag: java
---

* content
{:toc}

服务端接口为保证一般功能的完整性，需要对请求参数进行校验，以便更好地保证业务的完整性。Hibernate validator作为开源实现，能很好满足我的要求，解决实际的问题。

## 开始

我们假设用户使用的是maven来作为依赖管理工具，为了使用此类库，需要引入依赖。

```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>5.3.0.Alpha1</version>
</dependency>
```

另外，类库使用统一的表达式语言（el表达式）来作为其组成部分，所以需要额外引入JSR341的实现。若使用JavaEE容器，则其自身就带有，无需引入。否则需加入以下依赖。

```xml
 <dependency>
    <groupId>org.glassfish</groupId>
    <artifactId>javax.el</artifactId>
    <version>3.0.0</version>
</dependency>
```

对于java bean，我们为了减少getter/setter/toString的代码，我们使用lombok作为编译层的转换工具。所以需要额外引入依赖。

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.16.8</version>
</dependency>
```

所有这一切做好之后，我们就可以开始写我们的第一个入门示例了。这个示例主要用于演示annotation和启动代码。

### 定义java bean

```java
package com.zhangfb95.demo.hv.helloworld;

import lombok.Data;

import javax.validation.constraints.Min;
import javax.validation.constraints.NotNull;
import javax.validation.constraints.Size;

/**
 * @author zhangfb
 */
@Data
public class Car {

    @NotNull
    private String manufacturer;

    @NotNull
    @Size(min = 2, max = 14)
    private String licensePlate;

    @Min(2)
    private int seatCount;

    public Car(String manufacturer, String licensePlate, int seatCount) {
        this.manufacturer = manufacturer;
        this.licensePlate = licensePlate;
        this.seatCount = seatCount;
    }
}
```

@NotNull说明这个字段会做非空校验，@Size说明这个字段会校验其长度（字符串长度或集合大小），@Min说明这个Number不能小于给定值。

### 单元测试

```java
package com.zhangfb95.demo.hv.helloworld;

import org.junit.BeforeClass;
import org.junit.Test;

import javax.validation.ConstraintViolation;
import javax.validation.Validation;
import javax.validation.Validator;
import javax.validation.ValidatorFactory;
import java.util.Set;

import static org.junit.Assert.assertEquals;

/**
 * @author zhangfb
 */
public class CarTest {

    private static Validator validator;

    @BeforeClass
    public static void setUp() {
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        validator = factory.getValidator();
    }

    @Test
    public void manufacturerIsNull() {
        Car car = new Car(null, "DD-AB-123", 4);

        Set<ConstraintViolation<Car>> constraintViolations = validator.validate(car);

        assertEquals(1, constraintViolations.size());
        assertEquals("may not be null", constraintViolations.iterator().next().getMessage());
    }
}
```

首先，需要生成Validator，Validator是线程安全的类，所以我们可以使用单例来保存。这个类有3个方法用于校验。本例，我们使用最基本的validate方法，用于校验一个java bean。

## bean属性的约束

bean约束在java中是使用annotation来表现的，annotation出现的地方可以有以下几个地方

+ 字段域约束
+ 方法域约束
+ 类域约束

### 字段域约束

针对类的属性，我们可以在其上加入约束annotation，这就叫字段域约束，不管该字段的范围如何，都可以通过反射取得其值。如下：

```java
public class Car {

	@NotNull
	private String manufacturer;
	// ...
}
```

### 方法域约束

为了维护字段的范围，有时我们不想直接操作字段，而是通过字段的发布（getter方法）来操作。如下：

```java
public class Car {

	private String manufacturer;

	@NotNull
	public String getManufacturer() {
		return manufacturer;
	}
}
```

### 泛型参数约束

如果java bean中的字段存在泛型，则我们可以将约束加注在泛型变量上。@Valid可以作用于集合类、自定义类，用于标明此字段需要进行校验。
但是如果该字段为null，则需要在类型参数上标注非空约束。泛型参数的约束，可以是List、Map或者任何自定义的泛型。如下：

```java
public class Car {

	@Valid
	private List<@NotNull String> parts = new ArrayList<>();

	public void addPart(String part) {
		parts.add( part );
	}

	//...
}
```

### 类级别域约束

这种校验就比上述的校验作用范围大，它对于一些强要求校验会起到非常好的效用。
比如，我们要校验一个java bean，它有一个id和时间戳，我现在需要做以下校验：id不为空时，时间戳也不能为空，通过前面的方法我们是没法做到的，但是类级别域校验却可以。
此种校验，是基于整个类。

```java
@Data
@ValidServerUpdateTime
public class Car {

	private int id;

	private Date serverUpdateTime;

	//...
}

public class ValidServerUpdateTimeValidator implements ConstraintValidator<ValidServerUpdateTime, Car> {

	@Override
	public void initialize(ValidServerUpdateTime validServerUpdateTime) {
	}

	@Override
	public boolean isValid(Car car, ConstraintValidatorContext constraintValidatorContext) {
		if ( car == null ) {
			return true;
		}

		return car.getId() == null ? true : car.getServerUpdateTime() != null;
	}
}
```

### 约束的继承性

约束默认都是有继承关系的。比如：

```java
public class Car {

	private String manufacturer;

	@NotNull
	public String getManufacturer() {
		return manufacturer;
	}

	//...
}

public class RentalCar extends Car {

	private String rentalStation;

	@NotNull
	public String getRentalStation() {
		return rentalStation;
	}

	//...
}
```

RentalCar会有两个约束，一个是manufacturer，一个是rentalStation，只有两个都不为空，才能算校验通过。

### 约束的级联性

约束不仅仅能校验当前java bean的属性，如果属性为一个集合或者自定义java bean，我们可以使用@Valid作为其传递约束的标注。
如下所示，driver不能为null，且其内的字段也会校验。

```java
public class Car {

	@NotNull
	@Valid
	private Person driver;

	//...
}
```

### Validator的主要方法

1. <T> Set<ConstraintViolation<T>> validate(T object, Class<?>... groups); // 按照分组进行校验对象
1. Set<ConstraintViolation<T>> validateProperty(T object, String propertyName, Class<?>... groups); // 按照分组对对象中的某个字段进行校验
1. <T> Set<ConstraintViolation<T>> validateValue(Class<T> beanType, String propertyName, Object value, Class<?>... groups); // 按照分组对类中属性值为指定值（value）进行预测校验

### 内置约束

#### JSR310约束

|annotation|支持的数据类型|用途|
|@AssertFalse|Boolean, boolean|校验字段为false|
|@AssertTrue|Boolean, boolean|校验字段为true|
|@DecimalMax(value=,inclusive=)|BigDecimal, BigInteger, CharSequence, byte, ...|校验指定值是否小于xx值|
|@DecimalMin(value=,inclusive=)|BigDecimal, BigInteger, CharSequence, byte, ...|校验指定值是否大于xx值|
|@Max(value=)|BigDecimal, BigInteger, byte, short, int, long and the respective wrappers of the primitive types|校验指定值是否小于xx值|
|@Min(value=)|BigDecimal, BigInteger, byte, short, int, long and the respective wrappers of the primitive types|校验指定值是否大于xx值|
|@NotNull|如何类型|字段不为空|
|@Null|如何类型|字段为空|
|@Pattern(regex=,flag=)|CharSequence|字段满足具体正则表达式|
|@Size(min=, max=)|CharSequence, Collection, Map and arrays	|字段满足指定长度|
|@Valid|非包装类型|当前字段传递校验|

#### hibernate validator约束

|annotation|支持的数据类型|用途|
|@NotBlank|CharSequence|校验字段不为空|
|@NotEmpty|CharSequence, Collection, Map and arrays|校验不能为空|
|@Length(min=,max=)|CharSequence|校验长度在指定范围|

### ConstraintViolation的主要方法

|方法名|说明|举例|
|-----|-----|-----|
|getMessage()|错误信息返回|	"may not be null"|
|getMessageTemplate()|错误信息返回的模板|"{…​ NotNull.message}"|
|getRootBean()|校验的根bean|car|
|getConstraintDescriptor()|annotation|descriptor for @NotNull|
|getInvalidValue()|被校验的值|null|

## bean方法的约束

这种约束，实际场景中用得相对较少。但是作为三大约束之一，我们仍然有必要做一定的介绍。一种用得很少的技术，在某些时候可能会发挥其超乎想象的作用。
BeanValidation从1.1开始，就支持这种规范了。方法级约束主要分两种：

1. 方法的入参，调用前校验（普通方法或构造方法）；
1. 方法的出参，调用后校验（普通方法或构造方法）；

这种校验和普通的类方法校验不一样，需要使用代理模式来实现，庆幸的是hibernate已经为我们封装好，我们只需要使用【ExecutableValidator】就可以了。

### 方法入参约束

```java
public class RentalStation {

	public RentalStation(@NotNull String name) {
		//...
	}

	public void rentCar(
			@NotNull Customer customer,
			@NotNull @Future Date startDate,
			@Min(1) int durationInDays) {
		//...
	}
}
```

上述代码表明：1、构造方法参数name不能为空；rentCar方法的入参customer和startDate不能为空，startDate必须是将来的时间，durationInDays不能比1小。

可是，这段代码对于多个入参之间有相互约束是无能为力的，我们可以使用hibernate validator为我们提供的扩展特性来办到。在整个方法上面加入约束annotation，代替加入到参数上面。
如下：

```java
public class Car {

	@LuggageCountMatchesPassengerCount(piecesOfLuggagePerPassenger = 2)
	public void load(List<Person> passengers, List<PieceOfLuggage> luggage) {
		//...
	}
}
public class Garage {

	@ELAssert(expression = "...", validationAppliesTo = ConstraintTarget.PARAMETERS)
	public Car buildCar(List<Part> parts) {
		//...
	}

	@ELAssert(expression = "...", validationAppliesTo = ConstraintTarget.RETURN_VALUE)
	public Car paintCar(int color) {
		//...
	}
}
```

### 方法返回值约束

类似于字段约束，只需要在方法上面加入即可，同时可以使用@Valid来实现约束的级联性。

```java
public class RentalStation {

	@ValidRentalStation
	public RentalStation() {
		//...
	}

    @Valid
	@NotNull
	@Size(min = 1)
	public List<Customer> getCustomers() {
		//...
	}
}
```

### 非继承性

方法级约束相对于字段级约束来说，少了继承性。我们也很容易想到，子类对父类的方法有继承，如果方法名和入参一样，则会覆盖父类方法。

### 如何实现

我们对于方法级约束已经知道如何写了，但是它是怎样应用到实际中的呢。我们以示例的方式来说明。

```java
ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
executableValidator = factory.getValidator().forExecutables();
```

#### ExecutableValidator#validateParameters()

```java
Car object = new Car( "Morris" );
Method method = Car.class.getMethod( "drive", int.class );
Object[] parameterValues = { 80 };
Set<ConstraintViolation<Car>> violations = executableValidator.validateParameters(
		object,
		method,
		parameterValues
);

assertEquals( 1, violations.size() );
Class<? extends Annotation> constraintType = violations.iterator()
		.next()
		.getConstraintDescriptor()
		.getAnnotation()
		.annotationType();
assertEquals( Max.class, constraintType );
```

#### ExecutableValidator#validateReturnValue()

```java
Car object = new Car( "Morris" );
Method method = Car.class.getMethod( "getPassengers" );
Object returnValue = Collections.<Passenger>emptyList();
Set<ConstraintViolation<Car>> violations = executableValidator.validateReturnValue(
		object,
		method,
		returnValue
);

assertEquals( 1, violations.size() );
Class<? extends Annotation> constraintType = violations.iterator()
		.next()
		.getConstraintDescriptor()
		.getAnnotation()
		.annotationType();
assertEquals( Size.class, constraintType );
````

#### ExecutableValidator#validateConstructorParameters()

```java
Constructor<Car> constructor = Car.class.getConstructor( String.class );
Object[] parameterValues = { null };
Set<ConstraintViolation<Car>> violations = executableValidator.validateConstructorParameters(
		constructor,
		parameterValues
);

assertEquals( 1, violations.size() );
Class<? extends Annotation> constraintType = violations.iterator()
		.next()
		.getConstraintDescriptor()
		.getAnnotation()
		.annotationType();
assertEquals( NotNull.class, constraintType );
```

####  ExecutableValidator#validateConstructorReturnValue()

```java
//constructor for creating racing cars
Constructor<Car> constructor = Car.class.getConstructor( String.class, String.class );
Car createdObject = new Car( "Morris", null );
Set<ConstraintViolation<Car>> violations = executableValidator.validateConstructorReturnValue(
		constructor,
		createdObject
);

assertEquals( 1, violations.size() );
Class<? extends Annotation> constraintType = violations.iterator()
		.next()
		.getConstraintDescriptor()
		.getAnnotation()
		.annotationType();
assertEquals( ValidRacingCar.class, constraintType );
```

## 自定义message

## 分组

## 自定义约束