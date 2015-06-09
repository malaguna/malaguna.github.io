---
layout: post
title: "Liquifying your project"
date: 2015-06-09 23:29:51 +0200
comments: true
categories: java hibernate liquibase maven spring 
---

Sometimes, when you use Hibernate in a project, you rely on *hbm2ddl* utility, it is fast and seems a very nice feature to manage database. However, when the project starts growing up it comes more difficult to maintain. I suffered it recently in a medium size project (150 KLoc) and thus I decided to search for a better option.

After some research, I found **Liquibase** and now I am convinced it is the right way. In this post I am going to tell you how I use **Liquibase**.

<!-- more -->

Of course, I don't want to teach you all about Liquibase, [www.liquibase.org](http://www.liquibase.org/) is plenty of documentation. Instead, I tell you what I think Liquibase is good for, and how it saved my day.

Liquibase was designed to define and track DDL evolution independently of the RDBMS, so it is a database agnostic way to define and manage your database model. It has its own language to express **ChangeSet**. Liquibase groups **ChangeSet** into **ChangeLog** files. **ChangeLog** allows you to track database evolution. It is right to think about Liquibase as if it were a kind of CVS for databases.

Liquibase is also capable to obtain differences between two different databases, between a **ChangeLog** and a database, even between Hibernate Mappings and database!. Liquibase expresses differences as a set of **ChangeSet** that you can add to the **ChangeLog**.

## Sample project

