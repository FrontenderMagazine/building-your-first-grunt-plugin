*By [Mykyta Semenistyi][1]*

Client-side build systems have gained huge popularity due to the growth in
complexity of frontend development. This growth in complexity is due to two main
reasons: the migration of functional responsibilities to client-side and 
presentation enhancements. The oldest and probably most well-known of these 
build systems is Grunt. Its popularity has helped it develop a healthy ecosystem
whereby there are existing Grunt plugins for most developer tasks.

However you may still have your own task that needs to be performed that isn’
t covered by an existing plugin. That’s why you may need to learn to create your
own plugins for Grunt. In this article, I will walk you through creating your 
first Grunt plugin so that you’ll be prepared to build plugins of your own.

## Getting Started with grunt-init

The easiest way to begin building a Grunt plugin is to use internal Grunt
scaffolding tool called grunt-init. Assuming you have Node (and npm) installed, 
you can install grunt-init via the command line:

    npm install -g grunt-init

The template for the plugin is located at 
<https://github.com/gruntjs/grunt-init-gruntplugin> and it should be cloned
manually:

    git clone https://github.com/gruntjs/grunt-init-gruntplugin.git ~/.grunt-init/gruntplugin

Next, execute grunt-init in the folder where you would like to create your
plugin:

    grunt-init gruntplugin

The scaffolder will ask you plenty of questions, that help determine how it
will create the structure of plugin folder:

![grunt-plugin-1][2]

Grunt-init can generate a lot of different things related to Grunt. A full list
of templates and additional instructions may be[found here][3].

### Registering a Task

The generated files include a number of additional resource like a: readme, .
jshintrc, license, .gitignore, package.json. What we are actually interested in 
is the`plugin.js` file in tasks folder.

Plugin code can be wrapped within a `registerMultiTask` function:

    grunt.registerMultiTask(taskName, [description, ] taskFunction)

It is called a multi task because this task may contain different configuration
objects for different targets. The`taskName` string will be the same as a root
property of`grunt.initConfig` object.

