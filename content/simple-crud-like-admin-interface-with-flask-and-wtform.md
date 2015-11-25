Title: Simple CRUD-like admin interface with Flask and WTForm
Date: 2014-08-02 07:36
Author: jibreel
Slug: simple-crud-like-admin-interface-with-flask-and-wtform

Without a doubt, many times over the course as a web developer, you will
sooner or later need to create admin interface for a specific project
that you have been working on.

</p>
One of the most common use case for an admin interface is to edit data
for your models.

</p>
To be able to create a new object, (like a blog post), edit an existing
object, delete it, or simply just to view it's content.

</p>
CRUD stands for CREATE, READ, UPDATE, DELETE.

When bulding REST API, the UPDATE and DELETE part is probably used with
the HTTP methods PUT, DELETE.

</p>
However, for a web page, we don't deal with PUT and DELETE, rather, just
GET and POST,

</p>
so we only have that to worry about, however we still have to implement
a full CRUD for our model in the admin interface.

There is already pre-packaged Admin interface extensions for different
frameworks, Django for example has a full blown admin app already
included without installing anything external, Flask has several
extensions you can install, the two most popular is probably Flask-Admin
and Flask-SuperAdmin.

The former is simple to use and allow for great flexibility to customize
it, the later is a "supervitamined fork of the Flask-Admin" as they put
it.

</p>
One thing all these extensions share in common is the fact that you can
creat a CRUD operation from any given model, built from some of the
popular ORMs(object relational model), like Django's ORM or the
SQLAlchemy's ORM which is framework independent.

</p>
In a nutshell you provide your model, it builds the CRUD operations and
interface for you.

</p>
Simple enough, however the more abstraction you have to deal with for
any given projects the harder it become to customize it, you will have
to look deep in the documentation to find a clue to a certain
customization to modify it and so on.

</p>
Granted they give you incredible speed to build in Admin interface, you
barely have to do a few tweaks and that's it, but the more you want to
customize it the more annoying it becomes to work with, so, many times,
you are better of with creating your own Admin interface.

</p>
For simple projects it's easy enough to hard code the admin interface,
however the more apps and models your projects have it's nearly
imposible to just hardcode every single CRUD operation.

</p>
In this small tutorial, we will go through on how to build a simple CRUD
object from scratch in flask that will work somehow similar to already
existing extensions (except much simpler with less abstraction), that
you can tailer for you own need for a any given project.

</p>
The main purpose for this is that you can easy dig in to the code and
change it's behavier radicaly if needed, without going through
abstraction layers.

</p>
Of cource we will build on some existing technologies like the perfect
WTForm to build our forms, SQLAlchemy to create our model, and flask for
it's simplicity and power.

</p>
We shall use this simple blog app to build an admin interface for.

