If owners have vehicles, do you want to write your database connectivity like this?

```java
package cloud.stephen.springdatajdbc;

import java.util.List;

import org.springframework.data.jdbc.repository.query.Query;
import org.springframework.data.repository.PagingAndSortingRepository;

public interface OwnerRepository extends PagingAndSortingRepository<Owner, Long> {
    List<Owner> findByFirstName(String firstName);
    List<Owner> findByLastName(String lastName);

    @Query("SELECT OWNER.ID, OWNER.FIRST_NAME, OWNER.LAST_NAME FROM OWNER JOIN VEHICLE ON OWNER.ID = VEHICLE.OWNER WHERE VEHICLE.MAKE = :make")
    List<Owner> findByVehicleMake(String make);
}
```

That generates SQL for (only the trivial) CRUD operations like this

```sql
SELECT "OWNER"."ID" AS "ID", "OWNER"."LAST_NAME" AS "LAST_NAME", "OWNER"."FIRST_NAME" AS "FIRST_NAME" FROM "OWNER" WHERE "OWNER"."ID" = ?
```

Which you can access like this

```shell
curl "http://localhost:8080/owners/search/findByVehicleMake?make=VW"
```

To return

```json
{
  "_embedded" : {
    "owners" : [ {
      "firstName" : "Stephen",
      "lastName" : "Harrison",
      "vehicles" : [ {
        "make" : "Honda",
        "model" : "CR-V",
        "mileage" : 25000,
        "owner" : 1
      }, {
        "make" : "VW",
        "model" : "GTI",
        "mileage" : 30000,
        "owner" : 1
      } ],
      "_links" : {
        "self" : {
          "href" : "http://localhost:8080/owners/1"
        },
        "owner" : {
          "href" : "http://localhost:8080/owners/1"
        }
      }
    } ]
  },
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/owners/search/findByVehicleMake?make=VW"
    }
  }
}
```

You do? Cool. Then this is for you. Read on.

## A Cook's tour of Java persistence

Let's face it, we've been through a lot of tech around Java persistence frameworks. Those technologies have their place, for sure. And we use them a lot at XYZ.

