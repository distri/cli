CLI
===

    global.$ = global.jQuery = require "jquery"
    global.CoffeeScript = require "coffee-script"

    require "./deferred_patch"

    CSON = require "./cson"

    Packager = require "./packager"
    Builder = require "./builder"
    FileData = require "./file_data"

    mkdirp = require('mkdirp')
    fs = require "fs"

Compile and run distri projects from the command line.

Read local directories to get all file names for package.

    fileData = FileData()

    error = ->
      console.error arguments
      process.exit(1)

Build into standalone html

    readSourceConfig = (pkg) ->
      if configData = pkg.source["pixie.cson"]?.content
        CSON.parse(configData)
      else if configData = pkg.source["pixie.json"]?.content
        JSON.parse(configData)
      else
        {}

    initBuilder = ->
      builder = Builder()

      # Add editor's metadata
      builder.addPostProcessor (pkg) ->
        pkg.progenitor =
          url: "http://github.com/distri/cli/"

      # Add metadata from our config
      builder.addPostProcessor (pkg) ->
        config = readSourceConfig(pkg)

        pkg.version = config.version
        pkg.entryPoint = config.entryPoint or "main"
        pkg.remoteDependencies = config.remoteDependencies

      return builder

    builder = initBuilder()

    builder.build(fileData).then (pkg) ->
      config = readSourceConfig(pkg)

      Packager.collectDependencies(config.dependencies, {})
      .then (dependencies) ->
        pkg.dependencies = dependencies

        builtFiles = Packager.standAlone pkg

        index = builtFiles[0]

        mkdirp 'dist', (err) ->
          if err
            error "Error making output dir: #{err}"
          else
            fs.writeFileSync("dist/index.html", index.content)

            runIndex()
      , error
    , error

Open in google chrome

    runIndex = ->
      # TODO: Do we need to run from a webserver?
      spawn = require('child_process').spawn
      child = spawn("google-chrome", ["dist/index.html"])