Be sure you are runing a virtual environment so that you don't polute
you system by installing packages system wide, if you don't have it
already installed[read this for
instructions](http://docs.python-guide.org/en/latest/dev/virtualenvs/).

</p>
Create a folder called project, inside it app, and another folder inside
of it and call it admin

</p>
<div class="code">

    # all the dependencies you needpip install flaskpip install wtform # for the form creationpip install flask-sqlalchemy # for the database# you can install them all with one linepip install flask, wtform, flask-sqlalchemy

</div>

</p>
<div class="code">

    # project/app/__init__.pyfrom flask import Flask, render_template, request, sessionfrom flask.ext.sqlalchemy import SQLAlchemyapp = Flask('app')app.config.update(        DEBUG=True,        SQLALCHEMY_DATABASE_URI='sqlite:///../database.db'            )db = SQLAlchemy(app)@app.route('/')def home():    return 'Blog be here'# project/run.pyfrom app import appif __name__ == '__main__':    app.run(debug=True)

</div>

</p>
In the terminal `project/ ~$ python run.py`{.python}

</code> should run the development server

</p>
Now lets create a relativeley simple model.

create an empty \_\_init\_\_.py file in project/app/admin/

</p>
<div class="code">

    # project/app/admin/models.pyfrom datetime import datetimefrom .. import dbclass Blog(db.Model):    id = db.Column(db.Integer, primary_key=True)  # pretty self explanotory    title = db.Column(db.String(100), unique=True)    body = db.Column(db.Text())    created_on = db.Column(db.DateTime(), default=datetime.utcnow)    updated_on = db.Column(db.DateTime(), default=datetime.utcnow,                                          onupdate=datetime.utcnow)    def __init__(self, title="", body=""):        self.title = title        self.body = body        def __repr__(self):        return '<Blogpost - {}>'.format(self.title)

</div>

</p>
Your structure should now look like this.

</p>
    project├── app│   ├── admin│   │   ├── __init__.py│   │   ├── models.py│   └── __init__.py└── run.py

</p>
We have to initiate the database for the models that we have just
created Open a python shell from the project folder.

</p>
<div class="code">

    # the sqlalchemy db objectfrom app import db# we have to import the models so that sqlalchemy can detect them and create the db# how else would it know what to create ?from app.admin.models import *# creates the databasedb.create_all()# after the last command you should now be able to see database.db file# in the project folder

</div>

</p>
Ok, we now have a blog app, with the models created.

we have to actualy create some blog posts, and what better way to do
that other than through an admin interface? Let's get to business

Let's think for a moment on how we want the urls to be.

</p>
``` {.highlight}
/admin/blog/  [GET] return a list of all blog posts/admin/blog/  [POST] create a new the post /admin/blog/4/  [GET] display the form of a post by it's id/admin/blog/4/  [POST] should update the post of that id/admin/blog/new/  [GET] display an empty form based on the model/admin/blog/delete/4/  [GET] delete any given post by id
```

</p>
Lets get started by building them one by one, utilize a Blueprint for
the admin page, so that every view we build inside the admin.py becomes
url prefixed with /admin/.

<div class="code">

    # project/app/admin/admin.pyfrom flask import Blueprint, request, g, redirect, url_for, abort, \                  render_templateadmin = Blueprint('admin', __name__, template_folder='templates')# project/app/admin/__init__.py# so that we can import anything from admin folder# without the extra admin syntaxfrom admin import *# project/__init__.py# bellow line "db = SQLAlchemy(app)"from admin import admin # the Blueprint instance that we createdapp.register_blueprint(admin, url_prefix='/admin')

</div>

</p>
Blueprints are great way to split apps in different folders like in this
case the admin folder inside of of we place every thing relevant to the
admin interface that we are creating.

So if you have a very large application creating it thus would make it
manageable and modular.

</p>
Now we are ready for the CRUD operations.

</p>
<div class="code">

    # project/app/admin/admin.py# We make use of flask Methodview# it's a class that we can inherit from and define a get and post methods# inside of it and the url dispatcher will automaticly link to relevant http methodfrom flask.views import MethodView# the model to we want to administerfrom .models import Blog# our crud class that we will continiously expand on# and later use for any given model that we defineclass CRUDView(MethodView):  # default template that we will use to display a list    # of all items in a model    list_template = 'admin/list_view.html'    def __init__(self, model, endpoint, list_template=None):        self.model = model        self.endpoint = endpoint        # so we can generate a url relevant to this        # endpoint, for example if we utilize this CRUD object        # to enpoint comments the path generated will be        # /admin/comments/        self.path = url_for('.%s' % self.endpoint)        if list_template:            self.list_template = list_template    # all GET methods will link to this function    def get():        obj = self.model.query.all()        return render_template(list_template, obj=obj, path=self.path) # this turns the class to a function that flask can use for requests# the argument it takes is the endpoint that you can call in url_for()# since we are using a blueprint called admin the endpoint becomes# in this case 'admin.blog' or simply '.blog'# the '.' links to the current app(which is 'admin')view = CRUDView.as_view('blog')# now we just add the url ruleadmin.add_url_rule('/blog/', view_func=view, methods=['GET', 'POST'])

</div>

</p>
If we try to `python run.py`{.python}

</code> it wouldn't work, becaouse we havent created a template yet :(

</p>
The template bellow simply creates a <span
class="highlight-blue">CREATE</span> button

then loops over every item in the obj and displays it along with a <span
class="highlight-red">DELETE</span> button

</p>
<div class="code">

    <!-- project/app/admin/templates/admin/listview.html --><!DOCTYPE html><html><head>    <title>Admin</title>    <link rel="stylesheet" href="http://yui.yahooapis.com/pure/0.5.0/pure-min.css">    <style>        .button {          color: white;          border-radius: 4px;          text-shadow: 0 1px 1px rgba(0, 0, 0, 0.2);        }        .button.red {          background: #ca3c3c;        }        .button.blue {          background: #3caeca;        }        .listview > a {          margin: 10px 0 10px 20px;        }        .listview .pure-g {          margin: 2px auto;          padding: 10px 0 10px 20px;          background: #f4f6ff;        }        .listview .pure-u {          overflow: hidden;      }    </style></head><body>    <div class="listview">    <a class="pure-button pure-button-primary" href="{{path + 'new'}}">Create new</a>    {% for item in obj %}        <div class="pure-g">            <div class="pure-u-2-24">            <a class="pure-button pure-button-primary button red" href="{{path +'delete/'+ item.id|string}}">delete</a></div>            <div class="pure-u"><a class="pure-button pure-button-primary button blue" href="{{path + item.id|string}}">{{item}}</a></div>        </div>    {% endfor %}    </div></body></html>

</div>

</p>
`python run.py`{.python}

</code> renders the template and works as expected.

</p>
But wait, all i see is a "Create New" button :(

Well that's because our database is still empty.

we will have to create some post to be able to see them, you could do so
in the command line.

</p>
<div class="code">

    from app import dbfrom app.admin.models import Bloga = Blog('my first post', 'some looooong body')b = Blog('my second post', 'some short body')c = Blog('my third post', 'bla bla bla')db.session.add(a)db.session.add(b)db.session.add(c)db.session.commit()

</div>

</p>
Hit refresh and you should see a nice list of all the blog posts.

Since We want to be able to create post in the admin interface.

lets make a new url <span class="highlight">/admin/blog/blog\_id/</span>
we should be able to do a GET method and retrieve a blog form.

</p>
<div class="code">

    # project/app/admin/admin.py# add these two importsfrom wtforms.ext.sqlalchemy.orm import model_formfrom app import db# add this new url rule bellow the previes ruleadmin.add_url_rule('/blog/<int:obj_id>/', view_func=view,                   methods=['GET', 'POST'])# in the CRUDView add this bellow list_templatedetail_template = 'admin/detailview.html'# modify our get method to look like thisdef get(self, obj_id=''):    if obj_id:        # this creates the form fields base on the model        # so we don't have to do them one by one        ObjForm = model_form(self.model, db.session)        obj = self.model.query.get(obj_id)        # populate the form with our blog data        form = ObjForm(obj=obj)        # action is the url that we will later use        # to do post, the same url in this case        action = request.path        return render_template(self.detail_template, form=form,                               path=self.path, action=action)    obj = self.model.query.order_by(self.model.created_on.desc()).all()    return render_template(self.list_template, obj=obj, path=self.path)

</div>

</p>
The template of course...

</p>
<div class="code">

    <!-- project/app/admin/templates/admin/detailview.html  --><!DOCTYPE html><html><head>    <title>Admin</title>    <link rel="stylesheet" href="http://yui.yahooapis.com/pure/0.5.0/pure-min.css">    <style>        .button {          color: white;          border-radius: 4px;          text-shadow: 0 1px 1px rgba(0, 0, 0, 0.2);        }        .button.red {          background: #ca3c3c;        }        .button.blue {          background: #3caeca;        }        .item-form .pure-form {          width: 90%;          margin: 40px auto;        }        .item-form .pure-form legend {          color: #6b6b6b;          font-size: 1.2em;        }        .item-form .pure-form label {          margin-top: 30px;          color: #a5a5a5;        }        .item-form .pure-form textarea {          min-height: 200px;          padding: 10px;        }        .item-form .pure-form input, .item-form .pure-form select, .item-form .pure-form textarea {          width: 100%;        }        .item-form .pure-form input[type="checkbox"] {          width: 50px;        }        .item-form .pure-form button {          margin: 30px 0;        }    </style></head><body>    <div class="item-form">             <form class="pure-form pure-form-stacked"              action="{{action}}"              method="post">            <fieldset>                <legend>Item View</legend>                                {% for field in form %}                    {{field.label()}}                    {{field(class="pure-input")}}                                    {% endfor %}                <button type="submit" class="pure-button pure-button-primary pure-input-1-4">Submit</button>                <a class="pure-button pure-button-primary button red" href="{{ path +'delete/'+ action.split('/')[-2]}}">delete</a></div>            </fieldset>        </form>     </div></body></html>

</div>

</p>
This is how it should look like

</p>
[![screenshot](https://s3-us-west-2.amazonaws.com/jibreel/blog/list_blog_1.png)](https://s3-us-west-2.amazonaws.com/jibreel/blog/list_blog_1.png)

Clicking on ony of the blog post in the list should now display you a
nice form with all the data.

</p>
We have done the GET method, now for the POST part, where we actually
modify the form and submit it.

</p>
<div class="code">

    # project/app/admin/admin.py# in CRUDViewdef post(self, obj_id=''):    # either load an object to update if obj_id is given    # else initiate a new object, this will be helpfull    # when we want to create a new object instead of just    # editing existing one    if obj_id:        obj = self.model.query.get(obj_id)    else:        obj = self.model    ObjForm = model_form(self.model, db.session)    # populate the form with the request data    form = ObjForm(request.form)    # this actually populates the obj (the blog post)    # from the form, that we have populated from the request post    form.populate_obj(obj)    db.session.add(obj)    db.session.commit()        return redirect(self.path)

</div>

</p>
You should now be able to modify any existing blog post, easy right? :)

</p>
let's implement the <span class="highlight">/blog/new/</span> url,
actually we just need to register it in the url rule, and modify the get
method, we have already defined our post logic.

</p>
<div class="code">

    # project/app/admin/admin.py# to add the  operation url modify rules to look like thisadmin.add_url_rule('/blog/', view_func=view, methods=['GET', 'POST'])admin.add_url_rule('/blog/<operation>/', view_func=view, methods=['GET'])admin.add_url_rule('/blog/<int:obj_id>/', view_func=view,                    methods=['GET', 'POST'])# CRUDViewdef get(self, obj_id='', operation=''):    if operation == 'new':        ObjForm = model_form(self.model, db.session)        #  we just want an empty form        form = ObjForm()        action = self.path        return render_template(self.detail_template, form=form,                               path=self.path, action=action)        # ... rest of code omited

</div>

</p>
You will be able to create a new Post by Clicking the <span
class="highlight-blue">Create new</span> button. :)

</p>
Every thing is working now except for the delete, which is very easy to
implement

</p>
<div class="code">

    # project/app/admin/admin.pyadmin.add_url_rule('/blog/<operation>/<int:obj_id>/', view_func=view,                   methods=['GET'])# CRUDViewdef get(self, obj_id='', operation=''):    if operation == 'delete':        obj = self.model.query.get(obj_id)        db.session.delete(obj)        db.session.commit()        return redirect(self.path)    # ... rest of code omited

</div>

</p>
And that's all there is to it :) ... ok maybe there is a couple of
issues we haven't looked at that we can easly implement.

</p>
``` {.highlight}
"exclude" fields from the form."decorate" our view with for example 'auth' decorator and so on."filter" list view base on a given model query.
```

</p>
To exclude some fields we just supply an iterator as an argument
axclude.

Let's first define two helper methods list and detail to render our
views and change the code slighty to keep things DRY(don't repeat
yourself).

</p>
<div class="code">

    # this is the resulting CRUDVewclass CRUDView(MethodView):    list_template = 'admin/listview.html'    detail_template = 'admin/detailview.html'    def __init__(self, model, endpoint, list_template=None,                 detail_template=None, exclude=None):        self.model = model        self.endpoint = endpoint        # so we can generate a url relevant to this        # endpoint, for example if we utilize this CRUD object        # to enpoint comments the path generated will be        # /admin/comments/        self.path = url_for('.%s' % self.endpoint)        if list_template:            self.list_template = list_template        if detail_template:            self.detail_template = detail_template                self.ObjForm = model_form(self.model, db.session, exclude=exclude)    def render_detail(self, **kwargs):        return render_template(self.detail_template, path=self.path, **kwargs)    def render_list(self, **kwargs):        return render_template(self.list_template, path=self.path, **kwargs)    def get(self, obj_id='', operation=''):        if operation == 'new':            #  we just want an empty form            form = self.ObjForm()            action = self.path            return self.render_detail(form=form, action=action)                    if operation == 'delete':            obj = self.model.query.get(obj_id)            db.session.delete(obj)            db.session.commit()            return redirect(self.path)        if obj_id:            # this creates the form fields base on the model            # so we don't have to do them one by one            ObjForm = model_form(self.model, db.session)            obj = self.model.query.get(obj_id)            # populate the form with our blog data            form = self.ObjForm(obj=obj)            # action is the url that we will later use            # to do post, the same url with obj_id in this case            action = request.path            return self.render_detail(form=form, action=action)        obj = self.model.query.order_by(self.model.created_on.desc()).all()        return self.render_list(obj=obj)    def post(self, obj_id=''):        # either load and object to update if obj_id is given        # else initiate a new object, this will be helpfull        # when we want to create a new object instead of just        # editing existing one        if obj_id:            obj = self.model.query.get(obj_id)        else:            obj = self.model()        ObjForm = model_form(self.model, db.session)        # populate the form with the request data        form = self.ObjForm(request.form)        # this actually populates the obj (the blog post)        # from the form, that we have populated from the request post        form.populate_obj(obj)        db.session.add(obj)        db.session.commit()        return redirect(self.path)

</div>

</p>
To exclude something just supply a list of the fields you don't want

to the view creation, for example

</p>
<div class="code">

    view = CRUDView.as_view('blog', Blog, endpoint='blog',                        exclude=['created_on', 'updated_on'])

</div>

</p>
For a decorator you could just wrap the view like this.

</p>
<div class="code">

    def dec(f):    def decorated(*args, **kwargs):        print 'run decorator run'        return f(*args, **kwargs)    return decoratedview = dec(CRUDView.as_view('blog', Blog, endpoint='blog']))

</div>

</p>
But this is ugly, you won't have to do this after we create our
register\_crud helper function as you soon shall see.

</p>
But for now Lets see how we can add filter.

</p>
<div class="code">

    # add argument to __init__filters=None    # and set the atribute filtersself.filters = filters or {}# modify getdef get(self, obj_id='', operation='', filter_name=''):    #....    # list view with filter    if operation == 'filter':        func = self.filters.get(filter_name)        obj = func(self.model)        return self.render_list(obj=obj, filter_name=filter_name)        #... rest of code ommited# add filters to render listdef render_list(self, **kwargs):    return render_template(self.list_template, path=self.path,                           filters=self.filters, **kwargs)# add the rule admin.add_url_rule('/blog/<operation>/<filter_name>/', view_func=view,                    methods=['GET'])

</div>

</p>
The way that this works is that we add a dictionary of functions, to the
filter attribute, and then when we do a request we catch the filter name
and simply call it from the dictionary attribute and pass the model as
an argument, so we can always assume we have the model supplied to the
function,

from then on we simple do our query and return the object, which we will
display in the listview template.

</p>
So basicly we are just replacing the obj with the one returned by the
filter.

</p>
<div class="code">

    <!-- listview.html --><!-- template menu --><!-- Add this  right bellow the Create New button  -->{% if filters %}    {% if not filter_name %}        <a class="pure-button button blue" href="{{path}}">Show all</a>    {% else %}        <a class="pure-button" href="{{path}}">Show all</a>    {% endif %}{% endif %}{% for filter in filters %}    {% if filter == filter_name %}        <a class="pure-button button blue" href="{{path + 'filter/' + filter}}">{{filter.replace('_', ' ')}}</a>    {% else %}        <a class="pure-button" href="{{path + 'filter/' + filter}}">{{filter.replace('_', ' ')}}</a>    {% endif %}{% endfor %}

</div>

</p>
We now have a nice button display of filters, let's see how we can set
some up :)

</p>
<div class="code">

    # admin.pyblog_filters = {'created_asc': lambda model: model.query.order_by(model.created_on.asc()),'updated_desc': lambda model: model.query.order_by(model.updated_on.desc()),'last_3': lambda model: model.query.order_by(model.created_on.desc()).limit(3)}view = CRUDView.as_view('blog', Blog, endpoint='blog', filters=blog_filters)

</div>

</p>
If you are not much familiar with lambda basicly it's just a syntactic
sugar to write a one line function, for example

</p>
<div class="code">

    'created_asc': lambda model: model.query.order_by(model.created_on.asc())# this is the same as the line abovedef created_asc_filter(model):    return model.query.order_by(model.created_on.asc())'created_asc': created_asc_filter

</div>

</p>
That's it, browse to /admin/blog/ and see how nicely it displays the
filter menu :)

</p>
It is not so tidy to define all these urls for every model in our code,
let's define a helper functions that registers all these url for us, to
make the code more modular.

</p>
<div class="code">

    # admin.py# replace the view and url rules with this helper functiondef register_crud(app, url, endpoint, model, decorators=[], **kwargs):    view = CRUDView.as_view(endpoint, endpoint=endpoint,                            model=model, **kwargs)    for decorator in decorators:        view = decorator(view)    app.add_url_rule('%s/' % url, view_func=view, methods=['GET', 'POST'])    app.add_url_rule('%s/<int:obj_id>/' % url, view_func=view)    app.add_url_rule('%s/<operation>/' % url, view_func=view, methods=['GET'])    app.add_url_rule('%s/<operation>/<int:obj_id>/' % url, view_func=view,                     methods=['GET'])    app.add_url_rule('%s/<operation>/<filter_name>/' % url, view_func=view,                     methods=['GET'])

</div>

</p>
To register a url to a model simply call the function like this.

</p>
<div class="code">

    register_crud(admin, '/blog', 'blog', Blog, filters=blog_filters)

</div>

</p>
That was easy, right ?

What is more nice, is that we can do this to any model in the database.

</p>
Lets try adding comments to the database and see how we can display
them.

</p>
<div class="code">

    # project/app/admin/models.py# add this bellow our blog class modelclass Comment(db.Model):    id = db.Column(db.Integer, primary_key=True)    blog_id = db.Column(db.Integer, db.ForeignKey('blog.id'))    blog = db.relationship('Blog', backref=db.backref('comments',                                                      lazy='dynamic'))    username = db.Column(db.String(50))    comment = db.Column(db.Text())    visible = db.Column(db.Boolean(), default=False)    created_on = db.Column(db.DateTime(), default=datetime.utcnow)    def __init__(self, post='', username='', comment=''):        if post:            self.blog = post        self.username = username        self.comment = comment    def __repr__(self):        return '<Comment: blog {}, {}>'.format(self.blog_id, self.comment[:20])

</div>

</p>
From the terminal in the project dir, do

</p>
<div class="code">

    from app import dbfrom app.admin.models import *db.create_all()

</div>

</p>
We have now created the comment table.

</p>
How do you think we can display it in the admin interface?

</p>
<div class="code">

    # admin.py from .models import Commentregister_crud(admin, '/comments', 'comments', Comment)

</div>

</p>
That's it, head over to /admin/comments/ and will display a list of all
the comment.

</p>
It is empty, so hit the Create New and fill in some comment.

</p>
Create a Couple of filters.

</p>
<div class="code">

    comment_filters = {        'invisible': lambda model: model.query.filter_by(visible=False).all(),        'visible': lambda model: model.query.filter_by(visible=True).all()    }register_crud(admin, '/comments', 'comments', Comment, filters=comment_filters)

</div>

</p>
As you can see you could easely extend this to tens of models.

</p>
To display a menu with with our model we need to define a base html and
let listview and detailview inherit from it, that would also allow us to
overide the default templates without much code.

</p>
Refactor templates to look like this.

</p>
Note, I am including the css in the html for simplicity sake, but they
should live on their own file.

</p>
<div class="code">

    <!-- project/app/admin/templates/admin/admin.html --><!DOCTYPE html><html><head>  <title>Admin</title> <link rel="stylesheet" href="http://yui.yahooapis.com/pure/0.5.0/pure-min.css">  <style>        .main {         position: relative;          width: 90%;       margin: 0 auto;       height: inherit;        }        .main .menu ul {        width: 90%;     }     .main .menu ul li:last-child {          float: right;        display: inline-block;        }     .button {       color: white;       border-radius: 4px;        text-shadow: 0 1px 1px rgba(0, 0, 0, 0.2);      }     .button.red {       background: #ca3c3c;       }     .button.blue {          background: #3caeca;       }        .listview > a {         margin: 10px 0 10px 20px;        }        .listview .pure-g {          margin: 2px auto;          padding: 10px 0 10px 20px;          background: #f4f6ff;        }        .listview .pure-u {          overflow: hidden;        }           .item-form .pure-form {         width: 90%;       margin: 40px auto;      }     .item-form .pure-form legend {          color: #6b6b6b;          font-size: 1.2em;      }     .item-form .pure-form label {       margin-top: 30px;        color: #a5a5a5;        }     .item-form .pure-form textarea {        min-height: 200px;       padding: 10px;     }     .item-form .pure-form input, .item-form .pure-form select, .item-form .pure-form textarea {       width: 100%;        }     .item-form .pure-form input[type="checkbox"] {       width: 50px;       }     .item-form .pure-form button {          margin: 30px 0;     }   </style>        {% block head %} {% endblock %}</head><body>   <div class="main">        <div class="menu pure-menu pure-menu-open pure-menu-horizontal">            <a href="/" class="pure-menu-heading">Admin</a>            <ul>                <li><a href="{{ url_for('admin.blog')}}">Blog</a></li>                <li><a href="{{ url_for('admin.comments')}}">Comments</a></li>                <li><a class="pure-button" href="#not_implemented">Logout</a></li>            </ul>        </div>       {% block body %}      {% endblock %}   </div></body></html>

</div>

</p>
<div class="code">

    <!-- project/app/admin/templates/admin/listview.html -->{% extends 'admin/admin.html' %}{% block body %}  <div class="listview">    <a class="pure-button pure-button-primary" href="{{path + 'new'}}">Create new</a>      {% if filters %}         {% if not filter_name %}             <a class="pure-button button blue" href="{{path}}">Show all</a>            {% else %}               <a class="pure-button" href="{{path}}">Show all</a>            {% endif %}      {% endif %}           {% for filter in filters %}          {% if filter == filter_name %}               <a class="pure-button button blue" href="{{path + 'filter/' + filter}}">{{filter.replace('_', ' ')}}</a>           {% else %}               <a class="pure-button" href="{{path + 'filter/' + filter}}">{{filter.replace('_', ' ')}}</a>           {% endif %}      {% endfor %}      {% for item in obj %}            <div class="pure-g">              <div class="pure-u-2-24">             <a class="pure-button pure-button-primary button red" href="{{path +'delete/'+ item.id|string}}">delete</a></div>                <div class="pure-u"><a class="pure-button pure-button-primary button blue" href="{{path + item.id|string}}">{{item}}</a></div>            </div>     {% endfor %} </div>{% endblock %}

</div>

</p>
<div class="code">

    <!-- project/app/admin/templates/admin/detailview.html -->{% extends 'admin/admin.html' %}{% block body %}    <div class="item-form">           <form class="pure-form pure-form-stacked"              action="{{action}}"             method="post">          <fieldset>             <legend>Item View</legend>                            {% for field in form %}                   {{field.label()}}                    {{field(class="pure-input")}}                               {% endfor %}              <button type="submit" class="pure-button pure-button-primary pure-input-1-4">Submit</button>                <a class="pure-button pure-button-primary button red" href="{{ path +'delete/'+ action.split('/')[-2]}}">delete</a></div>            </fieldset>     </form>    </div>{% endblock %}

</div>

</p>
The End...

</p>
Do you know of a better way to create a CRUD like the one we just did

?

</p>
why not leave a comment with your suggestions ?

</p>
[Link for the code
repository](https://github.com/spaceexperiment/Code-for-CRUD-admin-tutorial)