The first version of [JDBC](https://docs.oracle.com/javase/8/docs/technotes/guides/jdbc/) came along in 1997 to abstract away the vagaries of calling the Oracle driver directly. Up to that point there was no connection pooling and it could take 1-2 seconds just to open a client to the database from a Java server. [FastCGI](https://en.wikipedia.org/wiki/FastCGI) was developed at Open Market (where the author worked at the time) to cache connections. A huge win.

1997 was a banner year for Java things. [Enterprise Java Beans](https://docs.oracle.com/javaee/6/tutorial/doc/gijsz.html) came along and promised to reduce the database load considerably among other benefits. It persisted state to local disk saving memory. It kind of worked at the expense of quite a bit of complexity. Early versions were super fragile. 

Along came [Hibernate](https://hibernate.org/) in 2001 as an alternative to EJB. It maps objects to datasets with automating for dirty reads, lazy loading. XYZ uses a lot of Hibernate. Or actually [Java Persistence API (JPA)](https://docs.oracle.com/javaee/6/tutorial/doc/bnbpz.html). Modern Hibernate is one of several JPA providers.

## Issues with JPA

When it works, it works well. It can generate SQL for all the databases. Lazy-loading of dependent objects can make memory usage and database calls efficient.

But in the author's experience it too-often doesn't work well. Just one of very many questions like [stackoverflow.com](https://stackoverflow.com/questions/10930870/jpa-lazyloading-fails-in-production-mode-in-testmode-it-works-fine) asks why this, for example.

```
org.hibernate.LazyInitializationException: failed to lazily initialize a collection of role: de.hoeso.gwt.platform.server.domain.common.Person.anschrift, no session or session was closed
    at org.hibernate.collection.AbstractPersistentCollection.throwLazyInitializationException(AbstractPersistentCollection.java:383)
    at org.hibernate.collection.AbstractPersistentCollection.throwLazyInitializationExceptionIfNotConnected(AbstractPersistentCollection.java:375)
    at org.hibernate.collection.AbstractPersistentCollection.readSize(AbstractPersistentCollection.java:122)
    at org.hibernate.collection.PersistentBag.size(PersistentBag.java:248)
    at de.hoeso.sis.server.services.common.impl.UserServiceBeanImpl.login(UserServiceBeanImpl.java:397)
    at de.hoeso.sis.server.rpc.LoginService.execute(LoginService.java:35)
```

Other people are much better at JPA than me and don't get the above so much.

JPA requires session state on thread-local storage for many operations, which is anathema to modern [reactive](https://www.reactivemanifesto.org/) architectures.

## So what's an alternate?

There's nothing wrong with JDBC. (Ignoring the fact column indices start at 1, a gift from Java ex-CTO and still the author's friend regardless, Graham Hamilton. You monster Graham. You monster!)

So let's leverage all JDBC's got to offer.

### Take a look at Spring Data JDBC and Spring Data Rest

Along with many others, the author loves [Spring Boot](https://spring.io/projects/spring-boot), so we're going to use it with [Spring Data JDBC](https://spring.io/projects/spring-data-jdbc).

!!! note
    __Please make sure you understand: There is no JPA here at all! This Spring Data JDBC, not Spring Data JPA.__

### Here's a complete application

All Spring Boot applications look the same.

#### `Application.java`

```java
package cloud.stephen.springdatajdbc;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.jdbc.repository.config.EnableJdbcRepositories;

@SpringBootApplication
@EnableJdbcRepositories
public class Application {
	public static void main(String[] args) {
		SpringApplication.run(Application.class, args);
	}
}
```

#### `Owner.java`

Simply an object with an `id`. This corresponds to the SQL

```sql
CREATE TABLE IF NOT EXISTS OWNER (
    ID INT AUTO_INCREMENT  PRIMARY KEY,
    FIRST_NAME VARCHAR(50) NOT NULL, 
    LAST_NAME VARCHAR(50) NOT NULL 
);
```

```java
package cloud.stephen.springdatajdbc;

import java.util.Set;

import org.springframework.data.annotation.Id;

public class Owner {
  @Id
  private final Long id;
  private final String firstName;
  private final String lastName;

  public Owner(final Long id, final String firstName, final String lastName) {
    this.id = id;
    this.firstName = firstName;
    this.lastName = lastName;
  }

  static Owner of(final Long id, final String firstName, final String lastName) {
    return new Owner(id, firstName, lastName);
  }

  Owner withId(Long id) {
    return of(id, firstName, lastName);
  }

  public Long getId() {
    return id;
  }

  public String getFirstName() {
    return firstName;
  }

  public String getLastName() {
    return lastName;
  }
}
```

!!! note
    This is actually a pretty nice object pattern. Spring Data takes full advantage of classes written this way.

#### `OwnerRepository.java`

This simply provides types for a generic interface.

```java
package cloud.stephen.springdatajdbc;

import org.springframework.data.repository.CrudRepository;

public interface OwnerRepository extends CrudRepository<Owner, Long> {
}
```

Wait what? Where's the implementation? This is just an interface.

Yes. The Spring Framework leverages Aspect Oriented Programming to create runtime proxies that either decorate concrete methods, or in this case implement interface methods. This provides fundamentals for Spring Boot, which takes it and runs with it. Fast. So all the interface methods in [`CrudRepository`](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/repository/CrudRepository.html) are proxied with clever code.

#### Maven `pom.xml` (snippet)

```xml
...
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jdbc</artifactId>
		</dependency>
		<dependency>
			<groupId>javax.persistence</groupId>
			<artifactId>javax.persistence-api</artifactId>
			<version>2.2</version>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-rest</artifactId>
		</dependency>
...
```

## Let's manipulate some data

Because we included the dependency `spring-boot-starter-data-rest` we get a REST interface for our data. For free. No code — let alone configuration — required. 

The exported REST interface uses [Spring HATEOAS](https://docs.spring.io/spring-hateoas/docs/current/reference/html/), which some people pronounce "hate-oas" when they either don't understand it, or do but don't like it. 

### List owners

Basic HATEOAS REST calls just name a model.

```shell
$ curl -i "http://localhost:8080/owners"
HTTP/1.1 200
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Content-Type: application/hal+json
Transfer-Encoding: chunked
Date: Sun, 24 May 2020 14:44:46 GMT

{
  "_embedded" : {
    "owners" : [ ]
  },
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/owners"
    },
    "profile" : {
      "href" : "http://localhost:8080/profile/owners"
    }
  }
}
```

Empty.

### Add an owner

```shell
$ curl -i -X POST -H "Content-Type:application/json" -d '{  "firstName" : "Alex", "lastName" : "Harrison" }' http://localhost:8080/owners
HTTP/1.1 201
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Location: http://localhost:8080/owners/1
Content-Type: application/hal+json
Transfer-Encoding: chunked
Date: Sun, 24 May 2020 15:26:42 GMT

{
  "firstName" : "Alex",
  "lastName" : "Harrison",
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/owners/1"
    },
    "owner" : {
      "href" : "http://localhost:8080/owners/1"
    }
  }
}
```

Cool. Looks like the `id` is `1`. 

### Add another

```shell
$ curl -i -X POST -H "Content-Type:application/json" -d '{  "firstName" : "Stephen", "lastName" : "Harrison" }' http://localhost:8080/owners
HTTP/1.1 201
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Location: http://localhost:8080/owners/2
Content-Type: application/hal+json
Transfer-Encoding: chunked
Date: Sun, 24 May 2020 15:26:52 GMT

{
  "firstName" : "Stephen",
  "lastName" : "Harrison",
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/owners/2"
    },
    "owner" : {
      "href" : "http://localhost:8080/owners/2"
    }
  }
}
```

With an `id` of `2`. Looks like Spring Data JDBC has created a sequence for ids.

### List them out

```shell
$ curl -i "http://localhost:8080/owners"
HTTP/1.1 200
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Content-Type: application/hal+json
Transfer-Encoding: chunked
Date: Sun, 24 May 2020 15:30:12 GMT

{
  "_embedded" : {
    "owners" : [ {
      "firstName" : "Alex",
      "lastName" : "Harrison",
      "_links" : {
        "self" : {
          "href" : "http://localhost:8080/owners/1"
        },
        "owner" : {
          "href" : "http://localhost:8080/owners/1"
        }
      }
    }, {
      "firstName" : "Stephen",
      "lastName" : "Harrison",
      "_links" : {
        "self" : {
          "href" : "http://localhost:8080/owners/2"
        },
        "owner" : {
          "href" : "http://localhost:8080/owners/2"
        }
      }
    } ]
  },
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/owners"
    },
    "profile" : {
      "href" : "http://localhost:8080/profile/owners"
    }
  }
}
```

### Delete someone (me 😿)

```shell
$ curl -i -X DELETE -H "Content-Type:application/json" http://localhost:8080/owners/2
HTTP/1.1 404
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Content-Length: 0
Date: Sun, 24 May 2020 15:35:45 GMT
```

### List them out again

```shell
$ curl -i "http://localhost:8080/owners"
HTTP/1.1 200
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Content-Type: application/hal+json
Transfer-Encoding: chunked
Date: Sun, 24 May 2020 15:36:13 GMT

{
  "_embedded" : {
    "owners" : [ {
      "firstName" : "Alex",
      "lastName" : "Harrison",
      "_links" : {
        "self" : {
          "href" : "http://localhost:8080/owners/1"
        },
        "owner" : {
          "href" : "http://localhost:8080/owners/1"
        }
      }
    } ]
  },
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/owners"
    },
    "profile" : {
      "href" : "http://localhost:8080/profile/owners"
    }
  }
}
```

### Update someone

```shell
$ curl -i -X PATCH -H "Content-Type:application/json" -d '{  "firstName" : "Alexander" }' http://localhost:8080/owners/1
HTTP/1.1 200
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Content-Type: application/hal+json
Transfer-Encoding: chunked
Date: Sun, 24 May 2020 15:37:40 GMT

{
  "firstName" : "Alexander",
  "lastName" : "Harrison",
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/owners/1"
    },
    "owner" : {
      "href" : "http://localhost:8080/owners/1"
    }
  }
}
```

Notice how we only specify `firstName` in the body of the `PATCH`. 

### Add some custom finders

```java
package cloud.stephen.springdatajdbc;

import java.util.List;

import org.springframework.data.repository.CrudRepository;

public interface OwnerRepository extends CrudRepository<Owner, Long> {
    List<Owner> findByFirstName(String firstName);
    List<Owner> findByLastName(String lastName);
}
```

Then we can say

```shell
$ curl -i "http://localhost:8080/owners/search/findByFirstName?firstName=Alex"
HTTP/1.1 200
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Content-Type: application/hal+json
Transfer-Encoding: chunked
Date: Sun, 24 May 2020 15:45:52 GMT

{
  "_embedded" : {
    "owners" : [ ]
  },
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/owners/search/findByFirstName?firstName=Alex"
    }
  }
}
```

Empty.

```shell
$ curl -i "http://localhost:8080/owners/search/findByFirstName?firstName=Alexander"
HTTP/1.1 200
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Content-Type: application/hal+json
Transfer-Encoding: chunked
Date: Sun, 24 May 2020 15:46:58 GMT

{
  "_embedded" : {
    "owners" : [ {
      "firstName" : "Alexander",
      "lastName" : "Harrison",
      "_links" : {
        "self" : {
          "href" : "http://localhost:8080/owners/1"
        },
        "owner" : {
          "href" : "http://localhost:8080/owners/1"
        }
      }
    } ]
  },
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/owners/search/findByFirstName?firstName=Alexander"
    }
  }
}
```

Bingo.

## Add a related table

That was dead-simple CRUD that we did close to zero work to create. It's also of zero use to us.

In any real-world database there are foreign keys, collections, one-to-many, many-to-many, and a lot of other things. Let's see how Spring Data JDBC and Spring Data REST handle that.

One-to-many and foreign keys are Java `Set`s. So let's do that.

### Create `Vehicle.java`

This corresponds to the database table

```sql
CREATE TABLE IF NOT EXISTS VEHICLE (
    ID INT AUTO_INCREMENT  PRIMARY KEY,
    MAKE VARCHAR(50) NOT NULL, 
    MODEL VARCHAR(50) NOT NULL ,
    MILEAGE INT NOT NULL,
    OWNER INT,
    FOREIGN KEY (OWNER) REFERENCES OWNER(ID)
);
```

```java
package cloud.stephen.springdatajdbc;

import org.springframework.data.annotation.Id;

public class Vehicle {
  @Id
  private final Long id;
  private final String make;
  private final String model;
  private final long mileage;
  private final Long owner;

  public Vehicle(final Long id, final String make, final String model, final long mileage, final Long owner) {
    this.id = id;
    this.make = make;
    this.model = model;
    this.mileage = mileage;
    this.owner = owner;
  }

  static Vehicle of(final Long id, final String make, final String model, final long mileage, final Long owner) {
    return new Vehicle(id, make, model, mileage, owner);
  }

  Vehicle withId(Long id) {
    return of(id, make, model, mileage, owner);
  }

  public Long getId() {
    return id;
  }

  public String getMake() {
    return make;
  }

  public String getModel() {
    return model;
  }

  public long getMileage() {
    return mileage;
  }

  public Long getOwner() {
    return owner;
  }
}
```

### `VehicleRepository.java`

```java
package cloud.stephen.springdatajdbc;

import org.springframework.data.repository.PagingAndSortingRepository;

public interface VehicleRepository extends PagingAndSortingRepository<Vehicle, Long> {
}
```

### Add vehicles to `Owner.java`

```java
package cloud.stephen.springdatajdbc;

import java.util.Set;

import org.springframework.data.annotation.Id;

public class Owner {
  @Id
  private final Long id;
  private final String firstName;
  private final String lastName;
  private final Set<Vehicle> vehicles;

  public Owner(final Long id, final String firstName, final String lastName, final Set<Vehicle> vehicles) {
    this.id = id;
    this.firstName = firstName;
    this.lastName = lastName;
    this.vehicles = vehicles;
  }

  static Owner of(final Long id, final String firstName, final String lastName, final Set<Vehicle> vehicles) {
    return new Owner(id, firstName, lastName, vehicles);
  }

  Owner withId(Long id) {
    return of(id, firstName, lastName, vehicles);
  }

  public Long getId() {
    return id;
  }

  public String getFirstName() {
    return firstName;
  }

  public String getLastName() {
    return lastName;
  }

  public Set<Vehicle> getVehicles() {
    return vehicles;
  }
}
```

### Manipulate vehicles

He's 18! So buy Alexander a car.

```shell
$ curl -i -X POST -H "Content-Type:application/json" -d '{  "make" : "VW", "model" : "Jetta", "mileage": 42000, "owner": 1 }' http://localhost:8080/vehicles
HTTP/1.1 201
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Location: http://localhost:8080/vehicles/1
Content-Type: application/hal+json
Transfer-Encoding: chunked
Date: Sun, 24 May 2020 16:09:20 GMT

{
  "make" : "VW",
  "model" : "Jetta",
  "mileage" : 42000,
  "owner" : 1,
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/vehicles/1"
    },
    "vehicle" : {
      "href" : "http://localhost:8080/vehicles/1"
    }
  }
}
```

Prove he owns it.

```shell
$ curl -i "http://localhost:8080/owners/1"
HTTP/1.1 200
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Content-Type: application/hal+json
Transfer-Encoding: chunked
Date: Sun, 24 May 2020 16:13:46 GMT

{
  "firstName" : "Alexander",
  "lastName" : "Harrison",
  "vehicles" : [ {
    "make" : "VW",
    "model" : "Jetta",
    "mileage" : 42000,
    "owner" : 1
  } ],
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/owners/1"
    },
    "owner" : {
      "href" : "http://localhost:8080/owners/1"
    }
  }
}
```

Cool. Let's buy me a car.

```shell
$ curl -i -X POST -H "Content-Type:application/json" -d '{  "make" : "Bugatti", "model" : "Veyron", "mileage": 10, "owner": 2 }' http://localhost:8080/vehicles
HTTP/1.1 201
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Location: http://localhost:8080/vehicles/3
Content-Type: application/hal+json
Transfer-Encoding: chunked
Date: Sun, 24 May 2020 16:15:47 GMT

{
  "make" : "Bugatti",
  "model" : "Veyron",
  "mileage" : 10,
  "owner" : 2,
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/vehicles/3"
    },
    "vehicle" : {
      "href" : "http://localhost:8080/vehicles/3"
    }
  }
}
```

### Another custom finder

Now we have a one-to-many relation, let's add a `JOIN` with a `WHERE` clause to `OwnerRepository.java`. We'll add a fancier base interface `PagingAndSortingRepository` for some more useful database calls at the same time.

```java
package cloud.stephen.springdatajdbc;

import java.util.List;

import org.springframework.data.jdbc.repository.query.Query;
import org.springframework.data.repository.PagingAndSortingRepository;

public interface OwnerRepository extends PagingAndSortingRepository<Owner, Long> {
    List<Owner> findByFirstName(String firstName);
    List<Owner> findByLastName(String lastName);

    @Query("SELECT OWNER.ID, OWNER.FIRST_NAME, OWNER.LAST_NAME FROM OWNER JOIN VEHICLE ON OWNER.ID = VEHICLE.OWNER WHERE VEHICLE.MAKE = :make")
    List<Owner> findByVehicleMake(String make);
}
```

We can see the custom finders like this

```shell
$ curl -i "http://localhost:8080/owners/search"
HTTP/1.1 200
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Content-Type: application/hal+json
Transfer-Encoding: chunked
Date: Sun, 24 May 2020 16:26:32 GMT

{
  "_links" : {
    "findByLastName" : {
      "href" : "http://localhost:8080/owners/search/findByLastName{?lastName}",
      "templated" : true
    },
    "findByVehicleMake" : {
      "href" : "http://localhost:8080/owners/search/findByVehicleMake{?make}",
      "templated" : true
    },
    "findByFirstName" : {
      "href" : "http://localhost:8080/owners/search/findByFirstName{?firstName}",
      "templated" : true
    },
    "self" : {
      "href" : "http://localhost:8080/owners/search"
    }
  }
}
```

And test it out

```shell
$ curl -i "http://localhost:8080/owners/search/findByVehicleMake?make=VW"
HTTP/1.1 200
Vary: Origin
Vary: Access-Control-Request-Method
Vary: Access-Control-Request-Headers
Content-Type: application/hal+json
Transfer-Encoding: chunked
Date: Sun, 24 May 2020 16:23:22 GMT

{
  "_embedded" : {
    "owners" : [ {
      "firstName" : "Alexander",
      "lastName" : "Harrison",
      "vehicles" : [ {
        "make" : "VW",
        "model" : "Jetta",
        "mileage" : 42000,
        "owner" : 1
      } ],
      "_links" : {
        "self" : {
          "href" : "http://localhost:8080/owners/1"
        },
        "owner" : {
          "href" : "http://localhost:8080/owners/1"
        }
      }
    } ]
  },
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/owners/search/findByVehicleMake?make=VW"
    }
  }
}
```

## What happened to transactions?

Sorry, you can't do them. 

Just kidding.

### Clean up the objects

We use the astonishing [Lombok](https://projectlombok.org/) to add methods required by the Java object model: `toString()`, `equals()`, and `hashCode()` These were conspicuously missing from the definition we initially had a go at it.

!!! note
    There's a [Visual Studio Code extension](https://marketplace.visualstudio.com/items?itemName=GabrielBB.vscode-lombok) for Lombok.

This makes `Owner.java`

```java
package cloud.stephen.springdatajdbc;

import java.util.Set;

import org.springframework.data.annotation.Id;

import lombok.Data;

@Data
public class Owner {
  @Id
  private final Long id;
  private final String firstName;
  private final String lastName;
  private final Set<Vehicle> vehicles;

  static Owner of(final Long id, final String firstName, final String lastName, final Set<Vehicle> vehicles) {
    return new Owner(id, firstName, lastName, vehicles);
  }

  Owner withId(Long id) {
    return of(id, firstName, lastName, vehicles);
  }
}
```

This is about as pithy as you can get in Java. `Vehicle.java` follows the same pattern.

## Add a service with transactions

Create `OwnerService.java` with this

```java
package cloud.stephen.springdatajdbc;

import java.util.Optional;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@Transactional
public class OwnerService {
    private final OwnerRepository ownerRepository;

    public OwnerService(final OwnerRepository ownerRepository, final Utils utils) {
        this.ownerRepository = ownerRepository;
    }

    public Optional<Owner> findById(final Long ownerId) {
        return ownerRepository.findById(ownerId);
    }

    public Owner save(final Owner owner) {
        return ownerRepository.save(owner);
    }

    public Iterable<Owner> findAll() {
        return ownerRepository.findAll();
    }

    public void deleteAll() {
        ownerRepository.deleteAll();
    }
}
```

We don't expose REST endpoints for service transactions in a controller. We probably should at some point.

!!! note
    Yes, the pattern is that all the service methods are pass-through to the underlying repository. They simply decorate the repository calls with transactions. The service is the __only__ place we should __ever__ put transaction demarcation. Here's why.

    If we put a transaction annotation on repositories, we couldn't coordinate multiple repository calls from a service, something they do all the time. If we put transactions on the controller, then service calls couldn't coordinate other service calls: Controllers should only be concerned with data marshalling and conversion. Transactions on the service tier is simply the correct place.

`VehicleService` follows the same pattern.

## Let's test all this

Tests in Spring Boot tend to look like this: A class annotation `@SpringBootTest`, a `@BeforeEach` hook for test-level initialization, and test methods annotated with `@Test`. The class annotation executes all the Spring Boot initialization.

### Testing a repository

```java
@SpringBootTest
class OwnerRepositoryTest {
	@Autowired
	private OwnerRepository ownerRepository;

	@Autowired
	private VehicleRepository vehicleRepository;

	@BeforeEach
	void spickAndSpanDatabase() {
		ownerRepository.deleteAll();
		vehicleRepository.deleteAll();
	}

	@Test
	void whenDatabaseEmpty_ThenReturnEmpty() {
		final Iterable<Owner> actual = ownerRepository.findAll();
		final Iterable<Owner> expected = emptyList();

		assertThat(actual).isEqualTo(expected);
	}

    ...
```
And so on.

`assertThat()` comes from [Assert4j](https://joel-costigliola.github.io/assertj/)'s fluent methods.

Here's another test.

```java
@Test
void whenSaveTwoOwnersAndDeleteOne_ThenReturnOne() {
  final Owner stephen = ownerRepository.save(Owner.of(null, "Stephen", "Harrison", emptySet()));
  final Owner alexander = ownerRepository.save(Owner.of(null, "Alexander", "Harrison", emptySet()));

  ownerRepository.delete(stephen);

  final Iterable<Owner> actual = ownerRepository.findAll();
  final List<Owner> expected = Arrays.asList(alexander);

  assertThat(actual).isEqualTo(expected);
}
```

Pretty prosaic. You get the idea.

### Testing a service

This is bit subtler because transactions are in play: We have to test not only the service methods but transactions too.

The the test class contains a field `TransactionTemplate`, which has method `execute` that takes a callback for the code we want to run inside a transaction context. Super handy.

```java
@SpringBootTest
class OwnerServiceTest {
	@Autowired
	private OwnerService ownerService;

	@Autowired
	private Utils utils;

	@Autowired
	private PlatformTransactionManager transactionManager;

	private TransactionTemplate transactionTemplate;

	@BeforeEach
	void spickAndSpanDatabase() {
		ownerService.deleteAll();

		transactionTemplate = new TransactionTemplate(transactionManager);
	}

	@Test
	void whenDatabaseEmpty_ThenReturnEmpty() {
		final Iterable<Owner> owners = ownerService.findAll();

		assertThat(owners).isEmpty();
	}

  ...
```

Now our tests can use the transaction context and look like

```java
@Test
void whenServiceMethodFails_ThenTransactionIsRolledBack() {
  assertThat(ownerService.findAll()).isEmpty();

  final Owner existing = ownerService.save(utils.randomOwner());

  assertThat(ownerService.findAll()).isEqualTo(asList(existing));

  transactionTemplate.execute(status -> {
    try {
      // Succeeds
      //
      ownerService.save(utils.randomOwner());

      // Fails
      //
      ownerService.save(Owner.of(null, null, null, null));
    } catch (final Exception e) {
      status.setRollbackOnly();
    }

    return "ok";
  });

  assertThat(ownerService.findAll()).isEqualTo(asList(existing));
}
```

Notice that both `save()`s are inside a transaction context. And the whole thing works because each method in `OwnerService` either takes an existing or creates a new transaction: That's the semantics of `@Transactional`.

## The upshot

We looked at whether we could replace high level but often-fragile JPA with something much simpler. 

Judge for yourselves whether Spring Data JDBC is for you. For the author, there's no alternative for Java persistence nowadays.

We hope you enjoyed your lazy afternoon as much as we did.