CSON
====

Currently a total hack.

    module.exports =
      parse: (source) ->
        Function("return #{CoffeeScript.compile(source, bare: true)}")()
