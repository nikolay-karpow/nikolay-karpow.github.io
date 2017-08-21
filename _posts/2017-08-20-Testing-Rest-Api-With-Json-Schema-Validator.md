---
layout: post
title:  "Testing REST API with JSON schema validator"
date:   2017-08-20 20:00:00 -0600
categories: REST java
comments: true
---
The important thing about any API is its stability. As Jaroslav Tulach has written in [this book](https://www.amazon.com/Practical-API-Design-Confessions-Framework/dp/1430209739), "APIs are like stars; once introduced, they stay with us forever". RESTful APIs are not an exception.

REST endpoints often return some JSON objects. Usually, there is some DTO, which is created in the controller and serialized into JSON automatically by some serializer like [Jackson](http://wiki.fasterxml.com/JacksonHome) or [Gson](https://github.com/google/gson) or whatever else you prefer.

Let's look at an example. Here is our endpoint:
{% highlight java %}
@RequestMapping("/greeting")
public ResponseEntity greeting() {
    Greeting greeting = new Greeting(new Person("Heisenberg"), "Guten Tag");
    return ResponseEntity.ok(greeting);
}
{% endhighlight %}

And here is what Greeting class looks like:
{% highlight java %}
@Data
public class Greeting {
    private final Person from;
    private final String msg;
}

@Data
public class Person {
    private final String name;
}
{% endhighlight %}

How would you test it? Well, you'd probably call it with something like MockMvc, get the JSON response, deserialize it into the same DTO (Greeting) and compare its fields with expected values.

{% highlight java %}
@Test
public void greetingReturnsCorrectDto() {
    Greeting greeting = RestAssuredMockMvc.given()
            .get("greeting")
            .andReturn()
            .as(Greeting.class);

    Greeting expected = new Greeting(new Person("Heisenberg"), "Guten Tag");
    assertEquals(greeting, expected);
}
{% endhighlight %}

What can go wrong here? If you change your DTO, the test will, probably, pass. E.g., you can rename or remove a field, but you use the same class in the test, so, it won't fail. However, you can't say, that about your clients, who expect the old response version.

Exactly that once happened in our project: we accidentally (actually, by IntelliJ IDEA refactoring) renamed a field in such a DTO. After that the client application developers complained about a broken endpoint. And no test showed us, that we break the API. what a pity...

One solution to this problem is to have a separate DTO class for tests. But it might be pretty inconvenient because of all that code-duplication consequences. It's, actually, pretty handy to use the same class. It might implement equals or other useful methods, which make testing easier.

Another alternative is to validate the response against a [JSON schema](http://json-schema.org/). It can be done with [RestAssuredMockMvc](http://static.javadoc.io/com.jayway.restassured/spring-mock-mvc/2.9.0/com/jayway/restassured/module/mockmvc/RestAssuredMockMvc.html) and [JsonSchemaValidator](http://static.javadoc.io/com.jayway.restassured/json-schema-validator/2.9.0/com/jayway/restassured/module/jsv/class-use/JsonSchemaValidator.html). We'll need the following dependencis:

{% highlight xml %}
<dependency>
    <groupId>com.jayway.restassured</groupId>
    <artifactId>spring-mock-mvc</artifactId>
    <version>2.9.0</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>com.jayway.restassured</groupId>
    <artifactId>json-schema-validator</artifactId>
    <version>2.9.0</version>
    <scope>test</scope>
</dependency>
{% endhighlight %}

The test:
{% highlight java %}
@Test
public void greetingsReturnsJsonWhichMatchesExpectedSchema() throws Exception {
    String greetingSchema = readGreetingSchema();

    RestAssuredMockMvc.given()
            .get("greeting")
            .then()
            .body(JsonSchemaValidator.matchesJsonSchema(greetingSchema));
}

private String readGreetingSchema() throws IOException {
    return new String(toByteArray(getResource("greeting-schema.json").openStream()));
}
{% endhighlight %}

The test is, actually, pretty trivial. The schema is a bit trickier, here is how it looks like for our example:

{% highlight json %}
{
  "type": "object",
  "required": [
    "from",
    "msg"
  ],
  "properties": {
    "from": {
      "type": "object",
      "required": [
        "name"
      ],
      "properties": {
        "name": {
          "type": "string"
        }
      },
      "additionalProperties": false
    },
    "msg" : {
      "type": "string"
    }
  },
  "additionalProperties": false
}
{% endhighlight %}

The schema is a valid JSON object. First we describe the overall type of the response - it's an object with required fields "from" and "msg", then we describe each field in the same manner in the "properties" object.

I have to admit, that writing such a schema is a rather tedious work if your response is of somewhat complicated structure. However, if it's for your public API testing, you're not going to change it often (the whole point is to check, it hasn't changed).  

There is a [tool](https://jsonschema.net/) for generating the schema automatically from JSON. I haven't used it a lot, but it looks promising. Unfortunately, I haven't found anything useful like that for generating the schema from Java code.

I wish you not to break your API. Hopefully, this article might help you with that.
