#A Small Ratpack Primer

Ratpack is a micro framework for building web applications. It is inspired by Sinatra, which aims to create web applications in Ruby with minimal effort. 

Ratpack differs from Sinatra in that it is purely implemented in Java. It sits on top of Netty. You can write Ratpack applications in pure java. But you don't have to. Ratpack has a strong focus on Groovy and Gradle. 

Let's take a tour of Ratpack.

##Getting started

To start play with Ratpack, you just need to have Java and Groovy installed. 

If you don't have the Java JDK and JAVA_HOME set, follow this guide ( http://docs.oracle.com/javase/7/docs/webnotes/install/ )

The easiest way to install groovy is via the GVM tool. Open a terminal and type:

```bash
curl -s get.gvmtool.net | bash
gvm install groovy
```

##Your first Ratpack application

Open a text file and save it as ratpack.groovy

```groovy
@GrabResolver("https://oss.jfrog.org/artifactory/repo") // (1)
@Grab("org.ratpack-framework:ratpack-groovy:0.9.0-SNAPSHOT") // 

import static org.ratpackframework.groovy.RatpackScript.ratpack 

ratpack { // (2)
    handlers { 
        get { 
            response.send "Hello World" // (3)
        }
    }
}
```

Open a terminal window and type:

```
groovy ratpack.groovy
```

The first time you run the script, it might take a little while as Groovy downloads the libraries that are needed to run Ratpack.

You should see the following output:

```
Aug 15, 2013 11:10:06 PM org.ratpackframework.server.internal.NettyRatpackService startUp
INFO: Ratpack started for http://localhost:5050
```

If you open a web browser and navigate to http://localhost:5050, you should see your ratpack application available. The words 'Hello World' should appear.

Let's break down the script a little bit. Leave the server running now.

1. this tells the groovy script to download the libraries needed for ratpack from the jfrog repository.
2. The main application, it is composed of several different handlers and modules
3. This is a get handler. Since it has no parameters, it will resolve to the root of our application. We are telling it to return the words 'Hello World'

###Adding another handler.

Let's try adding another handler. Open the ratpack.groovy file and add a second handler:

```groovy
ratpack { 
    handlers { 
        get { 
            response.send "Hello World" 
        }
        get('echo'){             // our new handler
            response.send "echo" 
        }
    }
}
```

Save the file and go to http://localhost:5050/echo. You should see the words echo appear. Notice we did not have to restart the server. Ratpack is capable of reloading any changes that you make. This is thanks to the Spring Reloading Agent. 

##Dynamic Urls

###Path Tokens

But this is the web. It should be exciting and fun. We should at least be able to specify dynamic parameters.

Let's change our echo handler so it returns what we entered:

```groovy
get('echo/:message'){
    response.send pathTokens.message
}
```

Save the file and navigate to http://localhost:5050/echo/repeat. 

You should see the words 'repeat' returned on the screen. In our handler, we have use the syntax :value. This tells Ratpack that whatever we enter in this path should be saved into the pathTokens variable.

Interestingly, if you type http://localhost:5050/echo/is this real! . You should see the URL encoded version of this string returned to you. is%20this%20real!

###Optional Path Tokens

What if we want parts of our path to be optional? 

Let's change our echo server to optionally remove parts of my message. 

This can be done by adding a question mark at the end of the optional parameter. :this?

```groovy
get('echo/:message/:trim?'){
    response.send pathTokens.message - pathTokens.trim
}
```

* Save and enter http://localhost:5050/echo/unomas/mas, you should see uno.
* Enter http://localhost:5050/echo/unodos, you should see unodos.

You can have as many optional path tokens as you want as long as they are not followed by a mandatory token. 

```groovy
get('echo/:message/:trim?/:add?'){
	response.send pathTokens.message - pathTokens.trim + ( pathTokens.add?:'' )
}
```

* This should return 'unotres' for http://localhost:5050/echo/unodos/dos/tres. 
* 'unodos' for http://localhost:5050/echo/unodos/tres

###Query Parameters

You can also have query parameters via the request.queryParams field in your handlers.

In the following example, we capitalize a message if the upper=true query parameter is passed into our handler.

```groovy
get('echo/:message'){
	String message = pathTokens.message
	if( request.queryParams.upper ){
		message = message.toUpperCase()
	}
	response.send message
}
```

* If you enter http://localhost:5050/echo/uno, you'll get uno
* If you enter http://localhost:5050/echo/uno?upper=false, you'll get uno.
* If you enter http://localhost:5050/echo/uno?upper=true, you'll get UNO.

The Ratpack DSL is very robust. You are able to specify different methods, prefixes, headers and content-negotiation behaviour. This should be outlined in the user manual soon. 

##Static Files

Ratpack makes it very easy to serve static files from a directory. Let's try adding an image and getting ratpack to serve this image.

* Create a folder called images at the same level as your ratpack.groovy script

* Add a file myimage.png

Now, we can create a simple handler for this directory.

