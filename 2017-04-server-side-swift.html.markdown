---
title: Server Side Swift
date: 2017-03-27 20:38 UTC
issue: april-2017
tags:
author: Roland Leth
author_twitter: @rolandleth
editor: Adrian Kosmaczewski
editor_twitter: @akosma
order: 3
---


# Vapor and its callback and throwing stacks

This article is about Vapor, a framework for writing a server with Swift. I'm an iOS developer at heart, and I love Swift, so it was only natural to eventually convert my own [blog](https://rolandleth.com) to it. So I did, and I would like to present a couple of features that I found interesting - middleware, and their callback chain, and error handling. We'll also make use of protocols and extensions to create hierarchies, expand functionality, and better structure the project.

Let's start with a (somewhat) simple server:

```swift
let drop = Droplet() // 1

try? addProvider(VaporPostgreSQL.Provider.self) // 2
drop.preparations += Model.self // 3

drop.middleware.insert(RedirectMiddleWare(), at: 0) // 4
drop.middleware.append(GzipMiddleWare()) // 5
drop.middleware.append(HeadersMiddleware())

drop.get("/test", handler: { request in
	print("4")
	return try JSON(node: "This is a test page.")
})
```
*Configuring the Droplet.*

We start by creating our droplet (1), we add a database provider (2) and a `Model` entity (3) to be used with it, then add our middleware. the order of the middleware usually doesn't matter, and is done by `appending` (5), them. Sometimes, though, we might have very specific needs, and we want one of the middleware to run first. In this case, we can insert it to be first on the stack (4), thus being the first one to be called, before any other internal ones.


Now, onto the middleware: how they operate, and more on why the are added like that. Vapor offers a `Middleware` protocol, which requires just one method:

```swift
func respond(to request: Request, chainingTo next: Responder) throws -> Response
```
*The Middleware's only method.*

Through this method, a middleware takes the request, modifies it (or not) accordingly, then either passes it to the next responder (which can be another middleware, or an endpoint handler), which returns a response, modifies it (or not) accordingly, either directly creates and returns a response. We'll see each of these cases in the examples to follow.

First on the stack, the `RedirectMiddleware`:

```swift
struct RedirectMiddleware: Middleware {

	func respond(to request: Request, chainingTo next: Responder) throws -> Response {
		print("1")
		guard request.uri.scheme != "https" else { // 1
			let response = try next.respond(to: request) // 2
			print("7")
			return response // 3
		}

		let uri = uriWithSecureScheme

		print("alternate")
		return Response(redirect: uri.description) // 4
	}

}
```
*RedirectMiddleware.*

The only purpose of this middleware is to check if the current request is a secure one (1). If it's not, we create and return (4) a new response with the request's `URI`, modified to have and `https` scheme. If it is already secure, we pass it to the next responder (2), and we are done - we return whatever that returns us (3). Since we want *all* requests to be secure, there's no need to let the request pass through all other middleware, thus its position at index `0`.

Our next middleware, the `HeadersMiddleware`:

```swift
struct HeadersMiddleware: Middleware {

	func respond(to request: Request, chainingTo next: Responder) throws -> Response {
		print("2")
		let response = try next.respond(to: request) // 1
		print("6")

		response.headers["Cache-Control"] = "public, max-age=86400" // 2

		// Disable the embedding of the site in an external one.
		response.headers["Content-Security-Policy"] = "frame-ancestors 'none'"
		response.headers["X-Frame-Options"] = "DENY"

		return response // 3
	}

}
```
*HeadersMiddleware.*

This one passes the request directly to the next responder (1), and only after that returns a response, we set some headers on it (2), and return it (3).

Next on the stack, and next responder, the `GzipMiddleware`:

```swift
struct GzipMiddleware: Middleware {

	func respond(to request: Request, chainingTo next: Responder) throws -> Response {
		print("3")
		let response = try next.respond(to: request) // 1
		print("5")

		response.body = gzippedBody // 2
		response.headers["Content-Encoding"] = "gzip" // 3
		response.headers["Content-Length"] = gzippedBody.length

		return response // 4
	}

}
```
*GZipMiddleware.*

