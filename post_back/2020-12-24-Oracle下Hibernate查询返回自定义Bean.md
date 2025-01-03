---
layout: post
title: 'Oracle下Hibernate查询返回自定义Bean'
date: 2020-12-24
author: Anarkh-Lee
cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: Hibernate







---

> 开发过程中，有时会遇到操作多表数据，返回的数据属于多表字段，将这些字段封装成一个bean，Hibernate查询返回自定义bean

代码示例：

```java
		//第一种：使用addScalar()后使用Transformers.aliasToBean()自动完成bean数据封装
		Session session = null;
		List<CC03DTO> list = new ArrayList<CC03DTO>();
		
		try {
			session = HibernateSessionFactory.factory.openSession();
			session.beginTransaction();
			
			
			Query query = session
			.createSQLQuery(
					"select t.aac001,t.aac002,t.aac006 from cc03 t WHERE aac002 = '1' ")
			.addScalar("aac001")
			.addScalar("aac002")
			.addScalar("aac006")
			.setResultTransformer(
					Transformers.aliasToBean(CC03DTO.class));
			
			list = query.list();
		} catch (Exception e) {
			e.printStackTrace();
			session.getTransaction().rollback();
			throw new LemisException(e,
					Constants.LEMIS_ERRNO_BUSINESS_EXCEPTION_M);
		} finally {
			if (session != null) {
				session.close();
			}
		}


		//第二种，手动赋值
		Session session = null;
		Nm_Zj92_WebDTO dto = new Nm_Zj92_WebDTO();
		
		try {
			session = HibernateSessionFactory.factory.openSession();
			
			String sql = "";
	        sql = " select t.aab004,t.aab001,t.aab998 from AB01 t where 1=1 ";
	        if (azj924 != null && !"".equals(azj924)) {
	            sql +=  " and t.aab998 = ?1";
	        }
	        Query q = null;
	        
	        if (azj924 != null && !"".equals(azj924)) {
	        	q.setParameter(1, "" + azj924 + "");
	        }
	        
	        ((SQLQuery) q).addScalar("aab004").addScalar("aab001").addScalar("aab998");
	        
	        List list = q.list();
	        if(list != null && !list.isEmpty()){
	        	Object[] object = (Object[]) list.get(0);
	        	dto.setAab004(object[0].toString() == null ? "" : object[0].toString());
	        	dto.setAab001(Long.valueOf(object[1].toString()));
	        	dto.setAzj924(object[2].toString() == null ? "" : object[2].toString());
	        }
		} catch (Exception e) {
			e.printStackTrace();
		}finally{
			if (session != null) {
				session.close();
			}
		}
```

CC03DTO：

```java
import java.math.BigDecimal;
import java.sql.Timestamp;

public class CC03DTO {
    private BigDecimal aac001;
    private String aac002;
    private Timestamp aac006;

    public BigDecimal getAac001() {
        return aac001;
    }

    public void setAac001(BigDecimal aac001) {
        this.aac001 = aac001;
    }

    public String getAac002() {
        return aac002;
    }

    public void setAac002(String aac002) {
        this.aac002 = aac002;
    }

    public Timestamp getAac006() {
        return aac006;
    }

    public void setAac006(Timestamp aac006) {
        this.aac006 = aac006;
    }
}

```





<iframe type="text/html" width="100%" height="385" src="http://www.youtube.com/embed/gfmjMWjn-Xg" frameborder="0"></iframe>