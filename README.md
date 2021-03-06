#About

This plugin makes Slick a first-class citizen of Play 2.1.


The play-slick plugins consists of 2 parts:
 - DDL schema generation Plugin that works like the Ebean DDL Plugin. Based on config it generates create schema and drop schema SQL commands and writes them to evolutions.
 - A wrapper DB object that uses the datasources defined in the Play config files. It is there so it is possible to use Slick sessions in the same fashion as you would Anorm JDBC connections.

The *intent* is to get this plugin into Play 2.2 if possible.

#Usage

##Installation 

In your application, add this configuration to the `project/Build.scala` file :

```scala
val appDependencies = Seq(
  //your dependencies
  "com.typesafe" % "play-slick_2.10" % "0.3.0" 
)

 val main = play.Project(appName, appVersion, appDependencies).settings(
  //your settings
  resolvers += Resolver.url("github repo for play-slick", url("http://loicdescotte.github.com/releases/"))(Resolver.ivyStylePatterns)
) 
```
  
### How to always use the master

Alternatively, you can use the master snapshots instead of a fixed release. In the `project/Build.scala` file, add :

```scala
.dependsOn(RootProject( uri("git://github.com/freekh/play-slick.git") ))
```

to your `play.Project`

Example :

```scala
val main = play.Project(appName, appVersion, appDependencies).settings(
  //your settings      
 ).dependsOn(RootProject( uri("git://github.com/freekh/play-slick.git") ))
```

##DDL plugin

In order to enable DDL schema generation you must specify the packages or classes you want to have in the `application.conf` file:
`slick.default="models.*"`
It follows the same format as the Ebean plugin: `slick.default="models.*"` means all Tables in the models package should be run on the default database.

It is possible to specify individual objects like: `slick.default="models.Users,models.Settings"`

##DB wrapper

The DB wrapper is just a thin wrapper that uses Slicks Database classes with databases in the Play Application . 

This is an example usage:
  
```scala
import play.api.db.slick.Config.driver.simple._

play.api.db.slick.DB.withSession{ implicit session =>
  Users.insert(User("fredrik","ekholdt"))
}
```


Or transactionally:

```scala
import play.api.db.slick.Config.driver.simple._

play.api.db.slick.DB.withTransaction{ implicit session =>
  val list = Query(Users).filter(name === "fredrik")
  val updatedUsers = update(list)
  Users.insertAll(updatedUsers)
}
```

You can load a different datasource than the default (defined by `db.default.* in application.conf`), using the DB name parameter :

```scala
play.api.db.slick.DB("myOtherDb").withSession{ implicit session =>
  Users.insert(User("fredrik","ekholdt"))
}
```

Using `import play.api.db.slick.Config.driver.simple.`_ will import the driver defing with the key `db.default.driver` in application.conf, or the one set by the test helpers in test mode (see test section for more information).

If you need to use more than one driver per mode, you can read the next section!


##Multiple datasources and drivers

When having multiple datasources and drivers it is recommended to use the cake pattern.
Do not worry about the scary name it is quite easy.

Hava look in the `samples` for an example of this or keep reading.

For each table you have, create a self-type of `play.api.db.slick.Profile` and import everything from the `profile` on your table:

```scala
trait UserComponent extends Profile { this: Profile =>
   import profile.simple._

   object Users extends Table[User]("USERS") { ... }
}
```

Then you just have to put all your tables together into a DAO (data access object) like this:

```scala
class DAO(override val profile: ExtendedProfile) extends UserComponent with FooComponent with BarComponent with Profile
```

Whenever you need to use you database you just new up your DAO and import everything in it and in its profile:
    
```scala
package models

import play.api.slick.DB
object current {
  val db = DB("mydb")
  val dao = new DAO(db.driver(play.api.Play.current))      
} 
```

or if you just want the default database : 

```scala
val dao = new DAO(DB.driver(play.api.Play.current))
```    

In your `application.conf` you can now do: `slick.default="models.current.*"`

You can then use it like this:

```scala
import current._
import current.simple._

DB.withSession{ implicit session => 
   Query(Users).list
}
```

If needed, you can also use another driver like this : 

```scala
object specificDb {
 val dao = new DAO(scala.slick.driver.MySQLDriver)     
}
```

Pweeh, there are a certain amount of lines of code there, but works great and scales along with the life cycle of your app: from the start, when you need tests, when you change the DB, ...


##Writing tests

To use a test datasource, you can use the `inMemoryDatabase` helper : 

```scala
import play.api.test.Helpers._

class DBSpec extends Specification {

  "DB" should {

   "my test" in new WithApplication(FakeApplication(additionalConfiguration = inMemoryDatabase())) {
        play.api.db.slick.DB.withSession{ implicit session =>
          //...
      }
    }

  }
}

```

This will use a H2 datasource with this url : "jdbc:h2:mem:play-test"


Off course, you can also use the default datasource :

```scala
"default ds" in new WithApplication {
    play.api.db.slick.DB.withSession{ implicit session =>
      //
}
```
   

Or a specific one : 

```scala
"specific ds" in new WithApplication {
  play.api.db.slick.DB("specific").withSession{ implicit session =>
    //
}
```


Copyright
---------

Copyright: Typesafe 2013
License: Apache License 2.0, http://www.apache.org/licenses/LICENSE-2.0.html
