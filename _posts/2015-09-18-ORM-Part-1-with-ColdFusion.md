---
layout: post
category : orm
tagline: "ORM Part 1"
tags : [ORM, ColdFusion]
title: ORM from Scratch, Part 1, with ColdFusion
---

SQL is something that I don't really enjoy.  When I first started programming, I went through the normal process of learning a server sided language (PHP) and then learning how to store data in a database (MySQL).  This of course, done all on a shared hosting environment for only a couple of dollars a month and relatively straight forward.  I didn't know how to create databases, I used cPanel for that.  I didn't understand SQL, I had phpMyAdmin (which let's be honest, is a great tool).  I understood tables, rows and columns, but they were a hassle.  I didn't like my data being forced into a structure because of the software I was using.  So I ended up writing file store system, using large JSON files, directory trees and all sorts of other weird combination of things to store and save data.

<!--more-->

-------

> This is the first post in a series to come of building an ORM (Object Relational Mapper) from Scratch.

-------



Over the years, I became more proficient at SQL and it was less of a hassle, but still an annoyance.  Then a few years ago I found [RedBeanPHP](http://www.redbeanphp.com/).  And in all of a sudden, I didn't have to worry about SQL, queries, or any of that.  RedBeanPHP can automatically create tables for you, and lets your program and models define the database.  This was exactly what I was looking for.  Years later, I still use RedBeanPHP for Rapid Prototyping applications because I don't know what my structure will look like at the end, and I don't need to.

###The Beginning of ORM

RedBeanPHP helped me look at the model first, and then look at the data second.  It was a tool that took off the stress off of designing the database first and let me focus on my application.  It completely relieved me of the dread of beginning a new project because I could just sit down and start writing.  As my application changed, and new things were added, RedBeanPHP adjusted my tables for me.  It's amazing and I can't thank [Gabor de Mooij](http://www.gabordemooij.com/) enough for creating it.

Unfortunately for me, my current job was programming in ColdFusion 9 on SQL Server.  We were using an IBO pattern (Iterator Business Object) with DAO (Data Access Objects) to pull the data from SQL Server.  It was a home grown system that had supported the business well for many years and probably will for many years to come.  While I was there, the development team decided they would like to move towards a more ORM like system so I decided to build one for us.  I was familiar with RedBeanPHP and enjoyed using, so I decided to pattern this ORM after that design.  We already had an existing database, and build system for our SQL and Tables so I didn't need to build the dynamic table creation along with any of the other complicated features of RedBeanPHP.  I just needed to be able to run CRUD operations on database tables and some other business requirements for standard columns and such.

###RedBeanCF

So after a few weeks, I had the initial base of what I named RedBeanCF in honor of RedBeanPHP.  The code has been open sourced, so if you write in ColdFusion and are backed by SQL Server, you can give it a shot here: [RedBeanCF](https://github.com/Prefinem/RedBeanCF).  After it was initially built out, we work on integrating it into our system on a few small, non-critical apps.  It was slowly adopted by the team and we soon started adding features.

It soon became the choice for the developers on the team and I started extending it until it was used by enough applications that we pushed it to production and started replacing our IBOs and DAOs with it.

###Dangit! I mean CRUD

When developing an ORM, everything comes down to the four main operations.  Create, Read, Update, Delete or CRUD for short.  Relationships, Lazy Loading, Searching.  All of these at some level are just a basic operation.  So our first objective is figuring out how we want to load and persist and object.

####Define your object

In ColdFusion (and other languages) we can create objects on the fly.  This is the basis of OOP (Object Oriented Programming).  For an ORM, we need an base object (bean) that will allow us to perform CRUD operations on and that can be extended in the future for other purposes (Models).  It is possible for creating an object that persists itself, but in doing so, you will have tightly bound the object to it's functionality.  With the IBOs, we were in the same situation although the IBO was passing the CRUD operations to the DAO's.  With our new ORM, we really only want one Class Object to do all the CRUD operations.  This is simpler, easier to extend and keeps memory usage down (one class instead of a class for every object):

**IBO and DAO Pattern**

* BaseIBO
* BaseDAO
* UserIBO
* UserDAO
* CompanyIBO
* CompanyDAO
* AddressIBO
* AddressDAO

**ORM Pattern**

* ORM Class
* ORM Bean
    * User Model
    * Company Model
    * Address Model

While we only save three loaded classes, if you extrapolate this to a real business application where you have hundreds of tables/models, you are saving yourself the equivalent number of loaded classes.  On top of that, for basic CRUD operations, we won't need model classes loaded so some objects will not require a Model class.


####The ORM Class

The ORM Class (in this case RedBeanCF) is the Class we will use for all the CRUD operations and in the future for lazy-loading and other business rules (which will be done through extending the class).  When starting, the first I did was work on getting data out of the database.  This means that we have to get it setup for loading the connection, and querying out the data.  Once we have the data, we will need to put it into an object so that we can use the data.  This will include loading the bean and shoving the record into the bean.  Because I am in a fan of small functions, it will be done in several parts.

#####Initialize the ORM Class

So lets first work on the initializing the class.  This will mean that when we instantiate the class, we will need the connection info. Coldfusion has an init class that can be used for some initialization, but since we only want on class, we don't want to pass it along with the init call.

    //rb.cfc
    component {

        public function init(){
            return this;
        }

        public function setup(required string dataSource){
            variables.dataSource = arguments.dataSource;
        }
    }

Usage goes something like this

    var RB = new rb();
    RB.setup("datsource_name");

That is pretty simple and straight forward, but we haven't done anything yet.  Next lets create a find method method.  We don't want to have load do everything, so we will be using two lower functions.  These two functions will be a dispense and a populate method.  One to create a bean, and the other to load the data.  We also want to simplify our query, so lets create a where method so we can have one place to write our query

    public function findOne(required string componentName, required string where, required array values){
        var bean = dispense(arguments.componentName);
        var records = whereQuery(bean, where, values);
        if(records.recordcount > 0){
            return populatebean(bean,records);
        }else{
            return;
        }
    }

This lets us pass our componentName ie, our table along with the where clause and any parameters like this:

    var user = RB.findOne('user','id = ?',[1]);

Now that we are trying to call the method, we need to dispense the bean, get the records, and populate the bean

    public function dispense(required string componentName){
        var bean = new bean();

        bean._rb = this;

        bean._info = {};
        bean._info.tableName = componentName;

        return bean;
    }

    private function whereQuery(required bean, string where="", array values = []){
        if(!len(trim(where))){
            where = " 1=1 ";
        }
        var queryService = new query();
        queryService.setDatasource(variables.dataSource);
        queryService.setName(arguments.bean._info.tableName);
        for(var value in values){
            queryService.addParam(value=value);
        }
        var results = queryService.execute(sql="SELECT * FROM [#arguments.bean._info.tableName#] WHERE #arguments.where#");
        return results.getResult();
    }

    private function populatebean(required bean, required records, required iterator = 1){
        bean._info.cache = {};
        for(var i = 1; i <= ListLen(records.columnList); i++){
            var column = ListGetAt(records.columnList,i);
            var value = records[column][iterator];
                bean[column] = value;
            bean._info.cache[column] = value;
        }
        return bean;
    }

For the bean, we are going to store the record in the bean.variables and store and instance of RedBeanCF in _rb so that we can pull any information we need too from the bean.  The _info variable will help us keep track of what type of object this is.  The whereQuery is our query that will take the where clause and add the parameters in a safe way.  populateBean will take the object and shove the record into the object.  We also will create a cache of the data so that we can tell if a value has changed for persistence later on.

Again, we are using this by making this call:

    var user = RB.findOne('user','id = ?',[1]);

We probably need to setup the bean class now that we are trying to shove data into it.  The bean at this point is a simple object with nothing special.

    //bean.cfc
    component {

        function init(){
            return this;
        }

    }

Once we have that done, we now have a user object with data in the bean._data structure.  Our data is stored in the bean.variable structure where we loaded them from the populatebean method.  This means we can access the data and set the data with simple '.' calls

    var firstName = user.firstName;
    user.firstName = 'William';

Now we have a viable object that we can use.  But what if want to be able to get back an array of beans because we want to load a set of objects.  Lets build a find method that returns an array of these beans

    //rb.cfc
    public function find(required string componentName, string where="", array values=[]){
        var bean = dispense(arguments.componentName);
        var records = whereQuery(bean, arguments.where, arguments.values);
        var allbeans = [];
        for(var i = 1; i <= records.recordcount; i++){
            var bean = dispense(arguments.componentName);
            bean = populatebean(bean, records, i);
            arrayAppend(allbeans,bean);
        }
        return allbeans;
    }

Instead of just loading the first record into one bean, we will loop over all the records and create an object for each one, and shove a record into each one.  This lets us have a standard array of beans which means we can use all our native for loops.

    var users = RB.find('user','firstName = ?',['William']);
    for(var i = 1; i <= ArrayLen(users); i++){
        WriteOutput(users[i].firstName);
    }

Congrats!  You now can now retrieve records from the database all in a nice object which you can extend and apply other functions too instead of writing queries and getting back a result set;

###Part 2 Coming Soon

