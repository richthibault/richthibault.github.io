---
layout: post
title: Queries leading to data modification are not allowed
tags:
 - Hibernate
 - Spring
 - MySQL
---

Man, this one had me pulling my hair out.  I was doing something I generally try to avoid - calling a stored procedure in MySQL from a Spring/Hibernate JPA app.  Something like this:

{% highlight java %}
StoredProcedureQuery storedProcedure = em.createStoredProcedureQuery("getSomeData");
storedProcedure.registerStoredProcedureParameter("myId", Integer.class, ParameterMode.IN);

storedProcedure.setParameter("in_id", myObj.getId());
storedProcedure.execute();
List<Object[]> rows = (List<Object[]>) storedProcedure.getResultList();
{% endhighlight %}

It gets some structured data from the database, and loads it into a list for further processing.  So far so good, and it worked fine for several months.  Until I tried to add a new feature the other day, calling this proc from within another transaction, and started getting this error:

{% highlight java %}
org.hibernate.exception.GenericJDBCException: Error calling CallableStatement.getMoreResults
Caused by: java.sql.SQLException: Connection is read-only. Queries leading to data modification are not allowed
{% endhighlight %}

All the Java methods were wrapped in `@Transactional(readOnly=true)`, so I couldn't understand why it thought there would be "data modification".  I tried changing the readOnly flag back and forth on all the methods involved, but couldn't shake it.

Finally I found that the fix is to add `READS SQL DATA` to the MySQL procedure.

{% highlight sql %}
CREATE PROCEDURE getSomeData(in_id int)
    READS SQL DATA
    BEGIN....
{% endhighlight %}

Now Hibernate and the MySQL driver can validate that no data will be changed, the error goes away, and I can stop yanking my hair out.