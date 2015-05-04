Heroku buildpack: Node.js with grunt support
============================================

Supported Grunt versions: 0.3 and 0.4.
See the Grunt [migration guide](https://github.com/gruntjs/grunt/wiki/Upgrading-from-0.3-to-0.4) if you are upgrading from 0.3.

This is a fork of [Heroku's official Node.js buildpack](https://github.com/heroku/heroku-buildpack-nodejs) with added [Grunt](http://gruntjs.com/) support.
Using this buildpack you do not need to commit the results of your Grunt tasks (e.g. minification and concatination of files), keeping your repository clean.

After all the default Node.js and npm build tasks have finished, the buildpack checks if a Gruntfile (`Gruntfile.js`, `Gruntfile.coffee`or `grunt.js`) exists and executes the `heroku` task by running `grunt heroku`. For details about grunt and how to define tasks, check out the [offical documentation](http://gruntjs.com/getting-started). You must add grunt to the npm dependencies in your `package.json` file.
If no Gruntfile exists, the buildpacks simply skips the grunt step and executes like the standard Node.js buildpack.


How it Works
------------

Here's an overview of what this buildpack does:

- Uses the [semver.io](https://semver.io) webservice to find the latest version of node that satisfies the [engines.node semver range](https://npmjs.org/doc/json.html#engines) in your package.json.
- Allows any recent version of node to be used, including [pre-release versions](https://semver.io/node.json).
- Uses an [S3 caching proxy](https://github.com/heroku/s3pository#readme) of nodejs.org for faster downloads of the node binary.
- Discourages use of dangerous semver ranges like `*` and `>0.10`.
- Uses the version of `npm` that comes bundled with `node`.
- Puts `node` and `npm` on the `PATH` so they can be executed with [heroku run](https://devcenter.heroku.com/articles/one-off-dynos#an-example-one-off-dyno).
- Caches the `node_modules` directory across builds for fast deploys.
- Doesn't use the cache if `node_modules` is checked into version control.
- Runs `npm rebuild` if `node_modules` is checked into version control.
- Always runs `npm install` to ensure [npm script hooks](https://npmjs.org/doc/misc/npm-scripts.html) are executed.
- Always runs `npm prune` after restoring cached modules to ensure cleanup of unused dependencies.
- Runs `grunt` if a Gruntfile (`Gruntfile.js`, `Gruntfile.coffee`or `grunt.js`) is found.
- Doesn't install grunt-cli every time.
- Installs `compass`, caching it for future use.

For more technical details, see the [heavily-commented compile script](https://github.com/BlackAndRedInc/heroku-buildpack-nodejs-grunt-compass/blob/master/bin/compile).

Usage
-----

Create a new app with this buildpack:

    heroku create myapp --buildpack BUILDPACK_URL=https://github.com/BlackAndRedInc/heroku-buildpack-nodejs-grunt-compass.git

Or add this buildpack to your current app:

    heroku config:add BUILDPACK_URL=https://github.com/BlackAndRedInc/heroku-buildpack-nodejs-grunt-compass.git

Set the `NODE_ENV` environment variable (e.g. `development` or `production`):

    heroku config:set NODE_ENV=production

Create your Node.js app and add a Gruntfile named  `Gruntfile.js` (or `Gruntfile.coffee` if you want to use CoffeeScript, or `grunt.js` if you are using Grunt 0.3) with a `heroku` task:

    grunt.registerTask('heroku:development', 'clean less mincss');

or

    grunt.registerTask('heroku:production', 'clean less mincss uglify');

Don't forget to add grunt to your dependencies in `package.json`. If your grunt tasks depend on other pre-defined tasks make sure to add these dependencies as well:

    "dependencies": {
        ...
        "grunt": "*",
        "grunt-contrib": "*",
        "less": "*"
    }

Push to heroku

    git push heroku master
    ...
    ----> Fetching custom git buildpack... done
    -----> Node.js app detected
    -----> Requested node range:  0.10.x
    -----> Resolved node version: 0.10.25
    -----> Downloading and installing node
    -----> Found Gruntfile
    -----> Augmenting package.json with grunt and grunt-cli
    -----> Restoring node_modules directory from cache
    -----> Pruning cached dependencies not specified in package.json
           npm WARN package.json mealgen@0.0.0 No repository field.
    -----> Installing dependencies
           npm WARN package.json mealgen@0.0.0 No repository field.
    -----> Caching node_modules directory for future builds
    -----> Cleaning up node-gyp and npm artifacts
    -----> Installing Compass
    -----> Restoring ruby gems directory from cache
    Updating installed gems
    Nothing to update
    -----> Caching ruby gems directory for future builds
    -----> Building runtime environment
    -----> Running grunt heroku: task

Debugging
---------

npm can be run with a verbose flag to help debugging if something fails when installing the dependencies.

* if the `VERBOSE` environment variable is set, npm is always run with verbose logging.
* if `BUILDPACK_RETRY_VERBOSE` is set, npm is relaunched in verbose mode if npm failed.

Thanks to [mackwic](https://github.com/mackwic) for these extensions.

If any ruby package (like compass) fails to install or run, check to make sure the ruby version listed in the compile script are correct to the latest Heroku versions of ruby.

To ensure that the correct version of ruby is being copied into the $build_dir/vendor folder, add the code

    OUTPUT="$(ls -a /app/vendor)"
    echo "${OUTPUT}"

above the lines 

    # copy the ruby interp into the build, why does it need this on production?
    cp -r /app/vendor/ruby-* $build_dir/vendor

in the compile script.  This will show the ruby folder when you push to Heroku.  If the ruby version has changed, you will need to update the following line:
    
    export GEM_HOME=$build_dir/.gem/ruby/<VERSION>

with the correct version.  Note that this is NOT the same as the version of ruby in /app/vendor. To get the correct version for this line, you need to put the code

    op="$(ls -a $build_dir/.gem/ruby)"
    echo "${op}"

AFTER the line that says
    
    status "Building runtime environment"

If the ruby version of the system is 1.9.2, the .gem version should be 1.9.1.  If the ruby version is 2.2.2, the .gem version should be 2.2.0.  

If the Major and Minor version are mismatched, you need to clear the build cache for the app out.  To do that, add the heroku-repo plugin

    heroku plugins:install https://github.com/heroku/heroku-repo.git -a <APP NAME>

then clear the cache out:

    heroku repo:purge_cache -a <APP NAME>

Further Information
-------------------

For more information about using Node.js and buildpacks on Heroku, see these Dev Center articles:

- [Heroku Node.js Support](https://devcenter.heroku.com/articles/nodejs-support)
- [Getting Started with Node.js on Heroku](https://devcenter.heroku.com/articles/nodejs)
- [Buildpacks](https://devcenter.heroku.com/articles/buildpacks)
- [Buildpack API](https://devcenter.heroku.com/articles/buildpack-api)
- [Grunt: a task-based command line build tool for JavaScript projects](http://gruntjs.com/)
- [Compass: SCSS with batteries](http://compass-style.org/)
