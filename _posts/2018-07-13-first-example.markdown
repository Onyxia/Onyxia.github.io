---
layout: post
title:  "First example"
categories: Scala FP Refactoring
---

Let us start with some sample code

{% highlight scala %}

object AddRequestIdHeaderFilter extends SimpleFilter[Request, Response] {
  val requestIdKeyName: String = "request-id"
  val requestRetriesKeyName: String = "request-retries"

  override def apply(request: Request, service: Service[Request, Response]): twitter.util.Future[Response] = {
    request.headerMap.get(requestIdKeyName) match {
      case Some(requestId: String) =>
        val retries = 1 + request.headerMap.get(requestRetriesKeyName).flatMap(retries => Try(retries.toInt).toOption).getOrElse(0)
        request.headerMap.set(requestRetriesKeyName, s"${retries}")
        service(request).map { response: Response =>
          val _ = response.headerMap
            .set(requestIdKeyName, requestId)
            .set(requestRetriesKeyName, s"${retries}")
          response
        }
      case None =>
        //set the request-id and retries number
        val uuid = UUID.randomUUID().toString
        val _ = request.headerMap
          .set(requestIdKeyName, uuid)
          .set(requestRetriesKeyName, "0")
        service(request).map { response: Response =>
          val _ = response.headerMap
            .set(requestIdKeyName, uuid)
            .set(requestRetriesKeyName, "0")
          response
        }
    }
  }
}
{% endhighlight %}

It is quite a lot of code, and it will take a few minutes for us to understand (which is our first anti-pattern). Let's list some of things it does (i.e. find the concerns):

* We have static reference data defining keys. 
* 


Let's look for anti patterns
* We are in an object. So while we can be tested, it's very hard for anything us to be tested (candidate article)
* Why am I 'defining a transformation' and 'doing the transformation' in the same code. This is a 'well I would do if differently' thing, rather than a huge source of pain
* We are hard coded to the Finagle request object so we have pulled in 50 libraries
* We 'know' how the request-id is coded up in the finale request object, so if we change that this code will break. We should ideally express this kind of thing in one place (mini article about why lens and getter are good)
* the calculation of val retries has quite a lot of anti patterns... it's hard to count them.
* the uuid is calculated using a singleton: makes it much harder to test


Stages:
* decouple from Finagle (if we've done this before it is trivial if not we start that work) once done, all future refactors like this are easy
** Decouple the future
** Decouple the HttpRequest
** Decouple the HttpResponse
** Put on backlog restructure projects, because we can do it bit by bit

*: Change from object to class or trait to remove the antipattern that makes code using us hard to test

* change the antipattern of changing the mutable response to the pretence of immutability (sadly if the underlying object isn't immutable we can do nothing)

{% highlight scala %}
    service(newRequest).map { response: Response =>
          response.withNewHeaders(requestIdKeyName -> requestId, requestRetriesKeyName -> s"${retries}")
        }
{% endhighlight %}


result so far

{% highlight scala %}
class AddRequestIdHeaderFilter[M[_] : Monad, Request: HttpRequest, Response: HttpResponse] {
  val requestIdKeyName: String = "request-id"
  val requestRetriesKeyName: String = "request-retries"

  def apply(request: Request, service: Request => M[Response]): M[Response] = {

    request.headerMap.get(requestIdKeyName) match {
      case Some(requestId: String) =>
        val retries = 1 + request.headerMap.get(requestRetriesKeyName).flatMap(retries => Try(retries.toInt).toOption).getOrElse(0)
        val newRequest = request.withNewHeader(requestRetriesKeyName, s"${retries}")
        service(newRequest).map {_ .withNewHeaders(requestIdKeyName -> requestId, requestRetriesKeyName -> s"${retries}")
        }
      case None =>
        //set the request-id and retries number
        val uuid = UUID.randomUUID().toString
        val newRequest = request.withNewHeaders(requestIdKeyName -> uuid, requestRetriesKeyName -> "0")
        service(newRequest).map { _.withNewHeaders(requestIdKeyName -> uuid, requestRetriesKeyName -> "0")        }
    }
  }
}
{% endhighlight %}

We now get rid of the abomination (replacing it with our own less than beautiful



{% highlight scala %}
class AddRequestIdHeaderFilter[M[_] : Monad, Request: HttpRequest, Response: HttpResponse] {
  val requestIdKeyName: String = "request-id"
  val requestRetriesKeyName: String = "request-retries"


  def orZero(t: Option[String]): Int = t.map(_.toInt).getOrElse(0).onErrorReturn(0)

  def findRetryCount[R: HasHeaders](req: R) = orZero(req.headers.get(requestRetriesKeyName).map(1 + _))

  def findRetryId[R: HasHeaders](req: R) = req.headers.getOrElse(requestIdKeyName, UUID.randomUUID().toString)

  def addNewHeaders[R: HasHeaders](r: R): R = r.withNewHeaders(requestIdKeyName -> findRetryId(r), requestRetriesKeyName -> findRetryCount(r).toString)

  def apply(request: Request, service: Request => M[Response]): M[Response] =
    service(addNewHeaders(request)).map(addNewHeaders[Response])

}

{% endhighlight %}




Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
