cassandra-driver-mapping
========================

Apache Cassandra (C*) never was easier.   
Enjoy this addon for the DataStax Java Driver which will bring you piece and serenity.     
The main goal is to enable JPA -like behaviour for entities to be persisted in C*.  
The module is not replacement for the DataStax Java Driver but time-saver addon to it.  
The module relies on DataStax Java Driver version 2.0 and JPA 2.1.  

Read more about [Cassandra, Datastax Driver and CQL3](http://www.datastax.com/documentation/gettingstarted/index.html).

### Table of Contents  
[Features](#features)  
[Jump Start](#start)  
[Various Mappings](#mapping)  
[Custom Queries](#queries)  
[Entity Metadata and Data Types](#metadata)  
[Spring Framework Example](#spring)  
[Coming Features](#comingfeatures)

<a name="features"/>
### Features

The features provided by the module include:

  - Manipulate Entity
  	* Get entity from Cassandra.
  	* Save entity to Cassandra.
	* Delete entity from Cassandra.
	* Run custom Queries built with datastax.QueryBuilder
	* Convert ResultSet into List of Entities

  - Generate Schema
  	* Create table from any Java Bean even without annotations. 
  	* Create tables and indexes from the JPA 2.1 annotated entities.
  	* Alter tables and indexes if entities definition has changed.
  	* Drop tables and indexes.

<a name="start"/>
### Jump Start


Install it in your application from Maven Central using the following dependency:
```
    <dependency>
      <groupId>com.valchkou.datastax</groupId>
      <artifactId>cassandra-driver-mapping</artifactId>
      <version>2.0.0-rc2</version>
    </dependency>
```
Create MappingSession instance:
```
	import com.datastax.driver.mapping.MappingSession;
    ...
    MappingSession mappingSession = new MappingSession(keyspace, session);
```    
You need to open the session and create the keyspace in prior to use MappingSession.  
If you are not familiar with procedure please refer to http://www.datastax.com/docs for Developers.  
Or look at the [Spring Framework Example](#spring) below.
 
Now you can play with your entity: 
```java
	Entity entity = new Entity();
    mappingSession.save(entity);
    
    entity = mappingSession.get(Entity.class, id);
    
    mappingSession.delete(entity);	
```
Very simple, isn't it? No mapping files, no scripts, no configuration files.   
You don't have to worry about creating the Table and Indexes for your Entity.  
All is built-in and taken care of. Entity definition will be automatically synchronized with C*. 

<a name="mapping"/>
### Various Mappings

	IMPORTANT!:   
	- All names are converted to lowercase.  
	- JPA annotations give you more features though are not required.  
	- If entity or field is not annotated it will provide its name as default.    
	- Id field is required for the entity.
	- Index annotation supported starting JPA 2.1.    
    - Index name must be unique within the keyspace.  
    - In C* you can have only one column per index.

   - Sample: Simple Bean
	```java
	public class Entity {
	
		private long Id;
		private String name;
		// public getters/setters ...
	}
	```
	Generates CQL3
	```
   		CREATE TABLE IF NOT EXISTS ks.entity (id bigint, name text,  PRIMARY KEY(id))
	``` 
	  
	  
   - Sample:JPA Entity
	```java
	@javax.persistence.Entity
	@javax.persistence.Table (name="mytable")
	public class Entity {
		
		@javax.persistence.Id
		private long Id;
		
		@javax.persistence.Column(name = "myname")
		private String name;
		// public getters/setters ...
	}
	```
	Generates CQL3
	```
   		CREATE TABLE IF NOT EXISTS ks.mytable (id bigint, myname text,  PRIMARY KEY(id))
	```     
	
	
   - Sample: JPA Entity with Indexes    
	```java
	@javax.persistence.Entity
	@javax.persistence.Table (name="mytable", 
	indexes = {
		@javax.persistence.Index(name="entity_email_idx", columnList="email" ), 
		@javax.persistence.Index(name="entity_name_idx", columnList="myname" ) 
	})
	public class Entity {
		
		@javax.persistence.Id
		private java.util.UUID code;
		
		@javax.persistence.Column(name = "myname")
		private String name;
		private String email;
		// public getters/setters ...
	}
	```
	Generates CQL3
	```
   		CREATE TABLE IF NOT EXISTS ks.mytable (code uuid, myname text, email text,  PRIMARY KEY(code)); 
		CREATE INDEX IF NOT EXISTS entity_email_idx ON ks.mytable(email);  
		CREATE INDEX IF NOT EXISTS entity_name_idx ON ks.mytable(myname);
	```   
   
   
   - Sample: Transient property
	```java
	@javax.persistence.Entity
	public class Entity {
		
		private java.util.UUID id;
		private String name;
		
		@Transient
		private BigDecimal calculable;
		
		// public getters/setters ...
	}
	```
	Generates CQL3
	```
   		CREATE TABLE IF NOT EXISTS ks.entity (id uuid, name text,  PRIMARY KEY(id))
	```     
   
   
   - Sample: Collections
   	```java
	import java.math.BigInteger;
	import java.util.Date;
	import java.util.List;
	import java.util.Map;
	import java.util.Set;
	
	import javax.persistence.Id;
	import javax.persistence.Table;
	
	@Table(name="entity")
	public class Entity {
		@Id
		private java.util.UUID id;
		private List<String> cats;
		private Set<Date> dogs;
		private Map<String, BigInteger> pets;
		
		// public getters/setters ...
	}
	```
	Generates CQL3
	```
   		CREATE TABLE IF NOT EXISTS ks.entity (id uuid, cats list<text>, dogs set<timestamp>, pets map<text, varint>,  PRIMARY KEY(id))
	```     
Collections must have generic type. Only java.util.List, Map and Set are allowed.  
Please refer Datastax CQL documentation [Use Collection] (http://www.datastax.com/documentation/cql/3.1/webhelp/index.html#cql/cql_using/use_collections_c.html#useCollections)

<a name="queries"/>
### Custom Queries

Datastax Driver shipped with a tool which allows us to build custom queries.  
You have 2 options to map your query on your Entity.  
- Option1: Build a query Statement and pass it to mappingSession.  
- Option2: Build and run the query with Datastax session and pass the Datastax ResultSet into mappingSession.  
The mappingSession will return you the result as List<Entity>.  
Highly recommended to use EntityTypeMetadata for retrieving table and column names when building Statements.
	```java
			// Option 1	
				EntityTypeMetadata emeta = EntityTypeParser.getEntityMetadata(Entity.class);
				EntityFieldMetaData fmeta = emeta.getFieldMetadata(field_name);
				Statement query = QueryBuilder
							.select().all()
							.from(sf.getKeyspace(), emeta.getTableName())
							.where(eq(fmeta.getColumnName(), value));
							
				List<Account> result = mappingSession.getByQuery(Entity.class, query);
	 ```

	```java
			// Option 2	
				EntityTypeMetadata emeta = EntityTypeParser.getEntityMetadata(Entity.class);
				EntityFieldMetaData fmeta = emeta.getFieldMetadata(field_name);
				Statement query = QueryBuilder
							.select().all()
							.from(sf.getKeyspace(), emeta.getTableName())
							.where(eq(fmeta.getColumnName(), value));
				
				ResultSet rs = session.execute(query);	
						
				List<Entity> result = mappingSession.getFromResultSet(Entity.class, rs);
				
	 ```
### Alter Behaviour

As your project is evolving you may want to refactor entity, add or delete properties and indexes.    
Please read to understand what will and will not be altered.
   
Not Alterable
   - add/delete/rename primary key columns.
   - change column data type to incompatible one, such as string to number.
   - change property name which is not annotated as @Column. This will be understood as a new property.
   	
Alterable
   - add new property.
   - delete property.
   - add index on column.
   - change datatype to compatible one. Compatibility is enforced by C*.	
   		
<a name="metadata"/>
### Entity Metadata and Data Types
   
You may want to access Entity metadata if you are building custom Statements.    
Entity Metadata contains corresponding table and column names.  
Entity Metadata can be easily accessed anywhere in your code as:
```java	
	EntityTypeMetadata emeta = EntityTypeParser.getEntityMetadata(Entity.class)
	emeta.getTableName(); // corresponding table name in C*
	
	// get field meta info by property name
	EntityFieldMetaData fdata = emeta.getFieldMetadata("email");
	 
	// corresponding column name in C*
	String columnName = fdata.getColumnName(); 
	
	 // all the persistent fields on entity
	List<EntityFieldMetaData> fields = emeta.getFields();
```	
Datastax driver has mapping of datastax types to java. But not all types are mapped as 1-to-1.  
[CQL3 data types to Java types](http://www.datastax.com/documentation/developer/java-driver/1.0/webhelp/index.html#java-driver/reference/javaClass2Cql3Datatypes_r.html)  
In order the mapping to work the module defines backward mapping for the types.  

Java type | CQL3 data type
--- | ---
int|int
long|bigint
float|float
double|double
boolean|boolean
java.lang.Double|double
java.nio.ByteBuffer|blob
java.math.BigDecimal|decimal
java.lang.String|text
java.util.Date|timestamp
java.lang.Boolean|boolean
java.lang.Integer|int
java.lang.Long|bigint
java.util.Map|map
java.lang.Float|float
java.util.Set|set
java.math.BigInteger|varint
java.util.UUID|uuid
java.util.List|list

You can override defaults as:
```java
	import com.datastax.driver.core.DataType;
	...
	Map<Class<?>, DataType.Name> mapping = new HashMap<Class<?>, DataType.Name>();
	.... populate the map
	EntityTypeParser.setDataTypeMapping(mapping);
```
Or override individual type:
```java
	import com.datastax.driver.core.DataType;
	...
	EntityTypeParser.overrideDataTypeMapping(javaClass, DataType.Name)
```

<a name="spring"/>
### Spring Framework Example 

- Configure properties.  
Let's imagine we have a property file /META-INF/cassandra.properties:
  ```
   	cassandra.keyspace=your_keyspace
	cassandra.node=127.0.0.1
   ```
   
- Include properties in spring config:
   ```xml
    <?xml version="1.0" encoding="UTF-8"?>
	<beans:beans xmlns="http://www.springframework.org/schema/mvc"
		xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
		xmlns:beans="http://www.springframework.org/schema/beans"
		xmlns:context="http://www.springframework.org/schema/context"
		xsi:schemaLocation="http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsd
		http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
		http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd">

		
		<!-- Enables the Spring MVC @Controller programming model -->
		<annotation-driven />

		<context:property-placeholder location="classpath*:/META-INF/cassandra.properties"/>  
	
		<context:component-scan base-package="your.package.path" />
	 </beans:beans> 
   ```

- Define Entity
	```java
		import java.util.UUID;
		
		import javax.persistence.Entity;
		import javax.persistence.Column;
		import javax.persistence.Id;
		import javax.persistence.Table;
		
		@Entity
		@Table(name="account", indexes = {@Index(name="account_email_idx", columnList="email" )})
		public class Account {
			@Id
			private String id = UUID.randomUUID().toString();
			
			@Column(name="email") 
			private String email;			
			
			public String getId() {
				return id;
			}
			public void setId(String id) {
				this.id = id;
			}			
			public String getEmail() {
				return email;
			}
			public void setEmail(String email) {
				this.email = email;
			}
		}
	```
		   
- Create session factory for C*:
	
	```java
		import org.springframework.beans.factory.annotation.Value;
		import org.springframework.stereotype.Repository;
		import com.datastax.driver.core.Cluster;
		import com.datastax.driver.core.Session;
		import com.datastax.driver.mapping.MappingSession;
		import com.datastax.driver.mapping.schemasync.SchemaSync;
		
		@Repository
		public class CassandraSessionFactory {
			
			@Value("${cassandra.keyspace}")
			private String keyspace;
			
			@Value("${cassandra.node}")
			private String node;
			
			private Cluster cluster;
			private Session session;
			private MappingSession mappingSession;
				
			public Session getSession() {
				if (session == null) {
					connect();
				}
				return session;
			}
		
			public MappingSession getMappingSession() {
				if (session == null) {
					connect();
				}
				return mappingSession;
			}
		
			public String getKeyspace() {
				return keyspace;
			}
			
			/** only 1 thread is permitted to open connection */
			protected synchronized void connect() {
				if (session == null) {
					cluster = Cluster.builder().addContactPoint(node).build();
					session = cluster.connect();
					session.execute("CREATE KEYSPACE IF NOT EXISTS "+ getKeyspace() +
						" WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 3 }");
						
					mappingSession = new MappingSession(getKeyspace(), getSession());
				}	
			}
		}
	```
	
- Create DAO and inject factory into it:
		
	```java		
		import java.util.List;
		
		import org.springframework.beans.factory.annotation.Autowired;
		import org.springframework.stereotype.Repository;
		
		import static com.datastax.driver.core.querybuilder.QueryBuilder.eq;
		import com.datastax.driver.core.Statement;
		import com.datastax.driver.core.querybuilder.QueryBuilder;
		import com.datastax.driver.mapping.EntityFieldMetaData;
		import com.datastax.driver.mapping.EntityTypeMetadata;
		import com.datastax.driver.mapping.EntityTypeParser;
		import com.datastax.driver.mapping.MappingSession;	
		
		@Repository
		public class AccountDAO {
			
			@Autowired
			CassandraSessionFactory sf;
			
			public void save(Account account) {
				sf.getMappingSession().save(account);	
			}
			
			public void delete(Account account) {
				sf.getMappingSession().delete(account);	
			}
			
			public Account getById(Object id) {
				return sf.getMappingSession().get(ChatAccount.class, id);	
			}
			
			public Account getByEmail(String email) throws Exception {
				
				Statement stmt = buildQueryForColumn("email", email);
				if (stmt==null) return null;
				
				List<Account> result = sf.getMappingSession().getByQuery(Account.class, stmt);
				if (result == null || result.size()==0) return null;
		
				return result.get(0);
			}
		
			
			/** Sample Building Select Statement for a single column with Datastax QueryBuilder */
			protected Statement buildQueryForColumn(String propName, Object propVal) {
				EntityTypeMetadata emeta = EntityTypeParser.getEntityMetadata(Account.class);
				EntityFieldMetaData fmeta = emeta.getFieldMetadata(propName);
				if (fmeta != null) {
					return QueryBuilder
							.select().all()
							.from(sf.getKeyspace(), emeta.getTableName())
							.where(eq(fmeta.getColumnName(), propVal));
				}
				return null;
			}
		}
	```
	
<a name="comingfeatures"/>
### Upcoming Features

   - Support composite Primary Keys
   - Support TTL and Timestamp
   - Enable optimistic lock for Entities (TBD)
   - Support options for Create Table 
	

