Packager
========

**HEYO This is taken straight from https://raw.github.com/STRd6/packager/master/packager.coffee.md**

TODO: Learn how to import this as a real package.

The main responsibilities will be bundling dependencies, and creating the
package.

    Packager =

If our string is an absolute URL then we assume that the server is CORS enabled
and we can make a cross origin request to collect the JSON data.

We also handle a Github repo dependency. Something like `STRd6/issues:master`.
This uses JSONP to load the package from the gh-pages branch of the given repo.

`STRd6/issues:master` will be accessible at `http://strd6.github.io/issues/master.jsonp`.
The callback is the same as the repo info string: `window["STRd6/issues:master"](... DATA ...)`

Why all the madness? Github pages doesn't allow CORS right now, so we need to use
the JSONP hack to work around it. Because the files are static we can't allow the
server to generate a wrapper in response to our query string param so we need to
work out a unique one per file ahead of time. The `<user>/<repo>:<ref>` string is
unique for all our packages so we use it to determine the URL and name callback.

      collectDependencies: (dependencies, cachedDependencies={}) ->
        names = Object.keys(dependencies)

        Deferred.when(names.map (name) ->
          value = dependencies[name]

          if typeof value is "string"
            if value.match /^http/
              $.getJSON(value)
            else
              if (match = value.match(/([^\/]*)\/([^\:]*)\:(.*)/))
                [callback, user, repo, branch] = match

                url = "http://#{user}.github.io/#{repo}/#{branch}.json.js"

                $.ajax
                  url: url
                  dataType: "jsonp"
                  jsonpCallback: callback
                  cache: true
                .promise()
              else
                reject """
                  Failed to parse repository info string #{value}, be sure it's in the
                  form `<user>/<repo>:<ref>` for example: `STRd6/issues:master`
                  or `STRd6/editor:v0.9.1`
                """
          else
            reject "Can only handle url string dependencies right now"
        ).then (results) ->
          bundledDependencies = {}

          names.forEach (name, i) ->
            bundledDependencies[name] = results[i][0]

          return bundledDependencies

Create the standalone components of this package. An html page that loads the
main entry point for demonstration purposes and a json package that can be
used as a dependency in other packages.

The html page is named `index.html` and is in the folder of the ref, or the root
if our ref is the default branch.

Docs are generated and placed in `docs` directory as a sibling to `index.html`.

An application manifest is served up as a sibling to `index.html` as well.

The `.js`, `.json`, and `.jsonp` build products are placed into the root level,
as siblings to the folder containing `index.html`. If this branch is the default
then these build products are placed as siblings to `index.html`

The optional second argument is an array of files to be added to the final
package.

      standAlone: (pkg, files=[]) ->
        if repository = pkg.repository
          branch = repository.branch

          if branch is repository.default_branch
            base = ""
          else
            base = "#{branch}/"
        else
          base = ""

        add = (path, content) ->
          files.push
            path: path
            content: content

        add "#{base}index.html", html(pkg)
        add "#{base}manifest.appcache", cacheManifest(pkg)

        json = JSON.stringify(pkg, null, 2)

        if branch
          add "#{branch}.json.js", jsonpWrapper(repository, json)

        return files

Generates a standalone page for testing the app.

      testScripts: (pkg) ->
        {distribution} = pkg

        testProgram = Object.keys(distribution).select (path) ->
          path.match /test\//
        .map (testPath) ->
          "require('./#{testPath}')"
        .join "\n"

        """
          #{dependencyScripts(pkg.remoteDependencies)}
          <script>
            #{packageWrapper(pkg, testProgram)}
          <\/script>
        """

    module.exports = Packager

Helpers
-------

Create a rejected deferred with the given message.

    reject = (message) ->
      Deferred().reject(message)

A standalone html page for a package.

    html = (pkg) ->
      """
        <!DOCTYPE html>
        <html manifest="manifest.appcache?#{+new Date}">
        <head>
        <meta http-equiv="Content-Type" content="text/html; charset=UTF-8" />
        #{dependencyScripts(pkg.remoteDependencies)}
        </head>
        <body>
        <script>
        #{packageWrapper(pkg, "require('./#{pkg.entryPoint}')")}
        <\/script>
        </body>
        </html>
      """

An HTML5 cache manifest for a package.

    cacheManifest = (pkg) ->
      """
        CACHE MANIFEST
        # #{+ new Date}

        CACHE:
        index.html
        #{(pkg.remoteDependencies or []).join("\n")}

        NETWORK:
        https://*
        http://*
        *
      """

`makeScript` returns a string representation of a script tag that has a src
attribute.

    makeScript = (src) ->
      script = document.createElement("script")
      script.src = src

      return script.outerHTML

