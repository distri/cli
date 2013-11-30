File Data
=========

    file = require "file"
    fs = require "fs"

Read the contents of the current directory as an array of `fileData`.

    module.exports = (encoding="utf-8") ->
      hidden = /^\..+/

      fileNames = []
      file.walkSync ".", (dirName, dirs, files) ->
        return if dirName.match /^node_modules/
        return if dirName.match /^dist/
        return if dirName.match hidden

        if files.length
          files.forEach (fileName) ->
            name = "#{dirName}/#{fileName}".replace /^\.\//, ""
            fileNames.push name unless name.match hidden

      fileNames.map (name) ->
        path: name
        content: fs.readFileSync name,
          encoding: encoding
