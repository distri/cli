A touched up version of jQuery's `when`. Succeeds if all promises succeed, fails
if any promise fails. Handles jQuery weirdness if only operating on one promise.

TODO: We should think about the case when there are zero promises. Probably
succeed with an empty array for results.

    global.Deferred = jQuery.Deferred

    Deferred.when = (promises) ->
      $.when.apply(null, promises)
      .then (results...) ->
        # WTF: jQuery.when behaves differently for one argument than it does for
        # two or more.
        if promises.length is 1
          results = [results]
        else
          results
      , (error) ->
        console.error error
        process.exit(1)
