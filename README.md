# Mockito

## Learning Goals

- Expand upon the application to provide an example for using mocks.
- Discuss the need for mocking objects when unit testing.
- Introduce a Mockito.

## Code-Along: Expand on the Spring Testing Demo Project

In our last lesson, we created a very simple application with one endpoint that
returns back the message `Hello World!` and wrote a unit test to test this
functionality.

Let's continue to expand off of this project and create an endpoint to return a
random cat fact. We'll be utilizing what we learned in the REST API Calls lesson
from the last module to connect to the API https://catfact.ninja/fact.

Create a new package called `dto` under `com.example.springtestingdemo`. Once
the package has been created, create a new Java class called `CatFactDTO` with
the following code:

```java
package com.example.springtestingdemo.dto;

import lombok.Data;

@Data
public class CatFactDTO {
    private String fact;
}
```

To connect to the external API, we'll need a `RestTemplate` bean. Modify the
`SpringTestingDemoApplication` to add the new bean method:

```java
package com.example.springtestingdemo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
public class SpringTestingDemoApplication {

   public static void main(String[] args) {
       SpringApplication.run(SpringTestingDemoApplication.class, args);
    }

    // Add a RestTemplate bean
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

}
```

Now we'll create another package called `service` under
`com.example.springtestingdemo`. Once the package has been created, create a new
Java class called `CatFactService` with the following code:

```java
package com.example.springtestingdemo.service;

import com.example.springtestingdemo.dto.CatFactDTO;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
@Slf4j
public class CatFactService {

    private static final String FACT_URI = "https://catfact.ninja/fact";
    private final RestTemplate restTemplate;
    
    @Autowired
    public CatFactService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }
    
    public CatFactDTO getFact() {
        CatFactDTO catFact = restTemplate.getForObject(FACT_URI, CatFactDTO.class);
        log.info("Retrieved cat fact from external source. Returning the DTO");
        return catFact;
    }
}
```

The above should be review from the REST API Calls lesson. Now that we have our
service written out, let's integrate it into our controller class and create an
endpoint, so we can retrieve a random cat fact:

```java
package com.example.springtestingdemo.controller;

import com.example.springtestingdemo.dto.CatFactDTO;
import com.example.springtestingdemo.service.CatFactService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class DemoController {

    private final CatFactService catFactService;

    @Autowired
    public DemoController(CatFactService catFactService) {
        this.catFactService = catFactService;
    }

    @GetMapping("/hello")
    public String hello() {
        return "Hello World";
    }

    @GetMapping("/cat-fact")
    public CatFactDTO getCatFact() {
       return catFactService.getFact();
    }
}
```

Great! Let's run our application and test this out ourselves first with Postman.
Open up Postman and type in "http://localhost:8080/cat-fact" for the request URL
and make sure that the request type is set to GET. Click "Send" and we should
see that the application has returned us a random cat fact.

