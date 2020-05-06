---
title: Code Review Detect SQLi in Java
author: WHmacmac
layout: post
permalink: Code_Review_Detect-SQLi_in_Java
category: blog
---


Why do we need code review when we can interact with the target and finding if it is vulnerable? After all the HTTP response's behavior should help us finding if something is not configured properly. Based on the attack, there are cases where we will not can deduce so easily if we are vulnerable or not. Making code review can help us finding what the interactive analysis misses.

## Contents
* [Introduction](#introduction)
* [Java Persistence API (JPA)](#jpa)
* [Hibernate](#hibernate)
* [PreparedStatement](#preparedstatement)
* [CallableStatement](#callablestatement)

## Introduction {#introduction}

There are a wide variety of SQLi techniques, attacks, vulnerabilities which can occur in different situations. Some of them include:
<ul>
<li>Obtaining hidden data.</li>
<li>Unsettle the application's logic.</li>
<li>Blind SQLi.</li>
<li>Examining the database.</li>
<li>Union attacks</li>
</ul>

SQLi vulnerabilities can in principle occur at any location within a query and in different query types. Most of us probably found SQLi within a WHERE clause of a SELECT query.<br/>
The most common other locations where a SQLi can occur are:

<ul>
<li>In SELECT statements, within the table or column name.</li>
<li>In SELECT statements, within the ORDER BY clause.</li>
<li>In INSERT statements, within the inserted values.</li>
<li>In UPDATE statements, within the updated values or the WHERE clause.</li>
</ul>

There are cases where the application takes the user's input and store it for a future use, instead of processing the user's input from a HTTP request and adding it into a SQL query.
This is usually done by storing the input into a database, no vulnerability arises at the point where the data is stored. The stored SQLi vulnerabilities are hard to be detected through interactive analysis.

Many programmers are thinking that today frameworks can resolve all the SQLi injection vulnerabilities, but this is wrong. Many are using SQL query and connections with the databases in a wrong manner. In what will follow, I will present some examples of how to detect SQLi in Java code and how to correctly write it.
   
## Java Persistence API (JPA) {#jpa}
Java Persistence API (JPA), is an ORM solution that is a part of the Java EE framework. It helps manage relational data in applications that use Java SE and Java EE. JPA allows the use of native SQL and defines its own query language, named, JPQL (Java Persistence Query Language). It is a common misconception that JPA (Java Persistence API) is SQL Injection proof. . 

### 1. Vulnerable usage of JPA

{% highlight java %}
List sql_result = EntityManager.createNativeQuery("Select * from PcComponents where component = " + component).GetResultList()
List sql_result = entityManager.createQuery("Select age from Persons age order where age.id = " + age.id).getResultList();
int sql_result = entityManager.createNativeQuery("Delete from Persons where CNP = " + CNP).executeUpdate();

{% endhighlight %}

I consider that component, age.id & CNP are user input. As you can see in the above examples, they have not been validated or escaped as required. Therefore, it leaves the above queries vulnerable to SQLi attacks.

### 2. Secure usage of JPA
<b>Native SQL<b/>

{% highlight java %}
Query sql_query = entityManager.createNativeQuery("Select * from PcComponents where component = ?", PcComponents.class);
List sql_result = sql_query.setParameter(1, "AMD Ryzen 5000").getResultList();

{% endhighlight %}

<b>Positional parameter in JPQL</b>
{% highlight java %}
Query jpql_query = entityManager.createQuery("Select person from Personss person where person.CNP = ?1");
List sql_result = jpql_query.setParameter(1, "1940000332211").getResultList();

{% endhighlight %}

<b>Named parameter in JPQL</b>
{% highlight java %}
Query jpql_query = entityManager.createQuery("Select employee from Employees employee where employee.salary > :salary");
List sql_result = jpql_query.setParameter("salary", new Long(1000)).getResultList();

{% endhighlight %}

<b>Named query in JPQL - Query named "PcComponents" being "Select component from PcComponents component where component.itemId = :itemId"</b>
{% highlight java %}
Query jpql_query = entityManager.createNamedQuery("PcComponents");
List sql_result = jpql_query.setParameter("itemId", "AMD-Ryzen-5000").getResultList();

{% endhighlight %}


If your JPA provider processes all input arguments to handle injection attacks then you should be covered.<br/>
Tip for devs: Never use string concatenation in your SQL queries.

## Hibernate 
Hibernate facilitates the storage and retrieval of Java domain objects via Object/Relational Mapping (ORM).Hibernate allows the use of "native SQL" and defines a proprietary query language, named, HQL (Hibernate Query Language).

### 1. Vulnerable usage of Hibernate
The story is repeating as you can see in the below examples, the inputs have not been validated or escaped as required. Therefore, it leaves the above queries vulnerable to SQLi attacks.
{% highlight java %}
List sql_result = session.createQuery("Select age from Persons age order where age.id = " + currentPersons.getId()).list();
List sql_result = session.createSQLQuery("Select * from PcComponents where component = " + PcComponent.getComponent()).list();

{% endhighlight %}

### 2. Secure usage of Hibernate
<b>Positional parameter in HQL</b>
{% highlight java %}
Query hql_query = session.createQuery("from PcComponents as component where component.id = ?");
List sql_result = hql_query.setString(1, "AMD Ryzen 5000").list();

{% endhighlight %}

	
<b>Named parameter in HQL</b>
{% highlight java %}
Query hql_query = session.createQuery("Select employee from Employees employee where employee.salary > :salary");
List sql_result = hql_query.setLong("salary", new Long(1000)).list();

{% endhighlight %}

	
<b>Named parameter list in HQL</b>
{% highlight java %}
List items = new ArrayList(); 
items.add("book"); items.add("clock"); items.add("ink");
List results = session.createQuery("from Cart as cart where cart.item in (:itemList)").setParameterList("itemList", items).list();

{% endhighlight %}


<b>JavaBean in HQL</b>
{% highlight java %}
Query hql_query = session.createQuery("from Persons as person where person.name = :person and person.surname = :surname");
List sql_result = hql_query.setProperties(javaBean).list(); //assumes javaBean has getName() & getSurname() methods.

{% endhighlight %}

	
<b>Native-SQL </b>
{% highlight java %}
Query sql_query = session.createSQLQuery("Select * from PcComponents where component = ?");
List sql_result = sql_query.setString(0, "AMD Ryzen 5000").list();

{% endhighlight %}


## PreparedStatement {#preparedstatement}
A PreparedStatement represents a precompiled SQL statement that can be executed multiple times without having to recompile for every execution.

### 1. Vulnerable usage of PreparedStatement
<b>Example 1:</b>
{% highlight java %}
String query = "SELECT * FROM Persons WHERE CNP ='"+ CNP + "'" + " AND FirstName='" + FirstName + "'";
Statement statement = connection.createStatement();
ResultSet response_query = statement.executeQuery(preparedstatement_query);

{% endhighlight %}

This code is vulnerable to SQL Injection because it uses dynamic queries to concatenate malicious data to the query itself. Notice that it uses the Statement class instead of the PreparedStatement class.

<b>Example2:</b>
{% highlight java %}
String query = "SELECT * FROM Persons WHERE CNP ='"+ CNP + "'" + " AND FirstName='" + FirstName + "'";
PreparedStatement statement = connection.prepareStatement(query);
ResultSet response_query = statement.executeQuery();

{% endhighlight %}

 Even though it uses the PreparedStatement class it is still creating the query dynamically via string concatenation.

### 2.Secure usage of PrepareStatement
For preveting SQLi we have to correctly use parameterized queries. By utilizing Java's PreparedStatement class, bind variables (the question marks) and the corresponding setString methods, SQL Injection can be easily prevented.

{% highlight java %}
PreparedStatement statement = connection.prepareStatement("SELECT * FROM Persons WHERE CNP=? AND FirstName=?");
statement.setString(1, CNP);
statement.setString(2, FirstName);
ResultSet result = statement.executeQuery();

{% endhighlight %}
