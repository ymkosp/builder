# builder


Rake driven application builder from a database schema. 

A set of rake tasks and ruby classes to generate various types of application artifacts using a database schema as the 
source of data.  This is useful if you want to create a Ruby On Rails (with AngularJS) application and you already have a legacy database or you prefer to create the database DDL manually to take advantage of the powerful features of your database engine.

## Dependencies

- Mysql2 or JDBC-Mysql connector
- Rake
- Erubis

## Installation

git clone https://github.com/damianham/builder.git

## Usage

### Step 1  - generate the column info schema file from the database
```
 rake -f builder/tasks/mysql2.rake build_schema[testdb,testuser,testpass]    
```

Generates a testdb_column_info.yml file from the testdb database schema using a username of testuser and password of testpass 
using the mysql2 gem.  Use jdbc_mysql.rake if using JRuby.  To get the best results the tables in the database should define foreign key constraints and use table and column comments throughout.  For example for MySQL we can create a set of related tables with

```
create table `users` (
  `id` int(10) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `username` varchar(32) NOT NULL COMMENT 'The user identity used to login to the system',
  `password` varchar(64) NOT NULL COMMENT 'MD5 digest of the users password',
  `secret` varchar(255) NOT NULL COMMENT 'The users answer to the secret question for account recovery', 
  `updated_at` TIMESTAMP COMMENT 'Rails convention - date time the record was last updated',
  `created_at` DATETIME COMMENT 'Rails convention - date time the record was initially created'
) COMMENT 'Users are people who can login to the system and create interesting posts and witty comments';

create table `posts` (
  `id` int(10) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `title` varchar(255) NOT NULL COMMENT 'The title of the post that will appear in lists, tables and headings',
  `content` text NOT NULL COMMENT 'The main body content of the post',
  `user_id` int(10) NOT NULL COMMENT 'Foreign key to the user record in the users table that created the post',
  `updated_at` TIMESTAMP COMMENT 'Rails convention - date time the record was last updated',
  `created_at` DATETIME COMMENT 'Rails convention - date time the record was initially created',
  KEY `posts_user_id` (`user_id`),
  CONSTRAINT `posts_user_id_fk` FOREIGN KEY (`user_id`) REFERENCES `users`(`id`)
) COMMENT 'Posts are created by users and contain really interesting information';

create table `comments` ( 
  `id` int(10) NOT NULL AUTO_INCREMENT PRIMARY KEY,
  `content` text NOT NULL COMMENT 'The main body content of the comment',
  `post_id` int(10) NOT NULL COMMENT 'Foreign key to the post record in the posts table the comment relates to',
  `user_id` int(10) NOT NULL COMMENT 'Foreign key to the user record in the users table that created the comment',
  `updated_at` TIMESTAMP COMMENT 'Rails convention - date time the record was last updated',
  `created_at` DATETIME COMMENT 'Rails convention - date time the record was initially created',
  KEY `comments_post_id` (`post_id`),
  KEY `comments_user_id` (`user_id`),
  CONSTRAINT `comments_post_id_fk` FOREIGN KEY (`post_id`) REFERENCES `posts`(`id`)
  CONSTRAINT `comments_user_id_fk` FOREIGN KEY (`user_id`) REFERENCES `users`(`id`)
) COMMENT 'Comments completely trash the contents of posts - or maybe not';

```

### Step 2 - create any other application structures using other generators

For example you may want to create a Ruby on Rails application using the rails application generator

```
rails new mywebapp
```

### Step 3 - Configure the builder

#### Copy builder_config.rb to the current working directory and edit the copied file to define the generators and their output folders. 

The builders are defined in a hash with the name of the builder as the key and the 
builder options, including the output folder, as the value.  The default
builders are

* RailsBuilder
* AngularRailsBuilder

The RailsBuilder generates controllers,models and views and the AngularRailsBuilder
generates Angular service factories, controllers and view partials.  
If you want to use the scaffold generator that ships with Rails then change 
RailsBuilder to RailsVanillaBuilder however note that the models generated by 
the Rails scaffold generator will not have the **has_many** and **belongs_to** 
method calls to define the class relationships, nor will it have the validation 
method calls for required columns.

For example if you created a new rails application using the rails application 
generator as described above and you have 
a directory structure like


> workspace

> > builder

> > mywebapp

> > builder_config.rb 


then you would set the output folders to the following values

```
#### define the output destinations relative to where you run the rake command

@builders = {
  RailsBuilder => {:output => './mywebapp'  },
  AngularRailsBuilder => {:output => './mywebapp' }
}
```

and run the rake commands in the workspace directory with the builder_config.rb 
file in the workspace directory.

You can also add :only and :except elements to each builder's options hash 
which are both arrays of method name symbols to limit the actions of each builder.

If the :only option exists then only methods named in the 
:only array will do anything, all other methods will return immediately.  Any 
methods named in the :except array will do nothing and return immediately.

You can also add the appname option e.g. :appname => 'myapp' for the 
Angular module name.  This is the name of the Angular application and is given as the
ng-app name in the html start tag.  The default Angular module name is 'mainapp'.


