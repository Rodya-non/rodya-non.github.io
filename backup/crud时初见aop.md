crud时create_time和update_time重复处理的优化。

## 1 数据库表结构处理
直接建表时使用字段控制更新规则，简单省事，自己码项目可以使用这个。
```sql
-- 添加创建时间字段（仅插入时赋值） 
ALTER TABLE `user` 
ADD COLUMN `create_time` DATETIME DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间' AFTER `username`; 
-- 添加更新时间字段（插入/更新时自动刷新） 
ALTER TABLE `user` 
ADD COLUMN `update_time` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间' AFTER `create_time`;
```

## 2 aop处理
苍穹外卖的实现方式。一般配合其他要更新的参数一起切面才有意义。
用户请求时发送token，jwt解析后用ThreadLocal存下用户信息，利用aop把userId和time等信息一起更新。

1. 制作注解
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AutoFill {
    OperationType value(); // 枚举类 INSERT 或 UPDATE
}
```

2. Aspect
```java
@Component  
@Slf4j  
@Aspect  
public class AutoFillAspect {  
  
    /**  
     * 切入点  
     * 指定拦截位置  
     */  
    @Pointcut("execution(* com.sky.mapper.*.*(..)) && @annotation(com.sky.annotation.AutoFill)")  
    public void autoFillPointCut(){}  
  
    @Before("autoFillPointCut()")  
    public void  autoFill(JoinPoint joinPoint){  
        log.info("自定义AOP,拦截处理公共字段");  
        //获取到当前被拦截的方法上的数据库操作类型   反射  
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();//方法签名对象  
        AutoFill autoFill = signature.getMethod().getAnnotation(AutoFill.class);//获得方法上的注解对象  
        OperationType operationType = autoFill.value();//获取数据库操作类型  
  
        //获取到当前被拦截的方法的参数--实体对象  
        Object[] args = joinPoint.getArgs();  
        if(args ==null || args.length==0){  
            return;  
        }  
  
        Object entity = args[0];  
  
        //包装好待调用的数据  其中BaseContext里封装了ThreadLocal 略
        LocalDateTime now = LocalDateTime.now();  
        Long currentId = BaseContext.getCurrentId();  
  
        //根据操作类型为切面执行不同的操作  
        if(operationType==OperationType.INSERT){  
            //四种  
            try {  
                Method setCreateTime = entity.getClass().getDeclaredMethod(SET_CREATE_TIME,LocalDateTime.class);  
                Method setUpdateTime = entity.getClass().getDeclaredMethod(SET_UPDATE_TIME,LocalDateTime.class);  
                Method setCreateUser = entity.getClass().getDeclaredMethod(SET_CREATE_USER,Long.class);  
                Method setUpdateUser = entity.getClass().getDeclaredMethod(SET_UPDATE_USER,Long.class);  
  
                setUpdateTime.invoke(entity,now);  
                setCreateTime.invoke(entity,now);  
                setCreateUser.invoke(entity,currentId);  
                setUpdateUser.invoke(entity,currentId);  
            } catch (Exception e) {  
                e.printStackTrace();  
            }  
        }else if(operationType==OperationType.UPDATE){  
            //两种  
            try {  
  
                Method setUpdateTime = entity.getClass().getDeclaredMethod(SET_UPDATE_TIME,LocalDateTime.class);  
                Method setUpdateUser = entity.getClass().getDeclaredMethod(SET_UPDATE_USER,Long.class);  
  
                setUpdateTime.invoke(entity,now);  
                setUpdateUser.invoke(entity,currentId);  
            } catch (Exception e) {  
                e.printStackTrace();  
            }  
        }  
    }  
}
```

3. 在Mapper层的方法上加上要用的注解。
```java
@AutoFill(value = OperationType.UPDATE)
```
