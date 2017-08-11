---
layout: post
title:  "Trap of apache HttpClient defaults"
date:   2017-08-11 15:15:00 -0600
categories: spring java
comments: true
---
Recently we chose to use apache http client in our code. We, actually, migrated form RestTemplate. We needed to write a custom decorator for it and with RestTemplate it's pretty inconvenient. When our app went to the testing environment, we found out some issues with default settings of HttpClient. I'm going to share them with you.

### Basic usage
Here is the basic example of ClosableHttpClient usage:

{% highlight java %}
@Test
public void test() throws Exception {
    CloseableHttpClient httpclient = HttpClients.createDefault();
    HttpGet httpGet = new HttpGet("http://localhost:9099/tests/test");

    try (CloseableHttpResponse httpResponse = httpclient.execute(httpGet)) {
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
        httpResponse.getEntity().writeTo(outputStream);

        System.out.println(new String(outputStream.toByteArray()));
    }
}
{% endhighlight %}

I run a simple app on 9099, which has the following controller method:
{% highlight java %}
private AtomicInteger simultaneousRequestsCount = new AtomicInteger(0);

@RequestMapping(method = RequestMethod.GET, value = "/test")
public String test() throws InterruptedException {
    String result = String.valueOf(simultaneousRequestsCount.incrementAndGet());
    TimeUnit.SECONDS.sleep(1);
    simultaneousRequestsCount.getAndDecrement();
    return result;
}
{% endhighlight %}
Controller responds with a count of the requests, that are currently serving.
Everything works fine so far, the test prints "1".

### Multiple threads
Now let's imaging we have a web application A, it receives an http request. Request handler does something useful and calls another service S with HttpClient (all in the request thread). The call is synchronous and takes some time. So, there are multiple threads, which send requests to the same service S using HttpClient.

{% highlight java %}
@Test
public void multipleThreads() throws Exception {
    CloseableHttpClient httpclient = HttpClients.createDefault();
    HttpGet httpGet = new HttpGet("http://localhost:9099/tests/test");

    int threadCount = 5;
    ExecutorService executorService = Executors.newFixedThreadPool(threadCount);

    for (int i = 0; i < threadCount; i++) {
        executorService.execute(() -> {
            try (CloseableHttpResponse httpResponse = httpclient.execute(httpGet)) {
                ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
                httpResponse.getEntity().writeTo(outputStream);

                System.out.println(new String(outputStream.toByteArray()));

            } catch (IOException e) {
                e.printStackTrace();
            }
        });
    }

    executorService.shutdown();
    executorService.awaitTermination(1, TimeUnit.MINUTES);
}
{% endhighlight %}
Now things get a little bit trickier. I don't know about you, but I'd expect this to print numbers from 1 to 5, but in fact it prints this:
{% highlight java %}
1
2
2
1
1
{% endhighlight %}

The reason is that HttpClient doesn't create a new connection for each request, but uses a connection pool instead. And by default it creates only 2 connections per route (route includes host and port). As we call the same route, all 5 threads go via just 2 connections. That was quite a surprise for me and my colleague. 
