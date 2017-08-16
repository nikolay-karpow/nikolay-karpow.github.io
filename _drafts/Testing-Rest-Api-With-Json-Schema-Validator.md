---
layout: post
title:  "Testing REST API with JSON schema validator"
date:   2017-08-16 20:00:00 -0600
categories: REST java
comments: true
---
The important thing about any API is its stability. As Jaroslav Tulach has written in [this book](https://www.amazon.com/Practical-API-Design-Confessions-Framework/dp/1430209739), "APIs are like stars; once introduced, they stay with us forever". RESTful APIs are not an exception.

REST endpoints often return some JSON objects. Usually, there is some DTO, which is created in the controller and serialized into JSON automatically by some serializer like [Jackson](http://wiki.fasterxml.com/JacksonHome) or [Gson](https://github.com/google/gson) or whatever else you prefer.

How would you test such an endpoint? Well, you'd probably call it with something like MockMvc, get the JSON response, deserialize it into the same DTO and compare its fields with expected values. What can go wrong here? If you change your DTO, the test will probably pass. E.g., you can rename or remove a field, but you use the same class in the test, so, it won't fail. However, you can't say, that about your clients, who expect the old response version.

Once we did exactly that: we accidentally (actually, by IntelliJ IDEA refactoring) renamed a field in such a DTO. After that the client application developers complained about a broken endpoint. And no test showed us, that we break the API, what a pity. 
