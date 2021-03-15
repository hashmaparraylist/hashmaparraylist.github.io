---
title: 重学Java (一) 泛型
date: 2021-03-15 16:01:46
tags: 
    - java
    - generic
categories:
    - 后端
---

## 1. 前言

泛型编程自从 Java 5.0 中引入后已经超过15个年头了。对于现在的 Java 码农来说熟练使用泛型编程已经是家常便饭的事情了。所以本文就在不对泛型的基础使用在做说明了。 如果你还不会使用泛型的话，可以参考下面两个链接

- [Java 泛型详解](https://blog.csdn.net/qq_24084925/article/details/68491132)
- [The Java™ Tutorials (Lesson: Generics)](https://docs.oracle.com/javase/tutorial/java/generics/index.html)

这篇文章就简答聊一下，我实际在开发工作中很少用的到泛型方法这个知识点，以及在实际项目中有哪些东西会使用到泛型。

## 2. 泛型方法

在阅读代码的时候我们经常会看到下面这样的方法 (这段代码摘自 `java.util.AbstractCollection`)

```java
 public <T> T[] toArray(T[] a) {
    // Estimate size of array; be prepared to see more or fewer elements
    int size = size();
    T[] r = a.length >= size ? a :
              (T[])java.lang.reflect.Array
              .newInstance(a.getClass().getComponentType(), size);
    Iterator<E> it = iterator();

    for (int i = 0; i < r.length; i++) {
        if (! it.hasNext()) { // fewer elements than expected
            if (a == r) {
                r[i] = null; // null-terminate
            } else if (a.length < i) {
                return Arrays.copyOf(r, i);
            } else {
                System.arraycopy(r, 0, a, 0, i);
                if (a.length > i) {
                    a[i] = null;
                }
            }
            return a;
        }
        r[i] = (T)it.next();
    }
    // more elements than expected
    return it.hasNext() ? finishToArray(r, it) : r;
}
```

那么 `pulic` 关键字后面的那个 `<T>` 就是用来标记这个方法是一个泛型方法。 那什么是泛型方法呢。

官方的解释是这样的

```
Generic methods are methods that introduce their own type parameters. This is similar to declaring a generic type, but the type parameter's scope is limited to the method where it is declared. Static and non-static generic methods are allowed, as well as generic class constructors.
```

通俗点来将就是将一个方法泛型化，让一个普通的类的某一个方法具有泛型功能。 如果在一个泛型类中增加一个泛型方法，那这个泛型方法就可以有一套独立于这个类的泛型类型。

通过一个简单的例子, 我们来看看

```java
/**
 * GenericClass 这个泛型类是一个简单的套皮的 HashMap
 */
public class GenericClass<K, V> {

    private Map<K, V> map = new HashMap<>();

    public V put(K key, V value) {
       return map.put(key, value);
    }

    public V get(K key) {
        return map.get(key);
    }

    // 泛型方法 genericMethod 可以接受一个全新的、作用域只限本函数的泛型类型T
    public <T> T genericMethod(T t) {
        return t;
    }

}
```

实际使用起来

```java
GenericClass<String, Integer> map = new GenericClass<>();
// put 和 get 方法的参数必须使用定义时指定的 String 和 Integer
System.out.println(map.put("One", 1));
System.out.println(map.get("One"));
// 泛型方法 genericMethod 就可以接受一个 String 和 Integer 以外的类型
System.out.println(map.genericMethod(new Double(1.0)).getClass());
```

我们再来看看 JDK 中使用到泛型方法的例子。我们最常使用的泛型容器 `ArrayList` 中有个 `toArray` 方法。JDK 在它的实现中就提供了两个版本，其中一个就是泛型方法的版本

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable 
{
    // 这是一个普通版本，返回一个Object的数组
    public Object[] toArray() {
        return Arrays.copyOf(elementData, size);
    }
    
    // 这是一个泛型方法的版本，将容器里存储的元素输出到 T[] 数组中。 其中 T 必须是 E 的父类，否则 System.arraycopy 会抛出 ArrayStoreException 异常      
    public <T> T[] toArray(T[] a) {
        if (a.length < size)
            // Make a new array of a's runtime type, but my contents:
            return (T[]) Arrays.copyOf(elementData, size, a.getClass());
        System.arraycopy(elementData, 0, a, 0, size);
        if (a.length > size)
            a[size] = null;
        return a;
    }
}
```

泛型方法总体上来说就是可以给与现有的方法实现上，增加一个更加灵活的实现可能。

## 3. 实战应用

在实际的项目中，对于泛型的使用，除了像倾倒垃圾一样往泛型容易里塞各种 java bean 和其他泛型对象。还能怎么使用泛型呢？

我们在实际的一些项目中，会对数据库中的一些表(多数时候是全部)先实现 CRUD (Create, Read, Update, Delete)的操作，并从这些操作中延伸出一些简单的 REST 风格的 WebAPI 接口，然后才会根据实际业务需要实现一些更复杂的业务接口。 

大体上会是下面这个样子。 

```java

// 这是一个简单的 Entity 对象
// 通常现在的 Java 应用都会使用到 Lombok 和 Spring Boot
@Data
@AllArgsConstructor
@NoArgsConstructor
@ToString
@Entity
@Table(name = "user")
public class User {
    @Id
    private Long id;
    private String username;
    private String password;
}

// 然后这个是 DAO 接口继承自 spring-data 的 JpaRepository
public interface UserDao extends JpaRepository<User, Long> {
}

// 在来是一个访问 User 资源的 Service 和他的实现
public interface UserService {
    List<User> findAll();
    Optional<User> findById(Long id);
    User save (User user)
    void deleteById(Long id);
}

@Service
public class UserSerivceImpl implements UserService {
    private UserDao userDao;
    public UserServiceImpl(UserDao userDao) {
        this.userDao = userDao;
    }

    @Override
    public List<User> findAll() {
        return this.dao.findAll();
    }

    @Override
    public Optional<User> findById(Long id) {
        return this.dao.findById(id);
    }

    @Override
    public User save(User user) {
        return this.dao.save(user);
    }

    @Override
    public void deleteById(Long id) {
        this.dao.deleteById(id);
    }
}

// 最后就是 WebAPI 的接口了
@RestController
@RequestMapping("/user/")
public class UserController{
    private UserService userService;
    public UserController(userService userService) {
        this.userService = userService;
    }

    @GetMapping
    @ResponseBody
    public List<User> fetch() {
        return this.userService.findAll();
    }

    @GetMapping("{id}")
    @ResponseBody
    public User get(@PathVariable("id") Long id) {
        // 由于是示例这里就不考虑没有数据的情况了
        return this.userService.findById(id).get();
    }

    @PostMapping
    @ResponseBody
    public User create(@RequestBody User user) {
        return this.userService.save(user);
    }

    @PutMapping("{id}")
    @ResponseBody
    public User update(@RequestBody User user) {
        return this.userService.save(user);
    }

    @DeleteMapping("{id}")
    @ResponseBody
    public User delete(@PathVariable("id") Long id) {
        User user = this.userService.findById(id);
        this.userService.deleteById(id);
        return user;
    }
}
```

大致一个表的一套相关接口就是这个样子的。如果你的数据库中有大量表的话，而且每个表都需要提供 REST 风格的 WebAPI 接口的话，那么这将是一个相当枯燥的而又及其容易出错的工作。

为了不让这项枯燥而又容易犯错的工作占去我们宝贵的私人时间，我们可以通过泛型和继承的技巧来重构从 Service 层到 Controller 的这段代码(感谢 spring-data 提供了 `JpaRepository`, 让我们不至于从 DAO 层重构)

## 3.1 Service 层的重构

首先是 Service 接口的重构，我们 Service 层接口就是定义了一组 CRUD 的操作，我们可以将这组 CRUD 操作抽象到一个父接口，然后所有 Service 层的接口都将继承自这个父接口。而接口中出现的 Entity 和主键的类型(上例中 User 的主键 id 的类型是 Long)就可以用泛型来展现。

```java
// 这里泛型表示 E 来指代 Entity, ID 用来指代 Entity 主键的类型
public interface ICrudService<E, ID> {
    List<E> findAll();
    Optional<E> findById(ID id);
    E save(E e);
    void deleteById(ID id);
}

// 然后 Service 层的接口，就可以简化成这样
public interface UserService extends ICrudService<User, Long> {
}
```

同样 Service 层的实现也可以使用相似的方法具体实现可以抽象到一个基类中。

```java
// 相比 ICrudService 这里有多了一个泛型 T 来代表 Entity 对应的 DAO, 我们的每一个 DAO 都继承自
// spring-data 的 JpaRepository 所以，这里可以使用到泛型的边界
public abstract class AbstractCrudService<T extends JpaRepository<E, ID>, E, ID> {
    private T dao;
    public AbstractCrudService(T dao) {
        this.dao = dao;
    }

    public List<E> findAll() {
        return this.dao.findAll();
    }

    public Optional<E> findById(ID id) {
        return this.dao.findById(id);
    }

    public E save(E e) {
        return this.dao.save(e);
    }

    public void deleteById(ID id) {
        this.dao.deleteById(id);
    }
}

// 那 Service 的实现类可以简化成这样
@Service
public class UserServiceImpl extends AbstractCrudService<UserDao, User, Long> implements UserService {
    public UserServiceImpl(UserDao dao) {
        supper(dao);
    }
}
```

同样我们可以通过相同的方法来对 Controller 层进行重构

```java
// Controller 层的基类
public abstract class AbstractCrudController<T extends ICrudService<E, ID>, E, ID> {
    private T service;
    public AbstractCrudController(T service) {
        this.service = service;
    }

    @GetMapping
    @ResponseBody
    public List<E> fetch() {
        return this.service.findAll();
    }

    @GetMapping("{id}")
    @ResponseBody
    public E get(@PathVariable("id") ID id) {
        // 由于是示例这里就不考虑没有数据的情况了
        return this.service.findById(id).get();
    }

    @PostMapping
    @ResponseBody
    public E create(@RequestBody E e) {
        return this.service.save(e);
    }

    @PutMapping("{id}")
    @ResponseBody
    public E update(@RequestBody E e) {
        return this.service.save(e);
    }

    @DeleteMapping("{id}")
    @ResponseBody
    public E delete(@PathVariable("id") ID id) {
        E e = this.service.findById(id).get();
        this.service.deleteById(id);
        return e;
    }
}

// 具体的 WebAPI
@RestController
@RequestMapping("/user/")
public class UserController extends AbstractCrudController<UserService, User, Long> {
    public UserController(UserService service) {
        super(service);
    }
}
```

经过重构可以消减掉 Servcie 和 Controller 中的大量重复代码，使代码更容易维护了。

## 4. 结尾

关于泛型就简单的说这些了，泛型作为 Java 日常开发中一个常用的知识点，其实还有很多知识点可以供我们挖掘，奈何本人才疏学浅，这么多年工作下来，只积累出来这么点内容。

文末放上示例代码的代码库: 

- [GitHub入口](https://github.com/hashmaparraylist/re-study-java)
- [gitee入口](https://gitee.com/hashmaparraylist/re-study-java)

