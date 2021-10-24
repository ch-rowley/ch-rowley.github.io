Flask's advantage is how lightweight it is: nothing has to be installed that isn't needed. Instead, a significant part of learning the framework involves picking and choosing between the many extensions and libraries that are available. More opinionated frameworks like Django can actually be easier, in so far as some of these choices are made for you, removing the need to invest so much effort in weighing up alternatives. This article is intended to reduce the time required to learn one aspect of Flask, by reviewing the options for marshalling data. 

### What is data marshalling?

The growth of Single Page Applications has meant that communication with a separate frontend has become more important than rendering HTML templates using engines like Jinja2.

It is now normal for the backend to provide an API that returns JSON to be consumed by a frontend which has no direct control over how the server side works. The data comes from database queries, with your code acting as a bridge between the database and the JSON output:

![Frontend sends JSON to the controller, and the controller loads and dumps database objects.](/assets/images/serialiser_diagram.svg)

In Flask, sending JSON back from endpoints is easy. All that is needed is to return a dictionary (and an appropriate HTTP status code if other than the default 200 OK), and the framework will automatically handle it for you: 

```python
@app.route("/simple")
def simple_controller():
    return {"message": "here is some JSON data"}
```
When dealing with complex data, putting each value into a dictionary becomes verbose. Let's say we have a web application that manages people's personal details. The controller for returning information could end up looking like this:

```python
@app.route("/complex/<int:person_id>")
def complex_controller(person_id):
    person = Person.query.get(person_id)
    return {"name": person.name, "age": person.age, "phone": person.phone, "email": person.email}
```

The repetition becomes difficult to read, and will break whenever we change the model. For instance, we will get an *AttributeError* if we change the name of the 'phone' field to 'mobile' in the model but not the controller (since it will try to access a field that no longer exists). Similarly, if we add an address field to the model, it won't show up in the output until we manually add it to the controller. 

What is needed is a way to automate the translation of the database objects into the JSON that we send back to the frontend. A possibility that may occur to you is to dynamically create a dictionary to return each field from the Person class. This will still require us to keep a list of the field names, but should be easier to maintain as we update the model:

```python
@app.route("/diy/<int:person_id>")
def diy_serialiser(person_id):
    person = Person.query.get(person_id)
    fields = ["name", "age", "phone", "email"]
    return {field: person.__dict__[field] for field in fields}
```

If this feels like reinventing the wheel, it is: this is the purpose of a *data marshalling* library. The library will take the database object and arrange it field by field into a suitable JSON response based on the blueprint it has been given. This is called *serialising* the data. 

### Flask-RESTful

When constructing APIs with Flask, one option is to let an extension called Flask-RESTful handle the endpoints for you. Instead of functions for each endpoint with the HTTP verbs set by a decorator, you can use a class for each endpoint, and a method for each verb.

To repeat the example from the official docs:

```bash
pip install flask-restful
```

```python
from flask import Flask
from flask_restful import Resource, Api

app = Flask(__name__)
api = Api(app)

class HelloWorld(Resource):
    def get(self):
        return {'hello': 'world'}

api.add_resource(HelloWorld, '/')

if __name__ == '__main__':
    app.run(debug=True)
```

