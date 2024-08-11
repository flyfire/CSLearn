Mockk 常用的功能

[toc]

## 注意事项

如果在有时候使用了静态的方式 mock 了对象，如果在最后放了执行 clearMock 或者 unmockkAll 函数，会导致对象泄露，造成其他其他 test 失败。

包含：

```kotlin
@Before
fun setup(){
    mockkConstructor()
    mockkStatic()
    mockkObject()
}

@After
fun teardown(){
    unmockkConstructor()
    unmockkObject()
    unmockkObject()

    // 或者
    unmockkAll()
}
```

或者升级到 junit5, 会自动释放
https://github.com/mockk/mockk/pull/739

## Spy
mock 一个真实的对象

```kotlin
// 测试 spy
@Test
fun testSpy(){
    val engine = Engine()
    val car = spyk(Car(engine))
    car.drive(Direction.NORTH)
    verify { car.drive(Direction.NORTH) }
}
```
## Partial mocking
使用 `callOriginal()` 在 `answers` 中返回，可以进行部分的 mock.

```kotlin
//  测试 Partial mocking
@Test
fun testPartialMocking() {
    val adder = mockk<Adder>() // 调用 Adder 的默认构造函数

    every { adder.addOne(any()) } returns -1
    every { adder.addOne(3) } answers { callOriginal() }

    val result1 = adder.addOne(2)
    assertThat(result1).isEqualTo(-1)

    val result2 = adder.addOne(3)
    assertThat(result2).isEqualTo(4)
}
```


## Capturing
需要知道函数的参数

```kotlin
// 测试 capture
@Test
fun testCapturing() {
    val engine = Engine()
    val car = spyk(Car(engine))

    val slotDirection = slot<Direction>()
    every { car.drive(capture(slotDirection)) } just runs

    car.drive(Direction.NORTH)

    assertThat(slotDirection.captured).isEqualTo(Direction.NORTH)
}
```


## 测试协程

添加依赖

```shell
testImplementation "org.jetbrains.kotlinx:kotlinx-coroutines-test:$rootProject.coroutines"
```

```kotlin
  // 测试协程
@OptIn(ExperimentalCoroutinesApi::class)
@Test
fun testCoroutine() = runBlockingTest{
    val engine = Engine()
    val car = spyk(Car(engine))

    coEvery { car.start() } coAnswers { callOriginal() }

    val startResult = car.start()

    assertThat(startResult).isTrue()
}
```

##  测试拓展函数

不同层级的拓展函数

```kotlin
class Ext{
    //Class 层级的拓展
    fun Car.extFun() = getSpeed() + 1
}

// module 层级的拓展
fun Car.extFun2() = getSpeed() + 1

// Object 层级的拓展
object MyObject{
    fun Car.extFun3() = getSpeed() + 1
}

```

测试

```kotlin
// 测试拓展函数 Class 层级
@Test
fun testExtensionFun() {
    val car = mockk<Car>()
    every { car.getSpeed() } answers { callOriginal() }

    with(mockk<Ext>()) {
        every { car.extFun() } returns 4
        val extFunResult = car.extFun()
        assertThat(extFunResult).isEqualTo(4)
        verify { car.extFun() }
    }
}

// 测试拓展函数 module 层级
@Test
fun testExtensionFun2() {
    val car = mockk<Car>()
    every { car.getSpeed() } answers { callOriginal() }
    every { car.extFun2() } returns 5

    mockkStatic("com.yxhuang.androiddailydemo.mockk.CarKt")
    val extFunResult = car.extFun2()

    assertThat(extFunResult).isEqualTo(6)
    verify { car.extFun2() }
}

// 测试拓展函数 Object 层级
@Test
fun testExtensionFun3() {
    val car = mockk<Car>()
    every { car.getSpeed() } answers { callOriginal() }

    mockkObject(MyObject)
    with(MyObject){
        every { car.extFun3() } returns 6
    }
    val extFunResult = car.extFun3()

    assertThat(extFunResult).isEqualTo(6)
    verify { car.extFun3() }
}

```

## 测试私有函数

