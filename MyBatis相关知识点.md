<!-- TOC -->

- [MyBatis相关](#mybatis%E7%9B%B8%E5%85%B3)
    - [通用知识点](#%E9%80%9A%E7%94%A8%E7%9F%A5%E8%AF%86%E7%82%B9)
        - [MyBatis Where语句中判断字段是否存在](#mybatis-where%E8%AF%AD%E5%8F%A5%E4%B8%AD%E5%88%A4%E6%96%AD%E5%AD%97%E6%AE%B5%E6%98%AF%E5%90%A6%E5%AD%98%E5%9C%A8)
        - [MyBatis-in](#mybatis-in)
        - [批量insert，批量保存](#%E6%89%B9%E9%87%8Finsert%EF%BC%8C%E6%89%B9%E9%87%8F%E4%BF%9D%E5%AD%98)
        - [批量update写法](#%E6%89%B9%E9%87%8Fupdate%E5%86%99%E6%B3%95)
        - [分页查询](#%E5%88%86%E9%A1%B5%E6%9F%A5%E8%AF%A2)
    - [错误&异常案例](#%E9%94%99%E8%AF%AF%E5%BC%82%E5%B8%B8%E6%A1%88%E4%BE%8B)
        - [exception: org.apache.ibatis.binding.BindingException: Invalid bound statement (not found)](#exception-orgapacheibatisbindingbindingexception-invalid-bound-statement-not-found)
        - [Text和Blob字段问题](#text%E5%92%8Cblob%E5%AD%97%E6%AE%B5%E9%97%AE%E9%A2%98)
        - [resultMap和resultType常见错误](#resultmap%E5%92%8Cresulttype%E5%B8%B8%E8%A7%81%E9%94%99%E8%AF%AF)

<!-- /TOC -->

# MyBatis相关

## 通用知识点

MyBatis如果查询字段到BaseResultMap中，查询的字段全部为null的化，其返回的实体类也是null，所以保险起见，必须把id查询出来。

范例：

```
MiniappUser selectNicknameAvatarurl(@Param("openId") String openId);
```

```
<select id="selectNicknameAvatarurl" resultMap="BaseResultMap">
    select id, nick_name, avatarUrl from hc_hiwork_center.t_hc_miniapp_user
    where open_id = #{openId}
    limit 1
</select>
```

### MyBatis Where语句中判断字段是否存在

单一条件

```
<if test="corpId!=null">
	and corp_id = #{corpId}
</if>
```

多条件

```
<if test="beginDate != null and endDate != null ">
	and c.create_time between #{beginDate,jdbcType=TIMESTAMP} AND #{endDate,jdbcType=TIMESTAMP}
</if>
```

### MyBatis-in

时长会遇到需要写`where a in ('1','2')`的写法，常规的脚本写法无法满足，其实MyBatis提供了相应的方案：

DAO层方法定义：

```
List<MenuDTO> selectMenuByRoleInfo(MenuByRoleInfoDTO dto);
```

注意方法的入参，其中包含一个List：
```
public class MenuByRoleInfoDTO implements Serializable {

    private static final long serialVersionUID = 1L;

    /**
     * 角色id列表
     */
    List<Long> roleIdList;

    public List<Long> getRoleIdList() {
        return roleIdList;
    }
    
    public void setRoleIdList(List<Long> roleIdList) {
        this.roleIdList = roleIdList;
    }

    public static long getSerialversionuid() {
        return serialVersionUID;
    }
    
}
```

在Mapper中的读取方法为：
```
-- ...省略前半段...
<if test="list != null and list.size() > 0 ">
	and role.id in
  <foreach collection="list" item="item" index="index" open="(" separator="," close=")">#{item}</foreach>
</if>
```

由于可以识别类中的List，故此方法支持多个参数的传入

补充：

如果参数只有一个List，例如`List<Long>`，则Mapper中的写法更换为：
```
-- ...省略前半段...
where role.id in
<foreach collection="list" item="item" index="index" open="(" separator="," close=")">#{item}</foreach>
```

区别为`collection`中的参数应当为`list`，其他不变。

### 批量insert，批量保存

有时候需要同一时间保存大量的数据，如果每次都是发起一个insert，效率非常低，这时候就用到批量保存了。

但是需要注意，如果一次性保存太多，会导致保存失败。可能跟MySQL的IO限制有关，笔者曾经一次保存7000条，直接报出异常。具体上限不清楚，500条以内还是安全的。

DAO层代码，入参通常为实体类的List：

```
int saveRoleRelationBatch(List<RoleBusinessUserRelation> records);
```

Mapper代码：

```
<insert id="saveRoleRelationBatch" parameterType="ArrayList">
	insert into t_hc_suite_role_business_user_relation (id, business_user_id, role_id, corp_id, platform_type, app_code, creator_business_user_id)
	<foreach collection="list" item="item" index="index" separator="union all">
		select
		#{item.id,jdbcType=BIGINT}, #{item.businessUserId,jdbcType=BIGINT},
		#{item.roleId,jdbcType=BIGINT},
		#{item.corpId,jdbcType=VARCHAR},
		#{item.platformType,jdbcType=VARCHAR}, #{item.appCode,jdbcType=VARCHAR},
		#{item.creatorBusinessUserId,jdbcType=BIGINT}
	</foreach>
</insert>
```

### 批量update写法

MySQL支持批量update，SQL为：
```sql
UPDATE TABLE SET column = CASE id WHEN 1 THEN 'element' END
WHERE id IN (1,2);
```

对应的mybatis写法为：
```xml
<update id="updateStatisticBatch" parameterType="ArrayList">
  update t_hc_suite_scheduling_info set total_count =
  <foreach collection="list" item="item" index="index" separator=" " open="case id" close="end">
    when #{item.id} then #{item.totalCount}
  </foreach>
  ,owe_deposit =
  <foreach collection="list" item="item" index="index" separator=" " open="case id" close="end">
    when #{item.id} then #{item.oweDeposit}
  </foreach>
  ,annual_leave =
  <foreach collection="list" item="item" index="index" separator=" " open="case id" close="end">
    when #{item.id} then #{item.annualLeave}
  </foreach>
  where id in
  <foreach collection="list" index="index" item="item" separator="," open="(" close=")">
    #{item.id,jdbcType=BIGINT}
  </foreach>
</update>
```

根据list参数是否为空，选择性进行更新字段的写法

```xml
<update id="updateBatchById">
    update t_hc_suite_manage_scheduling_department_count
    <set>
        course_count =
        <foreach collection="list" item="item" index="index" separator=" " open="case id">
            <if test="item.courseCount != null">
                when #{item.id} then
                #{item.courseCount}
            </if>
        </foreach>
        when -1 then course_count
        else course_count end
        ,
        user_count =
        <foreach collection="list" item="item" index="index" separator=" " open="case id">
            <if test="item.userCount != null">
                when #{item.id} then
                #{item.userCount}
            </if>
        </foreach>
        when -1 then user_count
        else user_count end
        ,
    </set>
    where id in
    <foreach collection="list" index="index" item="item" separator="," open="(" close=")">
        #{item.id,jdbcType=BIGINT}
    </foreach>
</update>
```

### 分页查询

查询类继类：PageParamUtils

SQL语句（通常是最末尾）：
```
-- ...
<if test="numPerPage != null and numPerPage > 0 and startIndex != null and startIndex >= 0">
	limit #{startIndex},#{numPerPage}
</if>
```

## 错误&异常案例
### exception: org.apache.ibatis.binding.BindingException: Invalid bound statement (not found)

出现这个异常通常是因为实际的类（通常是VO）与查询出的结果不一致导致。需要检查输入输出，关键字、命名等等。

### Text和Blob字段问题

MyBatis的代码生成器会自动针对Blob字段做优化，例如：

```xml
<resultMap id="BaseResultMap" type="comcom.test.XXX">
    <id column="id" jdbcType="BIGINT" property="id" />
</resultMap>
<resultMap extends="BaseResultMap" id="ResultMapWithBLOBs" type="comcom.test.XXX">
<result column="allow_obj" jdbcType="LONGVARCHAR" property="allowObj" />
</resultMap>
<sql id="Base_Column_List">
id
</sql>
<sql id="Blob_Column_List">
allow_obj
</sql>
<select id="selectByPrimaryKey" parameterType="java.lang.Long" resultMap="ResultMapWithBLOBs">
select 
<include refid="Base_Column_List" />
,
<include refid="Blob_Column_List" />
from table
where id = #{id,jdbcType=BIGINT}
</select>
```

常规情况下这样的代码没有问题。但是一旦是我们将字段冲Varchar改成了Text，就会出问题了，以前代码里如果用到了`BaseResultMap`和`Base_Column_List`，而且想要把Text字段查询出来，则需要修改为`ResultMapWithBLOBs`和`<include refid="Base_Column_List" />,<include refid="Blob_Column_List" />`。

无论怎样，Text字段一定要单独使用一条SQL来查询，尽量避免和其他字段一起查询出来，使用select *来进行查询，这种是需要严格杜绝的。

Text字段是由长度限制的，普通的Text，最大为64kb，longText最大为4GB。一次性不要查询那么大的数据，请使用流式查询。

### resultMap和resultType常见错误

再些XML代码的时候，经常会复制粘贴，如果返回时一个List<T>的话，要特别注意类型是否与resultType对应。如果使用resultMap，则List里面会填充这个表的实体类。且不会报错！！！