Here we also pass the response directly to the next responder (1), then add a couple of headers as well (3), but we also change the body (2) before returning (4) it.

How are middleware handled internally? Basically, just like we use them ourselves - `Droplet` conforms to the `Responder` protocol, which has just one method:

```swift
func respond(to request: Request) throws -> Response
```
*The Droplet's respond method.*

Compared to the `Middleware` protocol, this one can not be called by chaining it to anything else; it chains all of the middleware in its implementation, though:

```swift
extension Droplet: Responder {

	public func respond(to request: Request) throws -> Response {

		[...]

		print("0")
      let mainResponder = middleware.chain(to: routerResponder) // 1
		var response: Response

		do {
			response = try mainResponder.respond(to: request) // 2
		}
		catch {
			return Response(status: .internalServerError, headers: [:], body: "Error message".bytes) // 3
		}

		print("10")
		return response // 4
	}

}
```
*The Droplet's respond method implementation.*

Here we can see that a responder is created (1) by chaining all middlewares (not calling them, yet), and it's used to create a response out of the request (2), that will be returned (4), if everything goes OK. If that fails, a new response is created and returned (3), which will contain an error message, and a status code. How are the middleware chained (1), and called? With an extension on `Collection` and a `Responder` subclass, whose sole purpose is to hold and call a closure:

```swift
extension Request {

	public struct Handler: Responder {
		public typealias Closure = (Request) throws -> Response

		private let closure: Closure

		public init(_ closure: @escaping Closure) {
			self.closure = closure // 1
		}

		public func respond(to request: Request) throws -> Response {
			return try closure(request) // 2
		}
	}

}

extension Collection where Iterator.Element == Middleware {

    func chain(to responder: Responder) -> Responder {
        return reversed().reduce(responder) { nextResponder, nextMiddleware in // 3
            return Request.Handler { request in
                return try nextMiddleware.respond(to: request, chainingTo: nextResponder) // 4
            }
        }
    }

}
```
*Extending Request and Collection.*

The `chain` method reverses all middleware (3) and at each step, it creates and returns a new `Handler`, that calls the `Middleware`'s protocol method we talked about earlier, by passing the request and the current responder to it. This way, if we remember our (shortened, for an easier explanation) middleware stack, `[Redirect, Headers, Gzip]` will go through the chain method like this:

```text
reverse ->
[Gzip, Headers, Redirect] ->
create and return a Handler, that in its closure calls Gzip's respond(to: request, chainingTo: mainResponder) ->
create and return a Handler, that in its closure calls Headers' respond(to: request, chainingTo: gzipResponder) ->
create and return a Handler, that in its closure calls Redirect's respond(to: request, chainingTo: headersResponder) ->
return the Redirect Handler
```
*Chaining the middleware's closures.*

`Droplet`'s `respond` method is the place where the returned `Handler`'s `closure` is called (2), which will call the next, and so on. The last one that will be called is the `handler` closure passed in our `get` method, and is the first one that chains the responses / errors back:

```swift
drop.get("/test", handler: { request in
	print("4")
	return try JSON(node: "This is a test page.")
})
```
*get's handler.*

Finally, let's put everything together, and follow the chain. A secure requets gets sent like this:

```text
create the main responder ("0") ->
RequestMiddleware ("1") ->
Other, internal middleware ->
HeadersMiddleware ("2") ->
GZipMiddleware ("3") ->
MiscMiddleware1 ->
MiscMiddleware2
```
*Secure request chain.*

Then, the response is sent like this:

```text
get's handler ("4") ->
GZipMiddleware ("5") ->
HeadersMiddleware ("6") ->
Other, internal middleware ->
RequestMiddleware ("7") ->
response is sent to client ("10")
```
*Response chain.*