```kotlin
 // 测试私有方法
    @Test
    fun testPrivateCalSpeed() {
        val engine = spyk<Engine>(recordPrivateCalls = true)
        val car = Car(engine)

        // 调用私有函数 calSpeed()
        every { engine["calSpeed"]() } returns 20
        val speed = car.getSpeed()

        assertThat(speed).isEqualTo(20)

        // 反射调用私有函数 setName()
        InternalPlatformDsl.dynamicCall(engine, "setName", arrayOf("test name")) {
            mockk()
        }

        // 反射获取私有属性 name
        val name = InternalPlatformDsl.dynamicGet(engine, "name")
        println("dynamicGet name: $name ")
        assertThat(name).isEqualTo("test name")

        // 反射设置私有属性 name
        InternalPlatformDsl.dynamicSet(engine, "name", "name2")
        // 反射调用私有函数 getName
        val name2 = InternalPlatformDsl.dynamicCall(engine, "getName", arrayOf()) { mockk() }
        assertThat(name2).isEqualTo("name2")

        // 验证私有函数被调用
        car.getSpeed()
        verify { engine.invoke("setName").withArguments(arrayListOf("calSpeed test")) }
    }

```

例子二：

```kotlin
class PrivateFunctionsTest {

    class GenericsClass {
        private fun <T> updateItem(id: Long, name: String, date: T) {
            print("updateItem id $id, name $name")

        }

        fun pubCall() {
            updateItem(1L, "abc", "DATA")
        }
    }

    @Test
    fun testPrivateMethodWitGenericsClass() {
        val genericsClass = spyk<GenericsClass>()

        every {
            genericsClass["updateItem"](any<Long>(), any<String>(), any()) as Unit
        } just Runs

        genericsClass.pubCall()
    }


    class Abc {
        fun y() = x()

        private fun x() = "abc"
        
        fun a(genericsClass: GenericsClass) {
            z(genericsClass) {
                println(" it: $it")
            }
        }

        private fun z(genericsClass: GenericsClass, callback: (Boolean) -> Unit) {

        }
    }

    @Test
    fun testPrivateMethod() {
        val abc = spyk(Abc(), recordPrivateCalls = true)
        every { abc["x"]() } returns "def"

        val result = abc.y()

        assertThat(result).isEqualTo("def")
        verifySequence {
            abc.y()
            abc["x"]()
        }
    }
    
     @Test
    fun testPrivateMethodWithHigherOrderFunctionArg() {
        val abc = spyk(Abc(), recordPrivateCalls = true)
        val callback: (Boolean) -> Unit = mockk(relaxed = true)
        val genericsClass = mockk<GenericsClass>()

        justRun { abc invoke "z" withArguments listOf(genericsClass, callback)}

        abc.a(genericsClass)

        verifySequence {
            abc.a(genericsClass)
            abc["z"](any<GenericsClass>(), any<(Boolean) -> Unit>())
        }
    }


    @Test
    fun spyNoRecordingPrivateMethod() {
        val abc = spyk<Abc>()
        every { abc["x"]() } returns "def"

        val result = abc.y()

        assertThat(result).isEqualTo("def")
        verifySequence {
            abc.y()
        }
    }

    object Def {
        fun y() = x()
        private fun x() = "def"
    }

    // 测试 Object 私有函数
    @Test
    fun testObjectPrivateMethod() {
        mockkObject(Def, recordPrivateCalls = true)

        every { Def["x"]() } returns "ghi"

        val result = Def.y()

        assertThat(result).isEqualTo("ghi")
        verify {
            Def.y()
            Def["x"]()
        }
    }

    // 测试 Object 私有函数
    @Test
    fun testNoRecordingObjectPrivateMethod() {
        mockkObject(Def)

        every { Def["x"]() } returns "ghi"

        val result = Def.y()

        assertThat(result).isEqualTo("ghi")
        verify {
            Def.y()
        }
    }

    class MockClass {
        fun y(a: Int, b: Int?, def: Def?) = x(a, b, def)

        private fun x(a: Int, b: Int?, d: Def?) = "abc $a $b"
    }

    // 测试可空参数的私有函数
    @Test
    fun testPrivateCallsWithNullability() {
        val mockkClass = spyk<MockClass>(recordPrivateCalls = true)

        every { mockkClass["x"](any<Int>(), any<Int>(), any<Def>()) } returns "Test"

        val result = mockkClass.y(1, 3, null)

        assertThat(result).isEqualTo("Test")
        verify {
            mockkClass["x"](any<Int>(), any<Int>(), any<Def>())
        }
    }

    class PrivateNoReturnClass {
        fun myPublicMethod() {
            myPrivateMethod()
        }

        private fun myPrivateMethod() {

        }
    }

    @Test
    fun testPrivateMethodThatReturnsNoting(){
        val myClass = spyk<PrivateNoReturnClass>(recordPrivateCalls = true)
        every { myClass invokeNoArgs "myPrivateMethod" } returns Unit

        myClass.myPublicMethod()

        verify {
            myClass invokeNoArgs "myPrivateMethod"
        }
    }

    @Test
    fun justRunsWithPrivateMethod(){
        val mock = spyk<PrivateNoReturnClass>(recordPrivateCalls = false)
        justRun { mock invokeNoArgs "myPrivateMethod" }

        mock.myPublicMethod()
    }
}

```