![postman-get-cat-fact](https://curriculum-content.s3.amazonaws.com/spring-mod-2/testing/postman-cat-fact.png)

Note: The cat fact you received may be different from what is shown here since
the facts are "random".

Now let's try to unit test this! But wait... we added a service class, which
makes use of a `RestTemplate` bean. Does this make unit testing more
complicated? How are we supposed to unit test if a unit test isn't necessarily
supposed to know about all the inner workings of the Spring Framework?

## Mock Objects

Now and then, we come across a time in testing where it would really help to
mock up an object rather than add a bunch of dependencies to instantiate one.
The purpose of unit testing is to focus on testing the smallest piece of
functionality - not the state of other dependencies.

This is where Mockito can help us out!

**Mockito** is a mocking framework that can help us create mock objects in our
unit tests. It will help isolate the dependencies while allowing us to focus on
the testing aspect. The mock object that we can generate using Mockito will
return dummy data for testing purposes. This is important for unit testing
because we are only interested in testing the behavior of one method, not the
behavior of the other methods that this one method might depend on.

Let's look at how to use it by reviewing what we currently have in our unit test:

```java
package com.example.springtestingdemo.controller;

import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

class DemoControllerUnitTest {

   @Test
   void hello() {
      DemoController demoController = new DemoController();
      assertEquals("Hello World", demoController.hello());
   }
}
```

In review of how to write unit tests, let's add a `setUp()` method. The
`setUp()` method is where we can condense some set-up work that may be required
when running each test. We'll annotate it with the `@BeforeEach` annotation as
well to ensure it is run before each `@Test` runs.

```java
import com.example.springtestingdemo.service.CatFactService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import static org.junit.jupiter.api.Assertions.*;

class DemoControllerUnitTest {

    private DemoController demoController;
    private CatFactService catFactService;

    @BeforeEach
    void setUp() {
        catFactService = Mockito.mock(CatFactService.class);
        demoController = new DemoController(catFactService);
    }

    @Test
    void hello() {
        assertEquals("Hello World", demoController.hello());
    }
}
```

Since we modified the `DemoController` class, we'll need a `CatFactService`
instance to construct the `DemoController`. We'll construct both in the
`setUp()` method. But notice we're using Mockito to create the `CatFactService`
for us! This will create a mock instance of `CatFactService`. We can then use
the mock object just as we would use a normal instance of `CatFactService`.

If we run our unit test, everything should still pass. So now we can test the
new method we added to our `DemoController`: `getCatFact()`:

```java
package com.example.springtestingdemo.controller;

import com.example.springtestingdemo.dto.CatFactDTO;
import com.example.springtestingdemo.service.CatFactService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.mockito.Mockito;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.when;

class DemoControllerUnitTest {

    private DemoController demoController;
    private CatFactService catFactService;

    @BeforeEach
    void setUp() {
        catFactService = Mockito.mock(CatFactService.class);
        demoController = new DemoController(catFactService);
    }

    @Test
    void hello() {
        assertEquals("Hello World", demoController.hello());
    }

    @Test
    void getCatFact() {
        CatFactDTO catFact = new CatFactDTO();
        catFact.setFact("In ancient Egypt, when a family cat died," +
                "all family members would shave their eyebrows as a sign of mourning.");
        when(catFactService.getFact()).thenReturn(catFact);
        assertEquals(catFact, demoController.getCatFact());
    }
}
```

Since we're working with a mock service, we can hard code what we want the
`catFactService.getFact()` method to return, so that we know that no matter what
happens to the actual implementation of that method, our unit test will not
break. We know that the `getFact()` method will return a `CatFactDTO`, so let's
create a DTO object and set a random cat fact for it to return. Turning our
attention to the next line, we see
`when(catFactService.getFact()).thenReturn(catFact);` This construct allows us
to tell Mockito what to return when a specific method of our mock object is
called. This is what hard codes that specific response for every time that
method is called. Therefore, we can expect that same cat fact to be returned
whenever we call `demoController.getCatFact()` within this test method.

We should now be able to run the unit tests and have both of them pass.

There are a couple of crucial things to note here:

1. The Mockito mocking framework is able to substitute a "mock"/"fake"
   implementation of the `CatFactService` in the instance of the
   `DemoController` we are using in our unit test. The service class is usually
   something that is initialized and managed by Spring; however, we removed
   Spring from the equation in our unit test and were able to still successfully
   use a dummy version of the `CatFactService` exclusively to test the
   controller functionality. This is dependency injection in action, and is very
   important for managing the complexity of larger systems.
2. This is a great example of unit testing in action. The unit testing we are
   currently focused on,`getCatFact()` does not, and should not, care about the
   implementation of the service method, `getFact()`. Let's test something out.

   In the `CatFactService`, comment out the contents of the `getFact()` method
   and replace it with a `return null;` statement like this:

   ```java
    public CatFactDTO getFact() {
    //        CatFactDTO catFact = restTemplate.getForObject(FACT_URI, CatFactDTO.class);
    //        log.info("Retrieved cat fact from external source. Returning the DTO");
    //        return catFact;
    return null;
   }
   ```

   Re-run the `DemoControllerUnitTest`. See how everything still passed? The
   fact that we can make this unit test knowing we didn't even need to implement
   the service method that we'd be calling, is proof that our unit test is
   properly isolated from any of the dependencies of the method it tests.

## Conclusion

Mocking objects is a process that comes in handy in unit testing when the
functionality we are testing has external dependencies that really have nothing
to do with that specific function. In order to isolate and focus on the code
that needs to be tested, and ignore the state of the external dependencies, we
can create mocks of the dependencies. To help create mock objects, we can use a
mocking framework, like Mockito.

## Resources

- [Telerik: Unit Testing and Mocking Explained](https://www.telerik.com/products/mocking/unit-testing.aspx#:~:text=What%20is%20a%20mocking%20framework,substitutes%20for%20unit%20testing%20frameworks.)
- [Digital Ocean: Mockito Tutorial](https://www.digitalocean.com/community/tutorials/mockito-tutorial)