A non-secure request has a slightly longer path to follow:

```text
get("/test") ->
RequestMiddleware ("1") ->
RequestMiddleware ("alternate") ->
start over ->
RequestMiddleware ("1") ->
[continue with the rest of the secure path]
```
*Non-secure request chain.*

As you can see, the main advantages of middleware are that we can modularize code, by breaking it down into smaller, specialized classes. Due to the fact that each serves a single, usually small, purpose, they can be reasoned with much easier, are more expressive, and easier to test / change / remove. None of them know about each other, they all act on the request / response independently, without caring what happens down / up the chain.

The throwing chain follows the exact same path, and, at its core, is very straightforward: if anything throws an error and it is not handled, it will be passed back the request / response chain until a handler is found, or, it will reach the internal catch block (3 in the `extension Droplet: Responder` snippet), where a `Response` is created with a generic error message.

Let us see how one could handle errors. First, the most straightforward, in-place handling:

```swift
extension String: Error { } // 1

drop.get("/handle", handler: { request in
	do {
		let answer = try findAnswer(in: request)
		return JSON(answer) // 2
	}
	catch {
		return JSON("Answer couldn't be computed.") // 3
	}
})

drop.get("/pass-along", handler: { request in
	let answer = try findAnswer(in: request) // 4
	return JSON(answer) // 5
})

func findAnswer(for request: Request) throws -> String {
	guard let answer = request.extract("answer") as? String else {
		throw "Answer parameter is not present." // 6
	}

	guard computingFinishedFast else {
		throw "Computing took too long to finish."
	}

	return answer
}
```
*Throwing strings.*

By conforming `String` to `Error` (1), we can throw strings (6), just as if we'd have created an enum with associated values, conforming to `Error`.

In the case of the `handle` endpoint, if everything goes well in `findAnswer`, we return a `JSON` created out of our `answer` (2); if something goes wrong, we catch the error, and return a `JSON` created out of our error (3).

In the case of the `pass-along` endpoint, if everything goes well, the same scenario applies (5); but if an error is thrown, since we aren't handling it here, it will be passed on the callback chain (4), as previously mentioned.

We can already see that if we add other endpoints that makes use of the `findAnswer` method, we'll have to treat the error again, and again.

Another approach, recommended by Vapor's documentation, and a great idea per se, is to create an error handling middleware, that handles all errors. Let's start by creating an `AppError` entity, and change our `findAnswer` method:

```swift
enum AppError: Error {
	case argumentNotPresent(name: String)
	case computingTimedOut
}

drop.get("/handle", handler: { request in
	do {
		let answer = try findAnswer(in: request)
		return JSON(answer) // 1
	}
	catch {
		return JSON("Answer couldn't be computed.") // 2
	}
})

drop.get("/pass-along", handler: { request in
	let answer = try findAnswer(in: request) // 3
	return JSON(answer) // 4
})

func findAnswer(for request: Request) throws -> String {
	guard let answer = request.extract("answer") as? String else {
		throw AppError.argumentNotPresent(name: "answer") // 5
	}

	guard computingFinishedFast else {
		throw AppError.computingTimedOut // 6
	}

	return answer
}
```
*Throwing custom errors.*

We can still handle errors in-place (1, 2), but since we will be adding a middleware to treat errors, passing them along (3, 4) would be the go-to choice. Our `findAnswer` method now throws an `AppError` (5) with an associated value for the expected parameter, and is also throwing an error for the time out (6).

As we already know, we only need to implement one method, so let's see how we can do that:

```swift
struct ErrorHandlingMiddleware: Middleware {

	func respond(to request: Request, chainingTo next: Responder) throws -> Response {
        do {
            return try next.respond(to: request) // 1
        }
		  catch AppError.argumentNotPresent(let name) { // 2
            throw Abort.custom( // 3
                status: .badRequest,
                message: "Argument \(name) was not found." // 4
            )
        }
		  catch AppError.computingTimedOut { // 5
			  throw Abort.custom(
                status: .requestTimeout, // 6
                message: "Computing an answer has timed out."
            )
		  }
		  catch { // 7
				return Response( // 8
					status: .serverError,
					message: "Something unexpected happened."
				)
		  }
    }

}
```
*ErrorHandlingMiddleware.*

