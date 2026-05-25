Heroku is a platform for deploying, managing, and running applications written in Ruby, Node.js, Java, Python, Clojure, Scala, Go, PHP, and .NET.

Your app's source code and dependency file are all that's needed for Heroku to run your application.

## Deploying Applications

With Heroku, deploying apps is as easy as a git push. Once you install Heroku on your local machine, you can use standard git commands to deploy your app:

```bash
git push heroku main
```

There are other deployment options available in Heroku. For example, you can enable GitHub integration, which creates a separate application instance for every pull request, making continuous integration scenarios possible. You can also use the [Heroku API](https://devcenter.heroku.com/articles/build-and-release-using-the-api).  

## Building Application

After Heroku recive the source code of the app thje plataform automatic iniciates the build. The build mechanism is tycally language-specific(Build Processes Depending on the programming Language).

## Runing Applications on Dynos

Heroku executes applications by running a command you specified in the Procfile, on a preloaded with your build [dyno](Heroku/Dyno).