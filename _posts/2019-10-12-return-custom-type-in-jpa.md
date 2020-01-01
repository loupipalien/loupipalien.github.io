---
layout: post
title: "在 Jpa 中返回自定义类型"
date: "2019-10-12"
description: "在 Jpa 中返回自定义类型"
tag: [jpa, java]
---

### 在 Jpa 中返回自定义类型
假定实体类 User 定义如下
```
@Data
@Entity
@Table(name = "tbl_user")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private Long id;
    private String name;              // 昵称
    private String signature;         // 个性签名
    private String gender;            // 性别
    private String description;       // 个人说明
    private String avatar;            // 头像
    private Long role;                // 权限
    private Boolean disable;          // 是否冻结
    private Date createTime;          // 创建时间
    private Boolean isDelete;         // 是否删除
    private Long userId;              // 用户 Id
}
```
#### 将实体类中的部分字段封装为自定义类型
自定义返回类型定义如下
```
@Data
public class UserInfo {
    private Long id;
    private String name;              // 昵称
    private String gender;            // 性别
    private String description;       // 个人说明
}
```
由于自定义返回类 UserInfo 中的字段是实体类 User 的子集, 所以可以直接在 UserRepository 接口中定义方法
```
public interface UserRepository extends JpaRepository<User,Long> {

    /**
     * 用户名查询
     * @param name
     * @return
     */
    Optional<User> findByUsername(String name);

    /**
     * 用户名查询 (返回自定义类型)
     * @param name
     * @param type
     * @return
     */
    <T> Optional<T> findByUsername(String name, Class<T> type);
}
```
这是因为 Jpa 是基于反射做 ORM, 所以当自定义返回类型中的字段名称与实体类中的名称一致时, 可以使用这种方法; 如果当UserInfo 中的名称与实体类中不一致时
```
@Data
@AllArgsConstructor
public class UserInfo {
    private Long id;
    private String name;              // 昵称
    private String sex;               // 性别
    private String desc;              // 个人说明
}
```
可以使用 `@Query` 注解编写 JPQL 做映射
```
public interface UserRepository extends JpaRepository<User,Long> {

    ...

    /**
     * 用户名查询
     * @param username
     * @return
     */
    @Query(value = "SELECT new com.ltchen.demo.api.model.UserInfo(u.id, u.name, ui.sex, ui.desc) FROM User u WHERE u.name = ?1")
    UserInfo findByUsername(String name);
}
```

#### 将实体类中的部分字段封装为 Map 或 List 类型
```
public interface UserRepository extends JpaRepository<User,Long> {

    ...

    @Query(value = "SELECT new map(u.id, u.name, ui.sex, ui.desc) FROM User u WHERE u.name = ?1")
    List<Map<String,Object>> findByUsername(String name);

    @Query(value = "SELECT u.id, u.name, ui.sex, ui.desc FROM User u WHERE u.name = ?1")
    List<Object> findByUsername(String name);
}
```

#### 将复杂查询的字段封装为自定义类型
例如查询性别统计
```
@Data
@AllArgsConstructor
public class UserStatistics {

    private String sex;               // 性别
    private Long count;               // 统计值
}

public interface UserRepository extends JpaRepository<User,Long> {

    ...

    @Query(value = "SELECT new com.ltchen.demo.api.model.UserInfo(u.gender, count(u.id)) FROM User u GROUP BY u.gender")
    UserStatistics statisticsByGender();
}
```

>**参考:**
[JPA 自定义返回字段映射](https://www.jianshu.com/p/fbd157c3b4a4)  
[spring data jpa 查询自定义字段，转换为自定义实体](https://blog.csdn.net/zhu562002124/article/details/75097682)