The only thing we need to do is pass the request to the next responder inside a `do-catch` block. If everything goes well (1), the request will have reached our `get` handlers, the response has reached back to us (1), and we can return it further along the chain (1).

If something went wrong in our `findAnswer` method, we can do the catching ourselves and create and throw specific `Abort` errors (3) - an enum offered by Vapor, or create and return a `Response` (8).

Catching an error with an associated value (2) gives us the flexibility of extracting the parameter that was not present (4), opening up the possibility of throwing the same error, but for a different parameter:

```swift
func findMeaningOfLife(for request: Request) throws -> Int {
	guard let answer = request.extract("meaningOfLife") as? Int else {
		throw AppError.argumentNotPresent(name: "meaningOfLife")
	}

	return 42
}
```
*Throwing errors with associated values.*

We can now treat the timeout error separately (5), with an appropriate status (6), and we can also have a fallback `catch` (7) for everything else, where we simply return a `serverError`, with a generic message.

Finally, we just need to add our new middleware to the droplet, and we're good to go:

```swift
// [...]
drop.middleware.append(ErrorHandlingMiddleware())
// [...]
```
*Adding the error handling middleware.*

I'd like to mention one more neat feature of middleware: per-server configuration. Let's say we want our `ErrorHandlingMiddleware` to only run on production; locally we want our app to just crash, for some reason. Instead of appending the middleware like we saw so far, droplets offer [configuration files](https://vapor.github.io/documentation/guide/config.html), and we can make use of those.

First, let's add the middleware as a configurable, instead of appending it as a middleware:

```swift
// [...]
// drop.middleware.append(ErrorHandlingMiddleware()) -> replaced with:
drop.addConfigurable(middleware: ErrorHandlingMiddleware(), name: "error-handling")
// [...]
```
*Adding a configurable middleware.*

Then let's add the appropriate keys in our `Config/production/droplet.json` and `Config/staging/droplet.json` files:

```json
// production/droplet.json
{
    ...
    "middleware": {
        "server": [
            ...
            "error-handling", // 1
            ...
        ],
        "client": [
            ...
        ]
    },
    ...
}

// staging/droplet.json
{
    ...
    "middleware": {
        "server": [
            ... // 2
        ],
        "client": [
            ...
        ]
    },
    ...
}
```
*droplet.json*

Now, the `ErrorHandlingMiddleware` will be added to the production server's middleware (1) on app launch, but not on staging (2). A middleware can be added to both server and client, and to any `Config/server-type/droplet.json` file; also the ordering of the array will be respected.
<br>
As we're reaching the end of the article, we can see that Swift is a pretty viable solution for setting up a server. Protocols let us define a `Middleware` entity to be easily used to modify our requests / responses, and extensions gave us versatility of adding their handlers to a chained stack. We also made use of extensions to open up the possibility of `throwing` a string, for basic error handling, but also for simple things, like separating the `Request.Handler` hierarchy out of the main struct, for a better project structure.

By using Swift both on the server and on the client, one could also share models, functionality, and helpers / utilities, avoid duplication and by looking at the server and client as a whole, everything would be easier to reason with.

As for Vapor itself, as the framework of choice? It's not the best, nor the worst, according to [this](https://medium.com/@rymcol/benchmarks-for-the-top-server-side-swift-frameworks-vs-node-js-24460cfe0beb) article (9 months old, things might have changed), but it offers a really similar mindset to Express.js and Sinatra (those were my previous two choices for my own [blog](https://rolandleth.com)), and it also fit my needs pretty well (a blog with not too many requests, that didn't require any APIs, either).