# 获取List最后一个元素的工具类方法

1. **Apache Commons Collections**
```java
import org.apache.commons.collections4.CollectionUtils;

List<String> list = Arrays.asList("a", "b", "c");

// 方法1：使用CollectionUtils.get()
String last = CollectionUtils.get(list, list.size() - 1);

// 方法2：使用ListUtils.defaultIfEmpty()
String last = ListUtils.defaultIfEmpty(list, Collections.emptyList())
                      .get(list.size() - 1);
```

2. **Google Guava**
```java
import com.google.common.collect.Iterables;

List<String> list = Arrays.asList("a", "b", "c");

// 方法1：使用Iterables.getLast()
String last = Iterables.getLast(list);

// 方法2：带默认值的方式
String last = Iterables.getLast(list, "default");
```

3. **Java 8 Stream API**
```java
List<String> list = Arrays.asList("a", "b", "c");

// 方法1：使用Stream
Optional<String> last = list.stream().reduce((first, second) -> second);

// 方法2：使用skip
Optional<String> last = list.stream()
                           .skip(Math.max(0, list.size() - 1))
                           .findFirst();
```

4. **最佳实践建议**
```java
// 推荐使用Guava的方式，代码最简洁且自带空值处理
public String getLastElement(List<String> list) {
    return Iterables.getLast(list, null);
}

// 如果需要返回Optional
public Optional<String> getLastElementSafe(List<String> list) {
    return Optional.ofNullable(Iterables.getLast(list, null));
}
```

5. **注意事项**
- 所有方法在list为空时的处理：
  - Apache Commons：抛出IndexOutOfBoundsException
  - Guava：如果没有默认值会抛出NoSuchElementException
  - Stream：返回Optional.empty()
- 建议先判断list是否为空
- 考虑是否需要返回Optional 