I create a very simple project, called **Casiopea** to show you how I am using Liquibase. You can find this project as a github repo [Casiopea Project](https://github.com/malaguna/casiopea).

This project has no functional code, it only has minimal Maven, Spring, Hibernate and Liquibase configuration. It also has three domain entities, as depicted below, with its Hibernate mappings (xml mappings, not annotations).

{% img center /images/liquibase-casiopea/model-casiopea.png %}

As you can see in the class diagram, there are three entities: **Person**, **Project** and **Role** related through a **MemberShip** relationship entity. This model let me know the role every person plays in the projects they work. As said, it is only an example to show how to use Liquibase.

## What I want from Liquibase?

I don't want to waste my time, so I want to obtain SQL-DDL scripts from Hibernate mappings. Of course *hbm2ddl* is straight forward, but it doesn't serve to compare databases nor create SQL migration scripts for production systems.

So I want Liquibase to create SQL-DDL scripts for several RDBMS, compare databases, update databases and create migration scripts. To be more precise, I want Liquibase in my project to work as following:

{% img center /images/liquibase-casiopea/liquibase-diagram.png %}

1. `mvn compile liquibase:diff`: This Maven command compares Hibernate mappings against database and generates a **ChangeSet** that *moves* database to what Hibernate mappings expects.
2. The automatically generated **ChangeSet** must be revised by a developer so it can be included into the Liquibase **ChangeLog** file.
3. After **ChangeLog** modification, it is possible to update database through `mvn copmile liquibase:update`.

It is worth to say that manual revision of the second step is very important. It is very useful to adjust some things as foreign key names, database dependent stuff, data types size, and so on. Of course, it is possible to obtain SQL script with `liquibase:updateSQL` command.

## Project configuration

If you want to leverage your project with Liquibase, you have to do following configuration.

### Spring

Casiopea Project has an Spring application context descriptor (`src/main/resources/app-context.xml`) with two beans: 

 * `DataSource`: which points to database.
 * `SessionFactory`: which holds Hibernate mappings.

```xml

	<bean id="dataSource" destroy-method="close"
		class="org.apache.commons.dbcp.BasicDataSource">

		...

		<!-- Settings de conexion -->
		<property name="url" value="${hibernate.url}" />
		<property name="username" value="${hibernate.username}" />
		<property name="password" value="${hibernate.password}" />
	</bean>

	<bean	id="sessionFactory"
		class="org.springframework.orm.hibernate4.LocalSessionFactoryBean">
		<property name="dataSource" ref="dataSource" />

		<property name="mappingLocations">
			<list>
				<value>classpath:/org/malaguna/casiopea/model/Person.hbm.xml</value>
				<value>classpath:/org/malaguna/casiopea/model/Project.hbm.xml</value>
				<value>classpath:/org/malaguna/casiopea/model/MemberShip.hbm.xml</value>
			</list>
		</property>

		...
```

As you can see, lines 8-10 configure database url, username and password using Maven properties. Lines 18-20, point to mappings classpath location. It is important to define `mappingLocations` property of `LocalSessionFactoryBean` instead `mappingResources` property. The second it is not recognized by liquibase-hiberante plugin.

### Liquibase

Liquibase is configured by means of `liquibase.properties` file in resources folder. Here you can find relevant lines:

```properties
changeLogFile: src/main/resources/liquibase/changelog.xml
diffChangeLogFile: src/main/resources/liquibase/autolog-${project.version}-${user.name}-${timestamp}.xml
driver: ${hibernate.driver}
url: ${hibernate.url}
username: ${hibernate.username}
password: ${hibernate.password}
referenceUrl: hibernate:spring:app-context.xml?bean=sessionFactory
```
Lines 3-6, define database connection as in Spring application context, that is, using Maven properties. 

Line 2, defines `diffChangeLogFile` property. This is used when Liquibase calculates differences between mappings and database. Liquibase will write diff in a unique file named from project version, developer name and timestamp maven compilation.

### Maven POM

Here it is the main configuration. There are three main points:

1. Maven profiles, to define database connection properties
2. Liquibase plugin, to configure Liquibase execution and dependencies.
3. Resource filtering, to replace maven variables in Spring, Liquibase and other configuration properties.

#### Maven profiles

Following is a fragment of the Maven POM file, where a profile called `db-local` is defined:

```xml
	<!-- Debug config profiles -->
	<profile>
		<id>db-local</id>
		<activation>
			<activeByDefault>true</activeByDefault>
		</activation>
		<properties>
			<!-- Local HSQLDB server database -->
			<hibernate.dialect>org.hibernate.dialect.HSQLDialect</hibernate.dialect>
			<hibernate.driver>org.hsqldb.jdbcDriver</hibernate.driver>
			<hibernate.url>jdbc:hsqldb:hsql://localhost/casiopea</hibernate.url>
			<hibernate.username>sa</hibernate.username>
			<hibernate.password></hibernate.password>
		</properties>
	</profile>
```

In this profile, Maven points to a local HsqlDB instance called `casiopea`. You can easily change these values, or you can define new maven profiles to add more database connections.

If you want to start a local HsqlDB instance called `casiopea` you can download HsqlDB jar and run the following command:

```bash
$> java -cp /path/to/hsqldb-2.3.2.jar org.hsqldb.Server -database.0 file:/path/to/casiopea -dbname.0 casiopea

```

#### Liquibase plugin

Following is a fragment of the Maven POM file, where Liquibase plugin is configured:


```xml
	<plugin>
		<!-- liquibase management -->
		<groupId>org.liquibase</groupId>
		<artifactId>liquibase-maven-plugin</artifactId>
		<version>${liquibase.version}</version>
		<configuration>
				    <propertyFile>target/classes/liquibase.properties</propertyFile>
		</configuration>
		<dependencies>
			...
		</dependencies>
	</plugin>
```

Line 7, shows that `liquibase.properties` is not read from resources path but from target. This is because of the Maven filtering process.

#### Resource filtering

Resource filtering is configured in maven as usual, including the files that contain Maven properties. You can check it at the end of the Maven pom, inside build section.

## Running the project

Before running this project it is very important to say that it won't work. There is a bug with the Liquibase Hibernate plugin that ignores Spring SessionFactory bean, when using `org.springframework.orm.hibernate4.LocalSessionFactory`.

I documented this bug as [#61](https://github.com/liquibase/liquibase-hibernate/issues/61). Until this bug won't be resolved, Casiopea project does not work properly. Instead you could use a forked version of Liquibase Hibernate plugin where I solved this bug.

So, if you still want to use Casiopea project, you can clone my [liquibase-hibernate fork](https://github.com/malaguna/liquibase-hibernate) and install it localy using `mvn install`. This command will deploy in your local maven repository the 3.6-SNAPSHOT version that you need.

Once installed this dependecy, you can start a local instance of HsqlDB as explained above, and run `liquibase:diff`. You will find a message similar to this:

```
mall02@tk-x220-t:~/Repos/casiopea$ mvn compile liquibase:diff
[INFO] Scanning for projects...
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building Casiopea 0.0.1-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO] 

...

INFO 9/06/15 23:12: liquibase-hibernate: Reading hibernate configuration hibernate:spring:app-context.xml?bean=sessionFactory

...

INFO 9/06/15 23:12: liquibase-hibernate: Found mappingLocation classpath:/org/malaguna/casiopea/model/Person.hbm.xml
INFO 9/06/15 23:12: liquibase-hibernate: Adding resource  file:/home/mall02/Repos/casiopea/target/classes/org/malaguna/casiopea/model/Person.hbm.xml
INFO 9/06/15 23:12: liquibase-hibernate: Found mappingLocation classpath:/org/malaguna/casiopea/model/Project.hbm.xml
INFO 9/06/15 23:12: liquibase-hibernate: Adding resource  file:/home/mall02/Repos/casiopea/target/classes/org/malaguna/casiopea/model/Project.hbm.xml
INFO 9/06/15 23:12: liquibase-hibernate: Found mappingLocation classpath:/org/malaguna/casiopea/model/MemberShip.hbm.xml
INFO 9/06/15 23:12: liquibase-hibernate: Adding resource  file:/home/mall02/Repos/casiopea/target/classes/org/malaguna/casiopea/model/MemberShip.hbm.xml

...

[INFO] Differences written to Change Log File, src/main/resources/liquibase/autolog-0.0.1-SNAPSHOT-mall02-20150609231229.xml
[INFO] ------------------------------------------------------------------------
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 3.494s
[INFO] Finished at: Tue Jun 09 23:12:32 CEST 2015
[INFO] Final Memory: 17M/212M
[INFO] ------------------------------------------------------------------------
```

Line 11, tells you Liquibase Hibernate plugin has found Spring application context and `sessionFactory` bean.

Lines 15-20, list all mapped entities that are defined in `sessionFactory`. Finally, line 24, shows where Liquibase has written automated **ChangeSet**. You can edit this file and found how good is Liquibase ;)

Now you can review the autolog and copy all ChangeSets to ChangeLog file. After that, you can run `liquibase:update` or `liquibase:updateSQL` and smile for a while.

I hope you enjoy reading this post, and obviously I would like you find it helpful.

Thank you for reading!
