---
layout:     post
title:      A better way of deploying your SailsJS application to a production-esque environment
date:       2016-02-08 18:52:30
summary:    Sails does it good but we don't need to compile our assets everytime we lift
categories: sails
---

This post is loosely written as a part 2 to my previous post, [Dynamically configuring and bootstrapping your SailsJS application]({% post_url 2016-02-05-dynamically-configuring-and-bootstrapping-your-sailsjs-application %}) - so if you followed that and have re-written your `app.js` to something similar, then consider this post the `node app.s build` section.

So for this `build` task, we will get our sails application production ready. To me, that means we need the most minimal of dependencies, our code needs to be tested, we need to ship it so when our code runs, it runs the fastest speed possible, so for example if you scale horizontally by introducing a new server to the cluster we would like that server to start as fast as possible. I'll show you how to do that.

You can start with a new JS file that builds out your `dist` payload. Dist here stands for distribution, this is what we're going to distribute out either directly to our own servers or the amazing people that manage the servers for us.

## Sample build.js

<pre>
	<code class="javascript">
var path = require('path'),
    fs = require('fs-extra'),
    chalk = require('chalk'),
    root = path.resolve(__dirname, '../../'),
    pkg = require(root+'/package.json');

module.exports = function(command) {

    var dist = root+'/dist';

    console.time('build');
    console.log(chalk.yellow('Building'), pkg.name, chalk.yellow('Version'), pkg.version, chalk.yellow('Distribution'), dist);
    console.log('');

    /*
        Some of this code is taken directly from what sails does everytime it lifts
    */

    var grunt = require('grunt'),
        includeAll = require('include-all'),
        loadTasks = function(relPath) {
            relPath = path.resolve(path.resolve(__dirname, '../../'), relPath);
            return includeAll({
                dirname: require('path').resolve(__dirname, relPath),
                filter: /(.+)\.js$/
            }) || {};
        },
        invokeConfigFn = function(tasks) {
            for (var taskName in tasks) {
                if (tasks.hasOwnProperty(taskName)) tasks[taskName](grunt);
            }
        },
        taskConfigurations = loadTasks('./tasks/config'),
        registerDefinitions = loadTasks('./tasks/register');

    invokeConfigFn(taskConfigurations);
    invokeConfigFn(registerDefinitions);

    /*
        Don't forget to add any custom tasks that you need to run before/after your build
    */

    console.log(chalk.green('grunt'), 'Running tasks', chalk.blue('buildProd'), 'and', chalk.blue('mochaTest:build'));
    grunt.tasks(['buildProd'], {}, function() {

        console.log(chalk.green('grunt'), 'OK!');
        console.log('');

        console.log(chalk.green('dist'), 'Emptying', chalk.blue(dist));
        fs.removeSync(dist);
        fs.mkdirsSync(dist);

        /*
            Copying core bits that we need,
            sometimes you may not want to include models here if you want to build them dynamically
        */
        [
            'config',
            'sql',
            'tasks/startup',
            'views',
            'api',
            '.tmp/public/images',
            '.tmp/public/fonts',
            '.tmp/public/styles/kendo',
            'package.json',
            'app.js',
            'appConfig.js',
            '.tmp/public/robots.txt',
            '.tmp/public/styles/kendo/web/images/kendoui.woff',
            '.tmp/public/min/production.min.css',
            '.tmp/public/min/production.min.js'
        ].forEach(function(dir) {
            console.log(chalk.green('dist'), 'Copying', chalk.blue(dir));
            fs.copySync(root+'/'+dir, dist+'/'+dir);
        });


        /*
            Copying specific node_modules
        */
        [
            'fs-extra',
            'amqplib',
            'commander',
            'mysql',
            'chalk',
            'rc',
            'q',
            'moment',
            'qs',
            'ejs',
            'ldapauth-fork',
            'lodash',
            'include-all',
            'sails',
            'sails-mysql',
            'socket.io-client'
        ].forEach(function(module) {
            console.log(chalk.green('dist'), 'Copying node module', chalk.blue(module));
            fs.copySync(root+'/node_modules/'+module, dist+'/node_modules/'+module);
        });

        console.timeEnd('build');

    });

};
	</code>
</pre>

## Sample server.js

<pre>
	<code class="javascript">
var path = require('path'),
    chalk = require('chalk'),
    pkg = require('../../package.json');
module.exports = function(command) {

    console.log(chalk.yellow('Launching'), pkg.name, chalk.yellow('Version'), pkg.version);
    process.argv.push('--prod');
    process.env.NODE_ENV = 'production';

    console.log(chalk.green('prod'), 'Lifting environment');

    var sails = require('sails');
    sails.lift({
        hooks:{
            grunt:false
        }
    });

    console.log(chalk.green('prod'), 'Lifted environment');

};
	</code>
</pre>

So now in our project we have a shiny new `dist/` folder that took a while to create. We upload the contents of this folder and this folder only to our servers.

What did we really achieve though? The key thing here is that we've disabled grunt by specifying `grunt:false` in our `server.js` lift method. Grunt is a task runner, hopefully we don't need to run tasks anymore in production, like recompiling CSS or linting our code. They're dev tasks, and thats the whole point of having a task runner on your dev machine is that you can change a file, and grunt would do its magic, and saving you bundles of time.

We've also only copied over the bare necessities of our `node_modules/` folder. This helps for a bunch of reasons such as no dependency on npm or the time it takes to get the dependencies we need, and lets us trim the fat by shipping modules we only need in production. For example you wouldn't need the whole of bootstrap deployed as a node_module if you've already compiled your LESS into CSS.

In my application, running `sails lift --prod` takes 26 seconds, mostly due to uglifying our client side Javascript. When we pre-compile the application and deploy the dist folder it takes 1.3seconds to lift using `node app.js`. That is 95% reduction in load times.

So now if you roll out to more machines, or you just deploy a new version to the current server(s) you have, your exposure to users seeing half-compiled assets is now virtually nil.

Feel free to give me your thoughts and comments and maybe you have a better way to do it?

Ash
