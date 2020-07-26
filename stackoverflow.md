I attempted to create a springboot project by doing the following

* in eclipse 
    
    file > new > project > maven project (template) >
 
    <groupId>am.ik.archetype</groupId>
    <artificact>spring-boot-blank-archetype<artifact>
    <version>1.0.6</version>

To my surprise there was more than a pom.xml, there was java code. I have committed this project at this point to TODO

Several problems
- Wouldn't compile because it called a constructor that didn't exist
- I wrote the constructor and got it to compile and fixed it.
- Then the program had a run-time error

    Caused by: java.lang.ClassNotFoundException: javax.xml.bind.ValidationException
	at java.base/jdk.internal.loader.BuiltinClassLoader.loadClass(BuiltinClassLoader.java:583)
	at java.base/jdk.internal.loader.ClassLoaders$AppClassLoader.loadClass(ClassLoaders.java:178)
	at java.base/java.lang.ClassLoader.loadClass(ClassLoader.java:521)
	... 33 more
    
- After reading [this article on Stack Overflow](https://stackoverflow.com/questions/43574426/how-to-resolve-java-lang-noclassdeffounderror-javax-xml-bind-jaxbexception-in-j) I modified the pom.xml to bring in the missing dependencies no long incorporated in Java 9, and rebuilt my program.

- Running http://localhost:8080/calc?left=100&right=200 resulted in <em>There was an unexpected error (type=Not Acceptable, status=406)</em>
- Based on [this article](https://dzone.com/articles/spring-framework-restcontroller-vs-controller) I would have expected Result to have been marshaled to JSON and a JSON response received by the browser
- This test

    package com.bluehippo.springy;

    import com.jayway.restassured.RestAssured;
    import org.junit.Before;
    import org.junit.Test;
    import org.junit.runner.RunWith;
    import org.springframework.beans.factory.annotation.Value;
    import org.springframework.boot.test.IntegrationTest;
    import org.springframework.boot.test.SpringApplicationConfiguration;
    import org.springframework.test.context.junit4.SpringJUnit4ClassRunner;
    import org.springframework.test.context.web.WebAppConfiguration;

    import static com.jayway.restassured.RestAssured.given;
    import static com.jayway.restassured.RestAssured.when;
    import static org.hamcrest.CoreMatchers.is;

    @RunWith(SpringJUnit4ClassRunner.class)
    @SpringApplicationConfiguration(classes = App.class)
    @WebAppConfiguration
    @IntegrationTest({"server.port:0",
        "spring.datasource.url:jdbc:h2:mem:springy;DB_CLOSE_ON_EXIT=FALSE"})
    public class HelloControllerTest {
    @Value("${local.server.port}")
    int port;

    @Before
    public void setUp() throws Exception {
        RestAssured.port = port;
    }

    @Test
    public void testHello() throws Exception {
        when().get("/").then()
                .body(is("Hello World!"));
    }

    @Test
    public void testCalc() throws Exception {
        given().param("left", 100)
                .param("right", 200)
                .get("/calc")
                .then()
                .body("left", is(100))
                .body("right", is(200))
                .body("answer", is(300));
        }
    }

- Given that the HelloController:

    package com.bluehippo.springy;

    import lombok.Data;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.jdbc.core.namedparam.MapSqlParameterSource;
    import org.springframework.jdbc.core.namedparam.NamedParameterJdbcTemplate;
    import org.springframework.web.bind.annotation.RequestMapping;
    import org.springframework.web.bind.annotation.RequestParam;
    import org.springframework.web.bind.annotation.RestController;

    @RestController
    public class HelloController {
    @Autowired
    NamedParameterJdbcTemplate jdbcTemplate;

    @RequestMapping("/")
    String hello() {
        return "Hello World!";
    }

    @Data
    static class Result {
        private final int left;
        private final int right;
        private final long answer;
        
        public Result (int left, int right, long answer) {
        	this.left=left;
        	this.right=right;
        	this.answer=answer;
        	
        	if (left+right >500) {
        		throw new IllegalArgumentException("left+right must be less than 500");
        	}
        }
    }

    // SQL sample
    @RequestMapping("calc")
    Result calc(@RequestParam int left, @RequestParam int right) {
        MapSqlParameterSource source = new MapSqlParameterSource()
                .addValue("left", left)
                .addValue("right", right);
        return jdbcTemplate.queryForObject("SELECT :left + :right AS answer", source,
                (rs, rowNum) -> new Result(left, right, rs.getLong("answer")));
        }
    }

- Given that the controller could not have possibly had called the constructor with those arguments I have no idea what is going on and what is this unit test <em>doing</em>?


Recapping,

1. Where do these archetypes come from, can anybody just post this stuff?
2. Why did the response not get marshaled into the JSON
3. What is this unit test even doing as it doesn't call the method the calc url invokes