## 测试私有属性

```kotlin
class PrivatePropertiesTest {

    data class Person(var name: String)

    class Team {
        protected var person: Person = Person("Init")
            get() = Person("Ben")
            set(value) {
                field = value
                print("set person value $value")
            }

        protected fun fn(arg: Int): Int = arg + 5
        fun pubFn(arg: Int) = fn(arg)

        var memberName: String
            get() = person.name
            set(value) {
                person = Person(value)
                print("set memberName value $value")
            }
    }

    @Test
    fun testPrivateProperty() {
        val team = spyk(Team(), recordPrivateCalls = true)

        every { team getProperty "person" } returns Person("Big Ben")
        every { team setProperty "person" value Person("test")} just Runs
        every { team invoke "fn" withArguments listOf(5) } returns 3

        val memberNameValue = team.memberName
        assertThat(memberNameValue).isEqualTo("Big Ben")

        val fnResult = team.pubFn(5)
        assertThat(fnResult).isEqualTo(3)

        team.memberName = "test"
        assertThat(team.memberName).isEqualTo("Big Ben")

        verify { team getProperty "person" }
        verify { team setProperty  "person"  value Person("test")}
        verify { team invoke "fn" withArguments listOf(5) }
    }
}

```

## 测试 Null 参数

```kotlin
class NullsTest {

    interface Wrapper

    data class IntWrapper(val date: Int) :Wrapper

    class MockClass{

        fun function(a: Wrapper?, b: Wrapper?): Int?{
            return if (a is IntWrapper && b is IntWrapper){
                a.date + b.date
            } else {
                0
            }
        }
    }

    @MockK
    lateinit var mock: MockClass

    @Before
    fun setup(){
        MockKAnnotations.init(this)
    }

    @Test
    fun isNull(){
        every { mock.function(null, isNull()) } returns 4

        assertThat(mock.function(null, null)).isEqualTo(4)

        verify { mock.function(isNull(), null) }
    }

    @Test
    fun returnsNull(){
        every { mock.function(IntWrapper(1), IntWrapper(1)) } returns null

        assertThat(mock.function(IntWrapper(1), IntWrapper(1))).isEqualTo(null)

        verify { mock.function(IntWrapper(1), IntWrapper(1)) }
    }
}

```

## 测试 Relaxed 
MockK’s relaxed mocks are useful if the class under test uses a certain object, but you don’t want to define the behavior of this mock because it’s not relevant for the test. A relaxed mock returns dummy objects containing empty values.

```kotlin
class RelaxedMockingTest {

    class MockClass {

        fun option(a: Int, b: Int) = a + b

        fun optionUnit(a: Int, b: Int) {

        }
    }

    @Test
    fun testRegularOptionOk() {
        val mock = mockk<MockClass>(relaxUnitFun = true){
            every { option(1, 2) } returns 4
        }

        val result = mock.option(1, 2)

        assertThat(result).isEqualTo(4)
    }

    @Test
    fun testRegularOptionFail(){
        val mock = MockClass()

        assertFailsWith<MockKException> {
            assertEquals(4, mock.option(1, 2))
        }
    }
}

```




## 参考
- [mockk 官方文档](https://mockk.io/)
- [mockk 官方例子](https://github.com/mockk/mockk/tree/master/mockk/common/src/test/kotlin/io/mockk)