```
#### define the output destinations relative to where you run the rake command

@builders = {
  RailsBuilder => {:output => './myapp',:appname => 'myapp' ,
  # do not do anything for these methods
  :except => [:finalize_angular_root, :build_model, :build_controller] },

  AngularRailsBuilder => {:output => './myapp',:appname => 'myapp',
  # don't do anything except finalize the menu
  :only => [:finalize_menu]}
}
```

#### define the fields to use for each kind of template for each table

When generating the detail.html, form.html and list.html files for each
table/module limit the fields that are generated to the given list.
Any column name listed in :except is removed from all tables.

```
@fields = {
  :except => [:id,:created_at,:updated_at],
  :detail => {
    :users => [:email,:name,:gender,:birthday]  
  },
  :form => {
    :users => [:email,:name,:gender,:birthday]
  },
  :list => {
    :users => [:email,:name,:gender,:birthday],
    :places => [:name, :lat,:long,:map_link]
    }
}
```


#### define any tables to ignore
```
IGNORE_TABLES = ['dodgy_table1','dummy_table2']
```

Do not generate artifacts for dodgy_table1 and dummy_table2. The Rails migration
version table **schema_migrations** is ignored by the column info generator tasks.

#### Edit builder/lib/angular_rails.rb

define the type of list template to use - either 'list', or 'table'
```
# define to 'list' to use the list template 
# or 'table' to use the table template
  LIST_TYPE = 'table'
```  

Uses the builder/templates/ng/partial-table.erb template to produce the 
list.html for each class.

### Step 4  - customize the templates in the templates folder to your liking

Edit builder/templates/ng/*.erb.



### Step 5  - generate the application artifacts

```
rake -f builder/tasks/builder.rake build_classes[testdb[,namespace]]
```

generates the application artifacts in the output folder using the 
generated testdb_column_info.yml file as input.  
The namespace argument will place the artifacts in a subfolder with that 
namespace.  If no appname is supplied in the builder options then the Angular 
module name will adopt the namespace argument.

The footer,header,home,menu partials from builder/templates/ng/footer.html.erb 
etc. will be generated in mywebapp/public/partials.

The main angular module app.js will be generated in 
mywebapp/app/assets/javascripts/app along with the services.js module.

An Angular module with a service factory and controllers for the detail,form 
and list views will be generated from 
builder/templates/ng/module.js.erb in mywebapp/app/assets/javascripts/modules
for each database table.

Html view files for the detail,form and list views for each database table 
will be generated from builder/templates/ng/partial-detail.erb etc. in 
mywebapp/public/modules/__table_name__/.  
Either builder/templates/ng/partial-table.erb or
builder/templates/ng/partial-list.erb will be used to generate the list.html file
depending on the value of LIST_TYPE in builder/lib/angular_rails.rb

### Step 6  - integrate angular into your rails web application

Integrate the Angular app into your web application by adding the ng-app
attribute to the html opening tag of mywebapp/app/views/layouts/application.html.erb 
using either the appname option 
from the AngularRailsBuilder config, the namespace option to build_classes 
or the default 'mainapp'.

```
<html lang="en" ng-app='mywebapp'>
<head>
<% 
        # load angular from google CDN for production and locally for development/test
        ['angular.min.js','angular-animate.min.js','angular-cookies.min.js',
          'angular-resource.min.js','angular-cookies.min.js','angular-loader.min.js',
          'angular-messages.min.js',
          'angular-sanitize.min.js','angular-route.min.js'].each do |lib| %>
          <% if Rails.env.production?  %>
            <script src="http://ajax.googleapis.com/ajax/libs/angularjs/1.3.7/<%= lib %>"></script>
          <% else %>
            <script src="/javascript/<%= lib %>"></script> 
          <% end %>
        <% end %>

        <%= stylesheet_link_tag    'application', media: 'all', 'data-turbolinks-track' => true %>
        <%= javascript_include_tag 'application', 'data-turbolinks-track' => true %>
        <%= csrf_meta_tags %>

</head>
<body>
    <!-- the angular view -->
    <div ng-view></div>
</body>
</html>
```

Add the angular main module and the modules folder to 
app/assets/javascrips/application.js
```
//= require jquery
//= require jquery_ujs
//= require turbolinks 

//= require app/app
//= require app/services
//= require_tree ./modules
```

##  Complete run through

The following steps will create a running web application from scratch based on
a mysql database existing called testdb and a database user with the logon 
username:password credentials of testuser:testpass

```
# we don't want everything in the home folder so let's use Eclipse convention
cd ~/workspace  
git clone https://github.com/damianham/builder.git
rake -f builder/tasks/mysql2.rake build_schema[testdb,testuser,testpass]
rails new mywebapp
cp builder/builder_config.rb .
edit builder_config.rb
edit builder/lib/angular_rails.rb
rake -f builder/tasks/builder.rake build_classes[testdb]
edit mywebapp/app/views/layouts/application.html.erb
edit mywebapp/app/assets/javascripts/application.js
```


##  TODO

- So many things
- Create Rspec, Cucumber and CoffeeScript generators
- Create schema extractors for other database engines

please fork and contribute

