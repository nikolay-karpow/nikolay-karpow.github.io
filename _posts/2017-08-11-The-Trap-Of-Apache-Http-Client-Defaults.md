---
layout: post
title:  "The trap of apache HttpClient defaults"
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

I run a simple app on the port 9099, which has the following controller method:
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
Controller responds with a count of the requests, that are currently being served.
Everything works fine so far, the test prints "1".

### Multiple threads
Now let's imaging we have a web application A, it receives an http request. Request handler does something useful and calls another service S with HttpClient (all in the request thread). The call is synchronous and takes some time. So, there are multiple threads, which send requests to the same service S using HttpClient.

{% highlight java %}
@Test
public void multipleThreads() throws Exception {
    CloseableHttpClient httpclient = HttpClients.createDefault();
    HttpGet httpGet = new HttpGet("http://localhost:9099/tests/test");

    int threadCount = 30;
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
Now things get a little bit trickier. I don't know about you, but I'd expect this to print numbers from 1 to 30. However, it prints this:
{% highlight java %}
1
2
2
1
1
...
{% endhighlight %}

The reason is that HttpClient doesn't create a new connection for each request, but uses a connection pool instead. And by default it creates only 2 connections per route (route includes host and port). As we call the same route, all 5 threads go via just 2 connections. That was quite a surprise to me.
Let's fix it. We should use another way to create HttpClient, which allows to pass an instance of PoolingHttpClientConnectionManager:
{% highlight java %}
@Test
public void multipleThreads_maxPerRoute() throws Exception {
    PoolingHttpClientConnectionManager connectionManager = new PoolingHttpClientConnectionManager();
    connectionManager.setDefaultMaxPerRoute(30);
    CloseableHttpClient httpclient = HttpClients.custom()
            .setConnectionManager(connectionManager)
            .build();

    HttpGet httpGet = new HttpGet("http://localhost:9099/tests/test");

    int threadCount = 30;
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
The important part is
{% highlight java %}
connectionManager.setDefaultMaxPerRoute(30);
{% endhighlight %}
So, now it should work, right? Unfortunately, we're not there yet. This code produces numbers from 1 to 20, not to 30. Why? Well, there is one more parameter - max total connection count. Unfortunately, nothing warns us, that our maxPerRoute is greater, than maxTotal. The final fix:
{% highlight java %}
@Test
public void multipleThreads_maxPerRouteMaxTotal() throws Exception {
    PoolingHttpClientConnectionManager connectionManager = new PoolingHttpClientConnectionManager();
    connectionManager.setDefaultMaxPerRoute(30);
    connectionManager.setMaxTotal(30);
    CloseableHttpClient httpclient = HttpClients.custom()
            .setConnectionManager(connectionManager)
            .build();

    HttpGet httpGet = new HttpGet("http://localhost:9099/tests/test");

    int threadCount = 30;
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
Now it works as expected at last, producing numbers from 1 to 30.

### Connection request timeout
By default, if there is no free connection in the pool, connection manager blocks until such a connection appears. If you want to make sure, it doesn't block indefinitely, you can set connection request timeout like this:
{% highlight java %}
PoolingHttpClientConnectionManager connectionManager = new PoolingHttpClientConnectionManager();
connectionManager.setDefaultMaxPerRoute(30);
connectionManager.setMaxTotal(30);

RequestConfig requestConfig = RequestConfig.custom()
        .setConnectionRequestTimeout(5000)
        .build();

CloseableHttpClient httpclient = HttpClients.custom()
        .setConnectionManager(connectionManager)
        .setDefaultRequestConfig(requestConfig)
        .build();
{% endhighlight %}

### Conclusion
Default configuration of apache HttpClient is not suitable for multithread usage. When you use it to work with the same service from different threads, make sure you set up the connection pool appropriately. Also, don't forget to close responses, otherwise connections never go back to the pool.

I hope, this post can save you some debugging time. More detailed information about connection management can be found [here](https://hc.apache.org/httpcomponents-client-ga/tutorial/html/connmgmt.html)