The Grunt docs have nice API reference (<http://gruntjs.com/api/grunt> ) but we
’ll take a look at the most important parts for our purposes.

Within the task, **this** contains a lot of useful methods and properties, one
of them is

this.options()

method. It returns the 

options

objects defined within Gruntfile and also it may accept default values for
options:

    var options = this.options({
    enabled: false
    });

The `this.files` array contains the list of all the pairs of `src` and `dest`
files matching the pattern defined in configuration, all the globbing patterns 
will be resolved by the time of accessing the array. Most likely you are going 
to need to loop through these files, so the generated plugin contains the 
following lines:

    this.files.forEach(function(f) {
      var src = f.src.filter(function(filepath) {
        if (!grunt.file.exists(filepath)) {
          grunt.log.warn('Source file "' + filepath + '" not found.');
          return false;
        } else {
          return true;
        }
      }).map(function(filepath) {
        return grunt.file.read(filepath);
      }).join(grunt.util.normalizelf(options.separator));
    
      grunt.file.write(f.dest, src);
    
      grunt.log.writeln('File "' + f.dest + '" created.');
    });

As you can see each item of **this.files** array has an `src` property which is
actually an array of files. In the most common scenario you will filter those 
files by extension, content or some other criterion.

`grunt.file` (<http://gruntjs.com/api/grunt.file>) contains quite a few useful
methods for dealing with file system.

`grunt.util` (<http://gruntjs.com/api/grunt.util> ) contains helper functions
.

`grunt.log` (<http://gruntjs.com/api/grunt.log>) provides bunch of methods for
logging information to the console for different sets of options.

The last thing I would like to pay particular attention to is a way to handle
asynchronous actions within tasks.

    var done = this.async();
    someAsyncFunction(function(result){
    	done(result);
    });

The `this.async` method returns a `done` callback which should be called when
the task is performed. If result passed to done is false or error, the task is 
considered to have failed.

## Example Plugins

Now that we’ve covered the basics of Grunt plugin implementation, I’d like
to show you several plugins of my own and describe their peculiarities.

### Grep (<https://github.com/msemenistyi/grunt-grep>)

Grep was the first plugin I created for Grunt and probably the most complex one
. It was created in order to simplify environment-specific development. When you
work on a large application, you will definitely end up needing several builds 
based on different environments or other criteria.

Note: Their is a list of similar plugins within Addy Osmani’s article on
environment-specific builds
(
[ http://addyosmani.com/blog/environment-specific-builds-with-grunt-gulp-or-broccoli/][4]

Here’s the example of grep usage, when you configured it for production:

#### Source

    <link rel="stylesheet" href="./style.css"> <!--@grep dev-->
    <link rel="stylesheet" href="http://some.cdn/style.css"> <!--@grep production-->

#### Result

    <link rel="stylesheet" href="http://some.cdn/style.css"> <!--@grep production-->

Unlike most Grunt plugins, grep doesn’t have any dependencies, so it’s not
just a separate task being executed by Grunt, but rather an independent module.

You can find the source code [here][5].

Grep’s main functionality is implemented by utilizing searches via regular
expressions. If you want to work with file contents within your Grunt plugin, 
this snippet could be helpful to you:

    var src = grunt.util.normalizelf(src);
    var lines = src.split(grunt.util.linefeed);

This allows seamless work with text files as they solve the differences in line
endings, which may be hard to keep in mind. Grunt no longer supports
`grunt.util` hash, so here’s the source code of those properties:

    util.normalizelf = function(str) {
      return str.replace(/\r\n|\n/g, util.linefeed);
    };
    util.linefeed = process.platform === 'win32' ? '\r\n' : '\n';

Grep also takes great advantage of the Grunt API. All the operations with files
– read, write, check for presence, distinguishing files from folders – are made 
through`grunt.file` methods. If you plan to work with file content, the grep
source code may be useful to you.

### Capo (<https://github.com/msemenistyi/grunt-capo>)

Grunt-capo is a thin wrapper around the [Capo][6] module. In short, the plugin
is designed continuous integration of modules, helping to keep generated files 
up to date. Capo itself simplifies working with event-driven architectures in 
JavaScript, you can read more info about it in[this article][7].

The [source code][8] for the plugin is very concise and the most meaningful
part is written in just 3 lines. However, the major takeaway, in terms of 
learning, from this plugin is that the Capo module is asynchronous so the Grunt 
task also to be async. This is achieved by the method described in the beginning
of the article and you can read the source for clarification.

### Similar (<https://github.com/msemenistyi/grunt-similar>)

Grunt-similar is heavily dependent on Grunt and serves for enhancing the
comfort of using the task-runner via a console interface. It detects if the task
name entered was registered in Gruntfile and in case of failure it checks if 
there are tasks with similar names registered.

![grunt-plugin-2][9]

Let’s review the building blocks of the plugin. Here’s the part of all the
tasks retrieving and registering a new task with name entered:

    if (grunt.cli.tasks.length === 1){
    	var taskParts = grunt.cli.tasks[0].split(':');
    	var tasks = Object.keys(grunt.task._tasks);
    	if (tasks.indexOf(taskParts[0]) === -1){
    		grunt.registerTask(grunt.cli.tasks[0], ['similar']);
    	}
    }

`grunt.cli.tasks` is the array of tasks that were entered via the console and
`grunt.task._tasks` is the array of all tasks that are registered in the config
.

The `grunt.registerTask` method accepts the name of the task to be registered
and the array of tasks to be called. This is how the execution is being proxied 
to within the`similar` task.

Another line to note is how the plugin performs the execution of the task with
the appropriate name:

    grunt.task.run(similarTaskName);

## Conclusion

I think that in order to fully use any build system, like Grunt, it helps to
understand most of its internal mechanisms. I hope that my article clarified 
these to you. Feel free to examine the source code of any of the plugins 
discussed, as it is always better to learn seeing the implementation of certain 
parts of module.

Lastly, I should mention, as there’s been a lot of discussion about other
build tools and a performance races, of sorts, the developers of Grunt have also
set a goal to release the next, better version of the project. Work on this is 
currently in progress and can be found at[grunt-next][10].

 [1]: http://flippinawesome.org/authors/mykyta-semenistyi
 [2]: img/grunt-plugin-1.png
 [3]: http://gruntjs.com/project-scaffolding

 [4]: http://addyosmani.com/blog/environment-specific-builds-with-grunt-gulp-or-broccoli/
 [5]: https://github.com/msemenistyi/grunt-grep/blob/master/tasks/grep.js
 [6]: https://github.com/msemenistyi/capo

 [7]: http://www.binary-studio.com/2014/2/27/javascript-event-driven-architecture-capo-module
 [8]: https://github.com/msemenistyi/grunt-capo/blob/master/tasks/capo.js
 [9]: img/grunt-plugin-2.png
 [10]: https://github.com/gruntjs/grunt-next