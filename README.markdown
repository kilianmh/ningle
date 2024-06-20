# ningle

[![Build Status](https://travis-ci.org/fukamachi/ningle.svg?branch=master)](https://travis-ci.org/fukamachi/ningle)

"ningle" is a lightweight web application framework for Common Lisp.

## Usage

#### Start server
```common-lisp
(defvar *app* (make-instance 'ningle:app))

(clack:clackup *app*)
;; This should start clack server on http://127.0.0.1:5000
```
#### Most simple route
```common-lisp
(setf (ningle:route *app* "/")
      "Welcome to ningle!")
;; http://127.0.0.1:5000/
```
#### Parameters values
```common-lisp
(setf (ningle:route *app* "/login" :method :POST)
      #'(lambda (params)
          (if (authorize (cdr (assoc "username" params :test #'string=))
                         (cdr (assoc "password" params :test #'string=)))
              "Authorized!"
              "Failed...Try again.")))
```
Now you can access to http://localhost:5000/ and then ningle should show you "Welcome to ningle!".

## Installation
```common-lisp
(ql:quickload :ningle)
```

## Description

ningle is a fork project of [Caveman](http://fukamachi.github.com/caveman/). ningle doesn't require you to generate a project skeleton.

As this is a thin framework, you need to have subtle knowledge about [Clack](http://clacklisp.org). It is a server interface ningle bases on.

## Getting started

#### Request information
```common-lisp
(setf (ningle:route *app* "/request-information" :method :GET)
      #'(lambda (params)
	  (declare (ignore params))
	  (with-output-to-string (s)
	    (let ((*print-case* :downcase))
	      (format s "REQUEST-DATA: ~s" ningle:*request*)))))
```
This prints all request information.

#### Response information
```common-lisp
(setf (ningle:route *app* "/response-information" :method :GET)
      #'(lambda (params)
	  (declare (ignore params))
	  (with-output-to-string (s)
	    (let ((*print-case* :downcase))
	      (format s " RESPONSE-DATA: ~s" ningle:*response*)))))
```
#### Lack normal response
```common-lisp
(setf (ningle:route *app* "/lack-normal-response" :method :GET)
      #'(lambda (params)
	  (declare (ignore params))
	  (list 201
	        (list :content-language "de-DE")
	        (list "Das ist eine normale Nachricht"))))
```
See [Lack normal response](https://github.com/fukamachi/lack?tab=readme-ov-file#normal-response.
#### Usage of `*response*`
```common-lisp
(setf (ningle:route *app* "/ningle-response" :method :GET)
      #'(lambda (params)
          (declare (ignore params))
          (setf (lack.response:response-headers ningle:*response*)
                (append (lack.response:response-headers ningle:*response*)
                        (list :content-type "application/json"
			      :)))
          (setf (lack.response:response-body ningle:*response*)
                "[{\"test-response\":true}]")
          nil)) 
```
;; http://127.0.0.1:5000/ningle-response
;; supply content-type and content via ningle:*response*
;; details in response section

#### Hybrid response
```common-lisp
(setf (ningle:route *app* "/hybrid-response" :method :GET)
      #'(lambda (params)
          (declare (ignore params))
          (setf (lack.response:response-headers ningle:*response*)
                (append (lack.response:response-headers ningle:*response*)
                        (list :content-type "application/json")))
          (setf (lack.response:response-body ningle:*response*)
                "[{\"test-response\":false}]")
          "[{\"test-response\":true}]"))
```
;; Set response header in ningle:*response*
;; set output with returned value from route.
;; result value supersedes value set via response-headers

### Routing

ningle has the [Sinatra](http://www.sinatrarb.com/)-like routing system.

```common-lisp
;; GET request (default)
(setf (ningle:route *app* "/" :method :GET) ...)

;; POST request
(setf (ningle:route *app* "/" :method :POST) ...)

;; PUT request
(setf (ningle:route *app* "/" :method :PUT) ...)

;; DELETE request
(setf (ningle:route *app* "/" :method :DELETE) ...)

;; OPTIONS request
(setf (ningle:route *app* "/" :method :OPTIONS) ...)
```

Route pattern may contain "keyword" to put the value into the argument.

```common-lisp
(setf (ningle:route *app* "/hello/:name")
      #'(lambda (params)
          (format nil "Hello, ~A" (cdr (assoc :name params)))))
```

The above controller will be invoked when you access to "/hello/Eitaro" or "/hello/Tomohiro", and then `(cdr (assoc :name params))` will be "Eitaro" and "Tomohiro".

Route patterns may also contain "wildcard" parameters. They are accessible by `(assoc :splat params)`.

```common-lisp
(setf (ningle:route *app* "/say/*/to/*")
      #'(lambda (params)
          ; matches /say/hello/to/world
          (cdr (assoc :splat params)) ;=> ("hello" "world")
          ))

(setf (ningle:route *app* "/download/*.*")
      #'(lambda (params)
          ; matches /download/path/to/file.xml
          (cdr (assoc :splat params)) ;=> ("path/to/file" "xml")
          ))
```

Route matching with Regular Expressions:

```common-lisp
(setf (ningle:route *app* "/hello/([\\w]+)" :regexp t)
      #'(lambda (params)
          (format nil "Hello, ~A!" (first (cdr (assoc :captures params))))))
```

### Requirements

Routes may include a variety of matching conditions, such as the Accept:

```common-lisp
(setf (ningle:route *app* "/" :accept '("text/html" "text/xml"))
      #'(lambda (params)
          (declare (ignore params))
          "<html><body>Hello, World!</body></html>"))

(setf (ningle:route *app* "/" :accept "text/plain")
      #'(lambda (params)
          (declare (ignore params))
          "Hello, World!"))
```

You can easily define your own conditions:

```common-lisp
(setf (ningle:requirement *app* :probability)
      #'(lambda (value)
          (<= (random 100) value)))

(setf (ningle:route *app* "/win_a_car" :probability 10)
      #'(lambda (params)
          (declare (ignore params))
          "You won!"))

(setf (ningle:route *app* "/win_a_car")
      #'(lambda (params)
          (declare (ignore params))
          "Sorry, you lost."))
```

### Request

`ningle` provides a special variable [`*request*`](https://github.com/fukamachi/ningle/blob/master/context.lisp#L19) which will be bound to an instance of [`lack.request:request`](https://github.com/fukamachi/lack/blob/master/src/request.lisp#L45) for each request. Mostly you will need to read values from this structure. Take a look at the structure definition to see where to access which information.

Here some usage examples
```common-lisp
;; read body-parameters
(lack.request:request-body-parameters *request*)
;; => #<CIRCULAR-INPUT-STREAM {1009BC8933}>
```
```common-lisp
(lack.request:request-cookies *request*)
;; => ((session 388934395))
```

### Response

`ningle` provides a special variable [`*response*`](https://github.com/fukamachi/ningle/blob/master/context.lisp#L23) which will be bound to an instance of [`lack.response:response`](https://github.com/fukamachi/lack/blob/master/src/response.lisp#L19) for each request. The status-code is [200 by default](https://github.com/fukamachi/ningle/blob/master/context.lisp#L35). If you manipulate `*response*`, make sure to NOT return a cons in your route. Take a look at the structure definition to see where to put which information.

Processing of route return value depending on type:

- CONS
  - result will be used as [normal lack response](https://github.com/fukamachi/lack?tab=readme-ov-file#normal-response)
  - manipulations to `*response*` will be silently ignored and not be reflected in the sent http-response!

- NULL
  - `*response*` will be finalized with `lack.response:finalize-response`

- OTHERWISE (neither cons, nor null)
  - result will be set as value for `response-body` slot in `*response*`
  - `*response*` will be finalized with `lack.response:finalize-response`

You can for example set some headers
```common-lisp
(setf (lack.response:response-headers *response*)
      (append (lack.response:response-headers *response*)
              (list :content-type "application/json")))

(setf (lack.response:response-headers *response*)
      (append (lack.response:response-headers *response*)
              (list :access-control-allow-origin "*")))

(setf (lack.response:response-status *response*) 201)
```
Then you can set the json string either via `lack.response:response-body`
```common-lisp
(setf (lack.response:response-body *response*)
      "[{\"test-response\":true}]")
```

or by suppling the json as result of the route

```common-lisp
"[{\"test-response\":true}]"
```
Note: the value supplied as result of the route will replace the value set
via setf'ing the lack.response:response-body.

### Context

ningle provides a useful function named `context`. It is an accessor to an internal hash table.

```common-lisp
(setf (context :database)
      (dbi:connect :mysql
                   :database-name "test-db"
                   :username "nobody"
                   :password "nobody"))

(context :database)
;;=> #<DBD.MYSQL:<DBD-MYSQL-CONNECTION> #x3020013D1C6D>
```

### Using Session

ningle doesn't provide Session system in the core, but recommends to use [Lack.Middleware.Session](https://github.com/fukamachi/lack/blob/master/src/middleware/session.lisp#L20) with [Lack.Builder](https://github.com/fukamachi/lack/blob/master/src/builder.lisp#L62).

```common-lisp
(import 'lack.builder:builder)

(clack:clackup
  (builder
    :session
    *app*))
```

Of course, you can use other Lack Middlewares with ningle.

## See Also

* [Clack](http://clacklisp.org/)
* [Lack](https://github.com/fukamachi/lack)

## Author

* Eitaro Fukamachi (e.arrows@gmail.com)

## Copyright

Copyright (c) 2012-2014 Eitaro Fukamachi (e.arrows@gmail.com)

## License

Licensed under the LLGPL License.