`dependencyScripts` returns a string containing the script tags that are
the remote script dependencies of this build.

    dependencyScripts = (remoteDependencies=[]) ->
      remoteDependencies.map(makeScript).join("\n")

Wraps the given data in a JSONP function wrapper. This allows us to host our
packages on Github pages and get around any same origin issues by using JSONP.

    jsonpWrapper = (repository, data) ->
      """
        window["#{repository.full_name}:#{repository.branch}"](#{data});
      """

Wrap code in a closure that provides the package and a require function. This
can be used for generating standalone HTML pages, scripts, and tests.

    packageWrapper = (pkg, code) ->
      """
        ;(function(PACKAGE) {
        var oldRequire = window.Require;
        #{requireSource()}
        var require = Require.generateFor(PACKAGE);
        window.Require = oldRequire;
        #{code}
        })(#{JSON.stringify(pkg, null, 2)});
      """

    requireSource = ->
      """
      (function() {
        var cacheFor, circularGuard, defaultEntryPoint, fileSeparator, generateRequireFn, global, isPackage, loadModule, loadPackage, loadPath, normalizePath, rootModule, startsWith,
          __slice = [].slice;

        fileSeparator = '/';

        global = window;

        defaultEntryPoint = "main";

        circularGuard = {};

        rootModule = {
          path: ""
        };

        loadPath = function(parentModule, pkg, path) {
          var cache, localPath, module, normalizedPath;
          if (startsWith(path, '/')) {
            localPath = [];
          } else {
            localPath = parentModule.path.split(fileSeparator);
          }
          normalizedPath = normalizePath(path, localPath);
          cache = cacheFor(pkg);
          if (module = cache[normalizedPath]) {
            if (module === circularGuard) {
              throw "Circular dependency detected when requiring " + normalizedPath;
            }
          } else {
            cache[normalizedPath] = circularGuard;
            try {
              cache[normalizedPath] = module = loadModule(pkg, normalizedPath);
            } finally {
              if (cache[normalizedPath] === circularGuard) {
                delete cache[normalizedPath];
              }
            }
          }
          return module.exports;
        };

        normalizePath = function(path, base) {
          var piece, result;
          if (base == null) {
            base = [];
          }
          base = base.concat(path.split(fileSeparator));
          result = [];
          while (base.length) {
            switch (piece = base.shift()) {
              case "..":
                result.pop();
                break;
              case "":
              case ".":
                break;
              default:
                result.push(piece);
            }
          }
          return result.join(fileSeparator);
        };

        loadPackage = function(parentModule, pkg, path) {
          path || (path = pkg.entryPoint || defaultEntryPoint);
          return loadPath(parentModule, pkg, path);
        };

        loadModule = function(pkg, path) {
          var args, context, dirname, file, module, program, values;
          if (!(file = pkg.distribution[path])) {
            throw "Could not find file at " + path + " in " + pkg.name;
          }
          program = file.content;
          dirname = path.split(fileSeparator).slice(0, -1).join(fileSeparator);
          module = {
            path: dirname,
            exports: {}
          };
          context = {
            require: generateRequireFn(pkg, module),
            global: global,
            module: module,
            exports: module.exports,
            PACKAGE: pkg,
            __filename: path,
            __dirname: dirname
          };
          args = Object.keys(context);
          values = args.map(function(name) {
            return context[name];
          });
          Function.apply(null, __slice.call(args).concat([program])).apply(module, values);
          return module;
        };

        isPackage = function(path) {
          if (!(startsWith(path, fileSeparator) || startsWith(path, "." + fileSeparator) || startsWith(path, ".." + fileSeparator))) {
            return path.split(fileSeparator)[0];
          } else {
            return false;
          }
        };

        generateRequireFn = function(pkg, module) {
          if (module == null) {
            module = rootModule;
          }
          return function(path) {
            var otherPackage, otherPackageName, packagePath;
            if (otherPackageName = isPackage(path)) {
              packagePath = path.replace(otherPackageName, "");
              if (!(otherPackage = pkg.dependencies[otherPackageName])) {
                throw "Package: " + otherPackageName + " not found.";
              }
              if (otherPackage.name == null) {
                otherPackage.name = otherPackageName;
              }
              return loadPackage(rootModule, otherPackage, packagePath);
            } else {
              return loadPath(module, pkg, path);
            }
          };
        };

        if (typeof exports !== "undefined" && exports !== null) {
          exports.generateFor = generateRequireFn;
        } else {
          global.Require = {
            generateFor: generateRequireFn
          };
        }

        startsWith = function(string, prefix) {
          return string.lastIndexOf(prefix, 0) === 0;
        };

        cacheFor = function(pkg) {
          if (pkg.cache) {
            return pkg.cache;
          }
          Object.defineProperty(pkg, "cache", {
            value: {}
          });
          return pkg.cache;
        };

      }).call(this);
      """
