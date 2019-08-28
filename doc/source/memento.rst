Memento
=======

Some examples can be found into bloks test

.. note:: 

    The authentification will not be explained here, because the authentification 
    and authorization are used for **anyblok_pyramid**

    For the authorization, the name of the resource is defined by the attribute
    ``resource_name`` or ``model``

The querystring
---------------

The querystring of the url is parsed by the validator

example:
::

    /my/path?filter[fieldname][ilike]=something&order_by[asc]=fieldname&offset=0&limit=20

the element in the query string are:

* ``offset=0``: add an offset in the query
* ``limit=20``: limit the result
* ``order_by[asc]=fieldname``: order the results asc or desc by the fieldname given
* ``filter[fieldname][operator]=value``: the filters are seen with an **AND** condition between them
  
  * ``fieldname``: name of the model field or name of a relationship field: **name1.name2**
  * ``operator``: operator of the condition

    * **eq**
    * **like**
    * **ilike**
    * **lt**
    * **lte**
    * **gt**
    * **gte**
    * **in**
    * **or-like**
    * **or-ilike**
    * **or-lt**
    * **or-lte**
    * **or-gt**
    * **or-gte**

    .. note::
        
        for **in** and **or-..** the value is a string with **,** to separate the values

* ``~filter[fieldname][operator]=value``: the **~** means **not**
* ``context[key]=value``: add context for some filter

  .. note::

        the context is used with an **adapter**

* ``tag=value``: name one tag function defined in the Adapter to filter the query with

  .. note::

        the tag is used with an **adapter**

* ``tags=value1,value2,...``: list of the tag's function names to filter the query with

  .. note::

        the tags are used with an **adapter**


Create a simple CRUD resource
---------------------------

Exemple model::

    from anyblok import Declarations
    from anyblok.column import Integer, String

    @Declarations.register(Model)
    class Example():
        id = Integer(primary_key=True)
        name = String(label="Name", unique=True, nullable=False)

CRUD resource based on the example model::

    from anyblok_pyramid_rest_api.crud_resource import (
        CrudResource, resource)
    from anyblok_pyramid import current_blok

    @resource(collection_path='/examples', path='/examples/{id}',
              installed_blok=current_blok())
    class ExampleResource(CrudResource):
        model = 'Model.Example'


This example creates a CRUD based on the following paths:

* **/examples**: The collection path

  * post: The body waiting a list of element to create
  * get: Return a list of elements filtered by the querystring
  * put: Replace all elements in the list by new values
  * patch: Update all elements in list
  * delete: Delete the elements in the list

* **/examples/{id}**: element path: here id is the primary key of example

  * get: Return a formated element
  * put: Replace the element by new values
  * patch: Update the element
  * delete: Delete the element

the serialization and deserialization of the request is done by 
`AnyBlok Marshmallow <http://doc.anyblok-marshmallow.anyblok.org/>`_.
The schema is auto generated based on the Model

Create a CRUD with complex schema
-------------------------------

Address models::

    from anyblok import Declarations
    from anyblok.column import Integer, String
    from anyblok.relationship import Many2One, Many2Many
    
    
    Model = Declarations.Model
    
    
    @Declarations.register(Declarations.Model)
    class City:
    
        id = Integer(primary_key=True)
        name = String(nullable=False)
        zipcode = String(nullable=False)
    
        def __repr__(self):
            return '<City(name={self.name!r})>'.format(self=self)
    
    
    @Declarations.register(Declarations.Model)
    class Tag:
    
        id = Integer(primary_key=True)
        name = String(nullable=False)
    
        def __repr__(self):
            return '<Tag(name={self.name!r})>'.format(self=self)
    
    
    @Declarations.register(Declarations.Model)
    class Customer:
        id = Integer(primary_key=True)
        name = String(nullable=False)
        tags = Many2Many(model=Declarations.Model.Tag)
    
        def __repr__(self):
            return '<Customer(name={self.name!r}, tags={self.tags!r})>'.format(
                self=self)
    
    
    @Declarations.register(Declarations.Model)
    class Address:
    
        id = Integer(primary_key=True)
        street = String(nullable=False)
        city = Many2One(model=Declarations.Model.City, nullable=False)
        customer = Many2One(
            model=Declarations.Model.Customer, nullable=False,
            foreign_key_options={'ondelete': 'cascade'}, one2many="addresses")

Schema::

    from anyblok_marshmallow import SchemaWrapper
    from marshmallow import validates_schema, ValidationError
    from anyblok_marshmallow.fields import Nested
    
    
    class CitySchema(SchemaWrapper):
        model = 'Model.City'
    
    
    class TagSchema(SchemaWrapper):
        model = 'Model.Tag'
    
    
    class AddressSchema(SchemaWrapper):
        model = 'Model.Address'
    
        class Schema:
            # follow the relationship Many2One and One2One
            city = Nested(CitySchema)
    
    
    class CustomerSchema(SchemaWrapper):
        """Schema for 'Model.Customer'
        """
        model = 'Model.Customer'
    
        class Schema:
            # follow the relationship One2Many and Many2Many
            # - many=True is required because it is Many2Many or One2Many
            # - exclude is used to forbid the recurse loop
            addresses = Nested(AddressSchema, many=True, exclude=('customer', ))
            tags = Nested(TagSchema, many=True)
    
            @validates_schema(pass_original=True)
            def check_unknown_fields(self, data, original_data):
                unknown = set(original_data) - set(self.fields)
                if unknown:
                    raise ValidationError('Unknown field', unknown)

CRUD resource based on the address model and on the schema::

    from anyblok_pyramid_rest_api.crud_resource import (
        CrudResource, resource)
    from anyblok_pyramid import current_blok

    @resource(
        collection_path='/addresses/v3',
        path='/addresses/v3/{id}',
        installed_blok=current_blok()
    )
    class AddressResourceV3(CrudResource):
        model = 'Model.Example'
        default_schema = AddressSchema
