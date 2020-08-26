# 修改final变量

>https://blog.csdn.net/vi__iv/article/details/47294243

```java
Field field = MallLoggerStatics.class.getDeclaredField("ENVIROMENT_CODE");
field.setAccessible(true);
Field modifiersField = Field.class.getDeclaredField("modifiers");
modifiersField.setAccessible(true);
modifiersField.setInt(field,field.getModifiers()&~Modifier.FINAL);
field.setInt(null, 1);
field.setAccessible(false);
```