In your ratpack.groovy file, add the following lines at the end:

```groovy
prefix('images'){
		assets "images"
}
```

If you navigate to http://localhost:5050/images/myimage.png, you should see the image from that directory.

The assets hierarchy also mirrors your folder structure; Add a file under images/css/uno.css and you should be able to see this under http://localhost:5050/images/css/uno.css

It does not have to be a direct match to a directory, you can change the path

```groovy
prefix('images/public/today'){
		assets "images"
}
```

Now the image can be found at http://localhost:5050/images/public/today/myimage.png.

##Templates

So far, we have only been returning text in our application. But ratpack comes with the ability use external templates. 

TBD - GroovyTemplates and Handlebars

##Blocking Operations

One of the interesting features about ratpack is the fact that calls are non-blocking by default. However, if you have things that may take a little bit longer but are necessary for your application, you can force them into blaocking calls.

Here is an example from the FOASS application that converts a string of text into an mp3 before serving it back to the user:

```groovy
prefix("mp3") {
  get(":type/:p1/:p2?") {
    def (to, from, type) = betterPathTokens
    FuckOff f = service.get(type, from, to)
    exec blocking, 
        { f.toMp3() },  // what to run
        { byte[] bytes -> response.send "audio/mpeg", copiedBuffer(bytes) } // on success
  }
}
```

The exec blocking block takes in a few closures, the first one is the blocking operation, followed by what needs to happen on success. 

You can also provide an optional failure condition to the exec blocking operation, as outlined by the following test:

```groovy
get {
       exec blocking,
          { sleep 300; throw new Exception("!") }, // blocking
          { response.status(210).send("error: $it.message") }, // on error
          { /* never called */ } // on success
       }
}
```

##Building a Java-based application with Gradle.

While our small script is nice, it is fairly difficult to deploy and grow. We really need a good project structure to hold our application together.

Luckily, there is a tool called LazyBones that is capable of generating a good Groovy / Gradle application template for us.

First, install Lazybones

```bash
gvm install lazybones
```

Next, create your Ratpack project and provide it with a name,

```bash
lazybones create ratpack myApp
```

Bear in mind that there are two versions of the ratpack module, ratpack-lite is a stripped down bared bones template with little or no content.

You should now see an application created for you. To run the application, simply call

```bash
./gradlew run
```

To create a deployable version of your application, call

```bash
./gradlew iA
```

Let's explore the generated structure:

```
├── README.md
├── build.gradle
├── gradle
├── gradlew
├── gradlew.bat
└── src
    ├── main
    │   └── groovy
    ├── ratpack
    │   ├── config.groovy
    │   ├── public
    │   │   ├── images
    │   │   ├── lib
    │   │   ├── scripts
    │   │   └── styles
    │   ├── ratpack.groovy
    │   └── templates
    │       └── index.html
    └── test
        └── groovy
```

In this template, you can put your tests under src/test/groovy and your main code under src/main/groovy.

There is a config.groovy file that adds additional parameters to your application, as well as a ratpack.groovy file that contains an example of templating and has static assets set up to serve from the public directory.

Overall, this template is very intuitive and I highly recommend using this structure instead of trying to forage your own bit of madness.

There is a section with github repositories of other example applications if you would like to explore alternatives.

( There is currently a bug in the generated template - I have submitted a [pull request](https://github.com/pledbrook/lazybones/pull/72) )

##Modularizing your application.

As your application grows, it becomes more and more important to separate out bits of code so they can be tested and developed in isolation.

Ratpack provides a very powerful dependency injection mechanism powered by Google Guice.

TBD example.

##Deploying your application to the Cloud.

* Marco Vermeulen has worked very hard in getting Ratpack deployed into Heroku. You can read his [blog post](http://wiredforcode.com/blog/2013/08/05/deploy-ratpack-on-heroku/) on how to deploy Ratpack using a buildpack. Marco is working on a [gradle plugin](https://github.com/marcoVermeulen/gradle-heroku) that will make deployment into Heroku fairly simple.

* If you would rather build your application locally instead of having Heroku do it, I have some instructions [on how to deploy to Cloud Foundry without a Buildpack](http://fbflex.wordpress.com/2013/08/07/ratpack-to-cloudfoundry-with-java-buildpack/).

## Ratpack Examples 

* [A standalone Groovy script app](https://github.com/ratpack/example-ratpack-standalone-groovy-script)
* [A Groovy app built with Gradle](https://github.com/ratpack/example-ratpack-gradle-groovy-app)
* [A Java app built with Gradle](https://github.com/ratpack/example-ratpack-gradle-java-app)

##Reference Projects

There are a few good projects you can look at for inspiration:
* [FOASS](https://github.com/danveloper/ratpack-foaas/)
* [Ratpack Website](https://github.com/ratpack/ratpack/tree/master/ratpack-site)
* [GORM and MongoDB with Ratpack](https://github.com/tyama/ratpack-gorm-mongo-example/)
