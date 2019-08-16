# Appendix：Spring MVC 3.2 Preview: Introducing Servlet 3, Async Support

ENGINEERING

![img](https://gravatar.com/avatar/4b12b9c0c665bc0345467c1a218ed0f7?s=20&d=mm) [ROSSEN STOYANCHEV](https://spring.io/team/rstoyanchev)

MAY 07, 2012

 

**Last updated on November 5th, 2012 (Spring MVC 3.2 RC1)**

**Overview**

Spring MVC 3.2 introduces Servlet 3 based asynchronous request processing. This is the first of several blog posts covering this new capability and providing context in which to understand how and why you would use it.

The main purpose of early releases is to seek feedback. We've received plenty of it both here and in [JIRA](http://jira.springsource.org/browse/SPR) since this was first posted after the 3.2 M1 release. Thanks to everyone who gave it a try and commented! There have been numerous changes and there is still time for more feedback!

**At a Glance**

From a programming model perspective the new capabilities appear deceptively simple. A controller method can now return a `java.util.concurrent.Callable` to complete processing asynchronously. Spring MVC will then invoke the `Callable` in a separate thread with the help of a [`TaskExecutor`](http://static.springsource.org/spring/docs/3.1.x/spring-framework-reference/html/scheduling.html). Here is a code snippet before:

```java
// Before
@RequestMapping(method=RequestMethod.POST)
public String processUpload(final MultipartFile file) {
    // ...
    return "someView";
}

// After
@RequestMapping(method=RequestMethod.POST)
public Callable<String> processUpload(final MultipartFile file) {

  return new Callable<String>() {
    public Object call() throws Exception {
      // ...
      return "someView";
    }
  };
}
```

A controller method can also return a `DeferredResult` (new type in Spring MVC 3.2) to complete processing in a thread not known to Spring MVC. For example reacting to a JMS or an AMQP message, a Redis notification, and so on. Here is another code snippet:

```java
@RequestMapping("/quotes")
@ResponseBody
public DeferredResult<String> quotes() {
  DeferredResult<String> deferredResult = new DeferredResult<String>();
  // Add deferredResult to a Queue or a Map...
  return deferredResult;
}


// In some other thread...
deferredResult.setResult(data);
// Remove deferredResult from the Queue or Map
```

The above samples lead to many questions and we'll get to more details in subsequent posts. For now, I'll begin by providing some context around these features.

**Motivation for Asynchronicity In Web Applications**

The most basic motivation for asynchronicity in web applications is to handle requests that take longer to complete. Maybe a slow database query, a call to an external REST APIs, or some other I/O-bound operations. Such longer requests can exhaust the Servlet container thread pool quickly and affect scalability.

In some cases you can return to the client immediately while a background job completes processing. For example sending an email, kicking off a database job, and others represent *fire-and-forget* scenarios that can be handled with Spring's [@Async](http://static.springsource.org/spring/docs/3.1.x/spring-framework-reference/html/scheduling.html#scheduling-annotation-support-async) support or by posting an event to a [Spring Integration](http://www.springsource.org/spring-integration) channel and then returning a confirmation id the client can use to query for the results.

In other cases, where the result is required, we need to decouple processing from the Servlet container thread or else we'll exhaust its thread pool. Servlet 3 provides just such support where a Servlet (or a Spring MVC controller) can indicate the response should be left open after the Servlet container thread is exited.

To achieve this, a Servlet 3 web application can call `request.startAsync()` and use the returned *AsyncContext* to continue to write to the response from some other separate thread. At the same time from a client's perspective the request still looks like any other HTTP request-response interaction. It just takes longer to complete. The following is the sequence of events:

1. Client sends a request
2. Servlet container allocates a thread and invokes a servlet in it
3. The servlet calls `request.startAsync()`, saves the `AsyncContext`, and returns
4. The container thread is exited all the way but the response remains open
5. Some other thread uses the saved `AsyncContext` to complete the response
6. Client receives the response

There is of course a lot more to the Servlet async support. You can find [various](http://www.javaworld.com/javaworld/jw-02-2009/jw-02-servlet3.html) [examples](http://codingjunkie.net/jee-6-and-spring-mvc/) and [writeups](https://nurkiewicz.blogspot.com/2011/03/tenfold-increase-in-server-throughput.html), but the above sums up the basic, minimum concept you need to know. The [next post](http://blog.springsource.org/2012/05/08/spring-mvc-3-2-preview-techniques-for-real-time-updates/)covers a second motivation for asynchronous request processing -- the need for browsers to receive information updates in real time.



[comments powered by Disqus](https://disqus.com/)



原文地址：https://spring.io/blog/2012/05/07/spring-mvc-3-2-preview-introducing-servlet-3-async-support



