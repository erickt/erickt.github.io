---
layout: post
title: "Meal, Ready-to-Eat: A web framework for Rust"
date: 2012-07-05 08:55
comments: true
categories: [rust, mre]
---

I've been putting this off for far too long. For the last three months I've
been working on [Meal, Ready-to-Eat](https://github.com/erickt/mre), a web
framework for the [Rust programming language](http://rust-lang.org). I call it
MRE. Rust didn't have a TCP stack when I started the project, so instead I
built MRE on top of [Mongrel2](http://mongrel2.org). It talks
[Zeromq](http://zeromq.org), so I could get something up pretty quickly. It's
inspired by [Sinatra](http://www.sinatrarb.com/) and
[Express.js](http://expressjs.com/).  So take all this code with a grain of
salt. The design is very much in flux and there are some pretty rough edges.
Better to release early and get feedback though, right?

One word of warning though. Rust's syntax and semantics are still in flux, so
it's quite possible this blog post will be out of date by the time you read it.

Hello World
-----------

Let's start with the classic Hello World app. You can find the full example
[here](https://github.com/erickt/mre/blob/master/examples/helloworld/helloworld.rs).
This example is a little more verbose than frameworks like Sinatra and Express,
and not just because Rust is statically typed. Those other frameworks take
advantage of global variables and static initializers, but Rust doesn't, so we
have to make due with some boilerplate code.

```
let mre = mre::mre(
		// Create a zeromq context that MRE will use to talk to Mongrel2.
		alt zmq::init(1) {
			ok(ctx) { ctx }
			err(e) { fail e.to_str() }
		},

		// A UUID for this Mongrel2 backend.
		some("E4B7CE14-E7F7-43EE-A3E6-DB7B0A0C106F"),

		// The addresses to receive requests from.
		~["tcp://127.0.0.1:9996"],

		// The addresses to send responses to.
		~["tcp://127.0.0.1:9997"],

		// Create our middleware, which preproceses requests and
		// responses. For now we'll just use the logger.
		~[mre::middleware::logger(io::stdout())],

		// A function to create per-request data. This can be used by
		// middleware like middleware::session to automatically look
		// up the current user and session data in the database. We don't
		// need it for this example, so just return a unit value.
		|| ()
);
```

Eventually I would like to pull the Mongrel2 settings out into a separate
config file, so it should get a little more slim in the future. Once we have an
`mre` value, we can define some routes. 

```
do mre.get("^/$") |_req, rep, _m| {
		rep.reply_html(200u,
				"<html>\n" +
				"<body>\n" +
				"<h1>Hello world!</h1>\n" + 
				"</body>\n" +
				"</html>")
}
```

Routes are defined much like Sinatra. You'll find helpers for all the HTTP/1.1
methods. These method handlers take two arguments. The first is a PCRE regular
expression, which may have capture clauses, the second a response handler
closure. Whenever a request comes for a path that accesses this matching
handler, the closure will be called with a `mre::request`, `mre::response`, and
the regex match object. `mre::request` values, obviously, contain all the data
relevant for a given request. Most important the headers and the body.
`mre::response` values handle sending responses back to the client.

Finally, we start the MRE event loop, and we're off.

```
mre.run();
```

Models
------

MRE also comes with a basic database support, built on top of
[Elasticsearch](http://elasticsearch.org). Sure it's technically a a fulltext
search engine, but it also works quite well as a JSON object store. Plus,
there's a [Zeromq plugin](https://github.com/tlrx/transport-zeromq), so it was
pretty easy to plug it into MRE.  The plugin can be a bit of a pain to set up,
however, so I wrote up some directions for that
[here](https://github.com/erickt/rust-elasticsearch).

Let's rewrite our Hello World app to be a bit more interactive. Rather than
just saying Hello World, let's greet anyone who asks (Source is
[here](https://github.com/erickt/mre/blob/master/examples/helloeveryone)).
Before we begin with the MRE code, we need to create our Elasticsearch index:

```
curl -XPOST "http://localhost:9200/helloeveryone" -d '{
  "settings": {
    "index.number_of_shards": 1,
    "index.number_of_replicas": 0
  },
  "mappings": {
    "person": {
      "properties": {
        "timestamp": {"type": "date", "index": "not_analyzed"},
        "name": {"type": "string", "index": "not_analyzed"}
      }
    }
  }
}'
```

Next, lets make a model of all the people we'll greet. At it's heart, a
model is just a JSON object with some helper functions. Unfortunately Rust
still has some ways to go before we can write really clean models. There is no
support for inheritance or mixin classes, so we need to duplicate some code in
all the models. Also, our constructors are not that featureful. We don't
support mulitple constructors, nor is there a way to make a constructor
private. Fortunately we can hack our way to the API we want.

So enough preamble, lets see this in action. Below is our `person` model:

```
class _person {
    let model: model;

    new(model: model) {
        self.model = model;
    }

    fn id() -> @str {
        self.model._id
    }

    fn timestamp() -> @str {
        self.model.get_str("timestamp")
    }

    fn set_timestamp(timestamp: @str) -> bool {
        self.model.set_str("timestamp", timestamp)
    }

    fn name() -> @str {
        self.model.get_str("name")
    }

    fn set_name(name: @str) -> bool {
        self.model.set_str("name", name)
    }

    fn create() -> result<(), error> {
        self.model.create()
    }

    fn save() -> result<(), error> {
        self.model.save()
    }

    fn delete() {
        self.model.delete()
    }
}

type person = _person;
```

In order to work around not having private constructors, we create a class
called `_person`, which is then aliased to `person`. If we don't export
`_person`, then our constructor is effectively hidden.

Next, here's how to create and find the `person` models:

```
// Create a new person model.
fn person(es: client, name: @str) -> person {
    // Create a person. We'll store the model in the ES index named
    // "helloeveryone", under the type "person". We'd like ES to make the index
    // for us, so we leave the id blank.
    let person = _person(model(es, @"helloeveryone", @"person", @""));

    person.set_name(name);
    person.set_timestamp(@time::now().rfc3339());

    person
}

// Return the last 50 people we have said hello to.
fn last_50(es: client) -> [person] {
    // This query can be a little complicated for those who have never used
    // elasticsearch. All it says is that we want to fetch 50 documents on the
    // index "helloeveryone" and the type "person", sorted by time.
    do model::search(es) |bld| {
        bld
            .set_indices(~["helloeveryone"])
            .set_types(~["person"])
            .set_source(*json_dict_builder()
                .insert("size", 50.0)
                .insert_list("sort", |bld|
                    bld.push_dict(|bld|
                        bld.insert("timestamp", "desc");
                    });
                })
            );
    }.map(|model|
        // Construct a person model from the raw model data.
        _person(model)
    )
}
```

Here, since `person` is just a type alias, we can also create a function called
`person`. The underlying `_person` constructor then can be shared with
multiple functions, which lets us simulate having multiple constructors. So,
the users of the model have a clean api, which is exactly what we want.

We're almost done, so lets finish up and tie everything together in our `main`:

```
fn main() {
    // Create a zeromq context that MRE will use to talk to Mongrel2 and
    // Elasticsearch.
    let zmq = alt zmq::init(1) {
        ok(ctx) { ctx }
        err(e) { fail e.to_str() }
    };

    let mre = mre::mre(zmq, ...);

    // Connect to Elasticsearch, which we'll use as our database.
    let es = elasticsearch::connect_with_zmq(zmq, "tcp://localhost:9700");

    // Show who we'll say hello to.
    do mre.get("^/$") |_req, rep, _m| {
        // Fetch the people we've greeted.
        let people = person::last_50(es);

        // We want to render out our responses using mustache, so we need
        // to convert our model over to something mustache can handle.
        let template = mustache::render_file("index", hash_from_strs(~[
            ("names", do people.map |person| {
                hash_from_strs(~[
                    ("name", person.name())
                ])
            }.to_mustache())
        ]));
               
        rep.reply_html(200u, template)
    }

    // Add a new person to greet.
    do mre.post("^/$") |req, rep, _m| {
        // Parse the form data.
        let form = uri::decode_form_urlencoded(*req.body());

        alt form.find("name") {
          none {
            rep.reply_http(400u, "missing name");
          }
          some(names) {
            // Create and save our person. If successful, redirect back to
            // the front page.
            let person = person::person(es, (*names)[0u]);

            alt person.create() {
              ok(()) { rep.reply_redirect("/") }
              err(e) {
                // Uh oh, something bad happened. Let's just display the
                // error back to the user for now.
                rep.reply_http(500u, e.msg)
              }
            }
          }
        }
    }

    // Finally, start the MRE event loop.
    mre.run();
}
```

Middleware
----------

As you probably saw in the `mre::mre` constructor, MRE has some basic
support for middleware. Creating middleware is pretty easy. It's just a
function that matches this interface (That type variable matches the return
type for the closure passed in to `mre::mre`):

```
type middleware<T> = fn@(@request<T>, @response) -> bool;
```

Middleware gets called on each request in order, and is able to
read the request and it's headers, and modify the response hooks. Here's
`mre::middleware::logger`, to show how it works:

```
fn logger<T: copy>(logger: io::writer) -> middleware<T> {
    |req: @request<T>, rep: @response| {
        let old_end = rep.end;
        rep.end = || {
            let address = alt req.find_header("x-forwarded-for") {
              none { @"-" }
              some(address) { address }
            };

            let method = alt req.find_header("METHOD") {
              none { @"-" }
              some(method) { method }
            };

            let len = alt rep.find_header("Content-Length") {
              none { @"-" }
              some(len) { len }
            };

            logger.write_line(#fmt("%s - %s [%s] \"%s %s\" %u %s",
                *address,
                "-",
                time::now().strftime("%d/%m/%Y:%H:%M:%S %z"),
                *method,
                *req.path(),
                rep.code,
                *len));

            old_end();
        };

        true
    }
}
```

MRE also includes a `mre::middleware::session` middleware, which implements a
traditional cookie-based session authentication scheme. This one is
unfortunately a little more complicated to use. Starting off, you need to
create a new datatype to store the session data and give a constructor to `mre`
on how to make this per-request data:

```
type data = @{
    mut session: option<mre::session::session>,
    mut user: option<mre::user::user>,
};

let middleware = ~[
		mre::middleware::logger(io::stdout()),
		mre::middleware::session(es,
				@"blog",
				@"blog",
				@"session",
				|req: @request<data>, session, user| {
						req.data.session = some(session);
						req.data.user = some(user);
				}
		)
];

let mre = mre::mre(zmq,
		some("F0D32575-2ABB-4957-BC8B-12DAC8AFF13A"),
		~["tcp://127.0.0.1:9998"],
		~["tcp://127.0.0.1:9999"],
		middleware,
		|| @{ mut session: none, mut user: none }
);
```

Then you access this data through the `request.data` member:

```
do app.post("^/$") |req, rep, _m| {
    let id = alt req.user {
      none { @"world" }
      some(user) { user.id() }
    };

		rep.reply_html(200u,
				"<html>\n" +
        "<body>\n" +
        "<h1>Hello " + *id + "!</h1>\n" + 
        "</body>\n" +
        "</html>")
}
```

See the [blog](https://github.com/erickt/mre/tree/master/examples/blog) for a
complete example.