Whether it is worth using Flask-RESTful [is debated](#point_3) between Flask developers. It is largely a matter of whether you prefer using class methods for each HTTP verb (which I do, personally) or the simplicity of decorated functions. 

A strong reason to use Flask-RESTX instead (a fork of the original Flask-RESTful), is the automatic swagger support that it provides. Swagger is a way of documenting an API: it will look at your endpoints and produce information on how to use them. Whether you need this will depend on the nature of your project: it is very helpful for a large, public facing API, but less necessary for a small project where the endpoints are only utilised by a SPA that you have written yourself.  

### Marshalling and parsing

Flask-RESTful comes with inbuilt tools for parsing and marshalling data.  The two directions that the data can travel in are treated separately: the *marshall_with* decorator encodes data to send back to the user as JSON, while the *RequestParser* is used to create new objects with the JSON received from the frontend. 

The principle is the same for both: create an outline of what the data should look like (the names of the fields and their types), and then apply this outline to the JSON or the database objects as appropriate.

Let's say we have a web application for managing hotels that uses this model:

```python
class Hotel(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String, nullable=False)
    state = db.Column(db.String, nullable=False)
    rooms = db.Column(db.Integer, nullable=False)
```
We can then create this list of fields to serialise:

```python
from flask_restful import Api, Resource, fields, marshal_with, reqparse 


resource_fields = {
    "id": fields.Integer,
    "name": fields.String,
    "state": fields.String,
    "rooms": fields.Integer,
}
```

And tell the GET method to use these fields with the *marshall_with* decorator:

```python
class HotelAPI(Resource):
    @marshal_with(resource_fields)
    def get(self, hotel_id):
        hotel = Hotel.query.get(hotel_id)
        return hotel

api.add_resource(HotelsAPI, '/hotels', '/hotels/<int:hotel_id>')
```


This is the sort of response that a GET request to the endpoint would receive back (using HTTPie to query a Flask application running locally on port 5000):

```bash
~$ http :5000/hotels/1

HTTP/1.0 200 OK
Content-Length: 87
Content-Type: application/json
Date: Sat, 16 Oct 2021 09:10:49 GMT
Server: Werkzeug/2.0.1 Python/3.7.3

{
    "id": 1,
    "name": "The Overlook",
    "rooms": 237,
    "state": "Colorado"

}
```

To go in the other direction and create a new hotel, we have to create a parser:

```python
parser = reqparse.RequestParser()
parser.add_argument("name", type=str)
parser.add_argument("state", type=str)
parser.add_argument("rooms", type=int)
```

We can now use the parser's *parse_args* method to convert the JSON from the POST request into a Python dictionary. We then unpack the dictionary to create a new instance of the Hotel class, before adding and committing it to the database:

```python
    def post(self):                                                                      
        args = parser.parse_args()                                                       
        new_hotel_object = Hotel(**args)                                                 
        db.session.add(new_hotel_object)                                                 
        db.session.commit()                                                              
        return {"hotel_id": new_hotel_object.id}, 201  
```

This is the response we get after successfully creating a new hotel:

```bash
~$ http post :5000/hotels name="The Dolphin" state="New York" rooms="1408"

HTTP/1.0 201 CREATED
Content-Length: 22
Content-Type: application/json
Date: Sat, 16 Oct 2021 10:16:20 GMT
Server: Werkzeug/2.0.1 Python/3.7.3

{
    "hotel_id": 2

}
```

You may have noticed the main drawback: the serialiser and parser are likely to closely resemble the SQLAlchemy models, effectively duplicating the same structure three times.  First we write out the type of each field in the model, write them out again for resource fields, and then again for the parser.

An alternative called Marshmallow can help eliminate this duplication.  

### Marshmallow

Flask-RESTful has a notice in its documentation that its own request parsing features will eventually be deprecated, and recommends that its users then use Marshmallow instead. 

Marshmallow is a play on words, referring to how it *marshals* data. To do this, it requires that we define schemas, which are outlines of what the data should look like. We then create instances of the schema, which can translate back and forth between JSON and Python datatypes.  Converting to JSON is *dumping*, and from JSON is *loading*. 

We can define a schema for the Hotel model like this:

```bash
pip install marshmallow
```
```python
from marshmallow import Schema, fields 


class HotelSchema(Schema):                                                               
    id = fields.Int()                                                                    
    name = fields.Str()                                                                  
    state = fields.Str()                                                                 
    rooms = fields.Int() 
```

We then create an instance of the schema:

```python
hotel_schema = HotelSchema() 
```

And use it to dump the object that we retrieve from the database:

```python
class HotelsAPI(Resource):                                                               
    def get(self, hotel_id):                                                             
        hotel = Hotel.query.get(hotel_id)                                                
        return hotel_schema.dump(hotel)
```

As you can see, there is a strong similarity between the schema and Flask-RESTful's resource fields. An import difference however, is that we can use the same Marshmallow schema to get objects both into and out of our application. Loading the JSON returns a Python dictionary, which we can unpack to create a new instance of the Hotel class:

```python
    def post(self):                                                                      
        hotel_dict = hotel_schema.load(request.get_json())
        new_hotel_object = Hotel(**hotel_dict)
        return {"hotel_id": new_hotel_object.id}, 201
```

### Plain Marshmallow vs Flask flavour

Once you have decided to use Marshmallow, there is still another choice to be made between the standard Marshmallow library, and the modified forms specifically designed for Flask web applications. 

The first package is Marshmallow-SQLAlchemy. To use it on its own, we can just import its special schema classes:

```bash
pip install marshmallow-sqlalchemy
```

```python
from marshmallow_sqlalchemy import SQLALchemySchema SQLAlchemyAutoschema
```

It is though more common to use it in combination with the second package, Flask-Marshmallow. This adds additional features that are sometimes useful in building APIs (like URL fields) and also makes interaction with Flask-SQLAlchemy (the modified version of SQLAlchemy designed to work with Flask) a little easier.

Setting up Flask-Marshmallow means registering an instance with the Flask app, similar to the way that we register a Flask-SQLAlchemy database:


```bash
pip install flask-marshmallow
```

```python
from flask_marshmallow import Marshmallow

ma = Marshmallow()
ma.init_app(app) 
```

Refer to this instance for the fields and schemas that we would have previously imported from Marshmallow or Marshmallow-SQLAlchemy, such as *ma.Int* (instead of *fields.Int*), *ma.Schema*, or *ma.SQLAlchemySchema*.

For most applications, it is quicker and easier to use both of these integrated packages.  A few writers [do advocate](#point_4) using plain SQLAlchemy instead of Flask-SQLAlchemy for the reason that it makes the database more convenient to access outside of a Flask environment.  Even if you do decide that you want to do this, Marshmallow-SQLAlchemy works with plain SQLALchemy and you can continue to use it after dropping Flask-Marshmallow and Flask-SQLAlchemy from your project.

### AutoSchema

While having a schema that both loads and dumps has reduced duplication, we still have to write out the same fields that we already have in our models (such as that the hotel name should be a string). AutoSchema takes care of this for us:

```python
class HotelSchema(ma.SQLAlchemyAutoSchema):

    class Meta:
        model = Hotel:
        load_instance = True
```

This tells the schema to use the fields from the Hotel model. Everything will work just as if we had manually typed out each field, and will not need to be adjusted when we change the model in the future.

Setting *load_instance* to true means that JSON can now be loaded directly as SQLAlchemy objects:

```python
new_hotel_object = hotel_schema.load(request.get_json())                     
db.session.add(new_hotel_object)                                                 
```

This removes the need for the intermediate step of loading the JSON into dictionaries, and then unpacking those dictionaries into a SQLAlchemy model.

### Validation errors

We've seen how we can use Marshmallow schemas to create new objects from JSON, but what if the user supplies incomplete or wrong information? The schema validates the data that it tries to load and will raise an exception if it isn't what was expected. Unless it is handled, this will cause an internal server error. 

We'll need to use a try/except block:

```python
from marshmallow import ValidationError

class HotelsAPI(Resource):

    def post(self):
        try:                                                                         
            new_hotel_object = hotel_schema.load(request.get_json())
        except ValidationError as err:
            return err.messages, 422
                          
        db.session.add(new_hotel_object)
        db.session.commit()
        return {"hotel_id": new_hotel_object.id}, 201
```

Now consumers of our API will receive an informative message if they send incompatible data:

```bash
~$ http post :5000/hotels name="The Great Northern" state="Washington" rooms="Three Hundred and Fifteen"
            
HTTP/1.0 422 UNPROCESSABLE ENTITY
Content-Length: 56
Content-Type: application/json
Date: Sat, 16 Oct 2021 13:36:45 GMT
Server: Werkzeug/2.0.1 Python/3.7.3

{
	"rooms": [
        "Not a valid integer."
    
	]

}
```

As you can see, the automatic schema validation points out exactly which fields are causing the error, and why.

### Webargs

Webargs is another library that has the same purpose as the parser that comes with Flask-RESTful: convert request data to objects that our projects can use. It makes it easy to access input whether it comes from the JSON in a POST request, the headers, or the query string.

The *use_args* decorator allows us to define the fields in a similar way to what we have seen before:


```python
    @use_args(
        {
            "name": fields.Str(required=True),
            "state": fields.Str(required=True),
            "rooms": fields.Int(required=True),
        },
        location="json",
    )
```

While this would again lead to duplication with the Hotel model, Webargs also integrates perfectly with Marshmallow and can be passed a schema instead of having to list each field. The endpoint controllers are also simplified because returning an error when validation fails is now automatic, and a try/except block is no longer necessary:

```bash
pip install webargs
```

```python
from webargs.flaskparser import use_args

class HotelsAPI(Resource):
    @use_args(HotelSchema, location="json")
    def post(self, new_hotel_object):
        db.session.add(new_hotel_object)
        db.session.commit()
        return {"hotel_id": new_hotel_object.id} , 201
```

While this certainly looks better, it's worth noting that as boilerplate is eliminated from the controllers, some more pops up elsewhere. In the earlier example where we used Marshmallow with a try/except block, the validation error that we returned was in JSON format, as would be expected from an API. The default Webargs response for failed validation is HTML, and to use JSON we'll need to cut and paste an additional error handler function.

This is the default error we will get:

```bash
HTTP/1.0 422 UNPROCESSABLE ENTITY
Content-Length: 215
Content-Type: text/html; charset=utf-8
Date: Sat, 16 Oct 2021 16:00:52 GMT
Server: Werkzeug/2.0.1 Python/3.7.3

<!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 3.2 Final//EN">                                 
<title>422 Unprocessable Entity</title>
<h1>Unprocessable Entity</h1>
<p>The request was well-formed but was unable to be followed due to semantic errors.</p>
```

To change it, we need to include this code from the Webargs documentation:

```python
from flask import jsonify


# Return validation errors as JSON
@app.errorhandler(422)
@app.errorhandler(400)
def handle_error(err):
    headers = err.data.get("headers", None)
    messages = err.data.get("messages", ["Invalid request."])
    if headers:
        return jsonify({"errors": messages}), err.code, headers
    else:
        return jsonify({"errors": messages}), err.code
```

The messages will now be similar (although more deeply nested) to those returned by Marshmallow's *ValidationError*:

```bash
HTTP/1.0 422 UNPROCESSABLE ENTITY
Content-Length: 98
Content-Type: application/json
Date: Sat, 16 Oct 2021 16:07:17 GMT
Server: Werkzeug/2.0.1 Python/3.7.3

{
	"errors": {
		"json": {
			"rooms": [
                "Not a valid integer."
            
			]
        
		}
    
	}

}

```

Using Webargs makes sense if your project has a sufficient number of POST endpoints that it is worth the extra dependency and setup, but it isn't essential.

### Alternative Serialisation Libraries

While Marshmallow is the most used serialiser for Flask APIs, there are a number of alternatives. The advantage they generally offer over Marshmallow is speed. One is Toasted Marshmallow, which can run several times faster than Marshmallow. Another alternative is Serpy which was designed to be very fast, and came top of a benchmark comparison.

Serialisation speed will not concern everyone who is building web applications with Flask, and there are benefits in staying with a mature project like Marshmallow, not least the amount of support available from other developers. It is though helpful to know that there are alternatives if speed becomes an issue.

### Summary

These are my recommendations for a light Flask application: 

 - Use Flask-RESTX if you want to have automatic swagger documentation for your endpoints. If this isn't a concern, use Flask-RESTful or decorated functions, depending on whether you prefer a class based approach.

 - Whichever way you decide, avoid Flask-RESTful/X's built in serialiser.  Use the Flask and SQLALchemy integrated packages instead of plain Marshmallow, and take advantage of the simplicity of AutoSchemas.  

 - Start by using Marshmallow with a try/except block to load JSON input, and introduce Webargs later if it seems justified by the increasing complexity of your project.

Hopefully, this overview will let you build your Flask API more quickly, without the time and effort of first trying out a range of alternatives.

### Quick-start

To set up a new project with a selection of packages:

`~$ pip install Flask Flask-RESTX Flask-SQLAlchemy Marshmallow-SQLAlchemy Flask-Marshmallow`

### Resources

1. The GET and POST queries to a locally running Flask app were made using [HTTPie](https://httpie.io) in the Bash terminal. 

2. Here are the official docs for [Flask-RESTful](https://flask-restful.readthedocs.io/en/latest/) and its swagger supporting fork, [Flask-RESTX](https://flask-restx.readthedocs.io/en/latest/).

   It is also possible to use a class based approach without Flask-RESTful, by [subclassing MethodView](https://flask.palletsprojects.com/en/2.0.x/views/). A guide to doing this is [available here](https://pythonise.com/series/learning-flask/flask-api-methodview).

3. <a id="point_3">[Here is an example](https://www.reddit.com/r/flask/comments/3nuxxt/flask_restful_designers_which_approach_do_you/)</a> of a discussion about whether to use class methods or endpoint functions. Prominent Flask contributor Miguel Grinberg states that he prefers functions to Flask-RESTful.

4. Adding `/hotels` and `/hotels/<int:hotel_id>` to the API means that calls to both endpoints will be handled by the same class. We need the `/hotels` endpoint so we can POST new hotels. In order to be fully functional, the GET method would need to be extended to return all hotels when an ID number is not provided.

   For a helpful book that covers Flask development, including an API made with Flask-RESTful, see: [Mastering Flask Web Development, 2nd ed.](https://www.packtpub.com/product/mastering-flask-web-development-second-edition/9781788995405)

5. See <a id="point_4">[this article](https://towardsdatascience.com/use-flask-and-sqlalchemy-not-flask-sqlalchemy-5a64fafe22a4)</a> for an argument in favour of using plain SQLAlchemy instead of Flask-SQLAlchemy.

   Flask-Marshmallow simplifies interactions with the database session. If you want to use Marshmallow-SQLAlchemy without Flask-Marshmallow, add `transient = True` to the AutoSchema's *Meta* class to create objects that are not connected to a session.

6. The official docs for [Marshmallow](https://marshmallow.readthedocs.io/en/stable/), [Marshmallow-SQLAlchemy](https://marshmallow-sqlalchemy.readthedocs.io/en/latest/), and [Flask-Marshmallow](https://flask-marshmallow.readthedocs.io/en/latest/).

   Marshmallow has a [broad range of additional features](https://marshmallow.readthedocs.io/en/stable/extending.html), including the pre-processing and post-processing of data, as well as custom validation.

7. Two helpful articles that provide guides to using Marshmallow to serialise and validate data are available [here](https://kimsereylam.com/python/2019/10/25/serialization-with-marshmallow.html) and [here](https://oluchiorji.com/object-serialization-with-marshmallow/). 

8. Here are the [webargs docs](https://webargs.readthedocs.io/en/latest/). The section on returning errors as JSON is [here](https://webargs.readthedocs.io/en/latest/framework_support.html#error-handling). 

9. [Here is a link](https://voidfiles.github.io/python-serialization-benchmark/) to a benchmark test of the speed of various serialisers.

   The documentation for [Serpy: ridiculously fast object serialization](https://serpy.readthedocs.io/en/latest/).

   The project page for [Toasted Marshmallow](https://github.com/lyft/toasted-marshmallow), and a [short article discussing how to use it](https://eng.lyft.com/toasted-marshmallow-marshmallow-but-15x-faster-17bdcf34c760).
