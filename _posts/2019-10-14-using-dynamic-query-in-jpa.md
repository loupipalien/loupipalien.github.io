---
layout: post
title: "在 Jpa 中使用动态查询"
date: "2019-10-14"
description: "在 Jpa 中使用动态查询"
tag: [jpa, java]
---

### 在 Jpa 中使用动态查询
假定实体类 UserInfo 定义如下
```
@Data
public class UserInfo {
    private Long id;
    private String name;              // 昵称
    private String gender;            // 性别
    private String description;       // 个人说明
}
```
查询时可能按单个或多个条件查询, 动态查询适合用于这个场景; 以下有两种方式实现

#### 通过 Criteria API
```Java
public class UserInfoExtendRepository {

    @PersistenceContext
    EntityManager manager;

    public List<UserInfo> getUserInfo(String name, String gender) {
        CriteriaBuilder builder = manager.getCriteriaBuilder();
        CriteriaQuery<UserInfo> query = builder.createQuery(UserInfo.class);
        // 查询的表
        Root<UserInfo> root = query.from(UserInfo.class);
        // 拼接过滤条件
        List<Predicate> predicates = Lists.newArrayList();
        if (name != null) {
            predicates.add(builder.equal(root.get("name"), name));
        }
        if (gender != null) {
            predicates.add(builder.equal(root.get("gender"), gender));
        }
        query.where(Iterables.toArray(predicates, Predicate.class));
        // 执行查询
        TypedQuery<TblStatistics> typedQuery = manager.createQuery(query);
        // 返回结果
        return typedQuery.getSingleResult();
    }
}
```
#### 通过实现 JpaSpecificationExecutor 接口
`JpaSpecificationExecutor` 如下, 方法参数 `Specification` 接口有一个方法 `toPredicate`, 返回值正好是 `Predicate`, 而 `Predicate` 相对于 SQL 的过滤条件; 与上一个方法相比, 这种写法不需要指定查询的表是哪一张, 也不需要通过 `Criteria` 实现排序和分页
```Java
public interface JpaSpecificationExecutor<T> {
    T findOne(Specification<T> spec);

    List<T> findAll(Specification<T> spec);

    Page<T> findAll(Specification<T> spec, Pageable pageable);

    List<T> findAll(Specification<T> spec, Sort sort);

    long count(Specification<T> spec);
}
```
`UserInfoRepository` 继承 `JpaSpecificationExecutor`
```Java
public interface UserInfoRepository extends PagingAndSortingRepository<UserInfo, String>, JpaSpecificationExecutor<UserInfo> {}
```
实现 `Specification`
```Java
public static Specification<UserInfo> getSpecification(String name, String gender) {
       return new Specification<UserInfo>() {
           @Override
           public Predicate toPredicate(Root<UserInfo> root, CriteriaQuery<?> query, CriteriaBuilder builder) {
               Predicate predicate = null;
               if (name != null) {
                   Predicate p = builder.equal(root.get("name"), name);
                   predicate = predicate != null ? builder.add(predicate, p) : p;
               }
               if (gender != null) {
                   Predicate p = builder.equal(root.get("gender"), gender);
                   predicate = predicate != null ? builder.add(predicate, p) : p;
               }
               return predicates;
           }
       };
   }
```

>**参考:**
[Spring Data JPA 动态查询](https://www.jianshu.com/p/45ad65690e33#)
[JPA 自定义返回字段映射](https://www.jianshu.com/p/fbd157c3b4a4)
