Bin Bs
======

    path = require('path')
    fs = require('fs')
    dir = path.join(path.dirname(fs.realpathSync(__filename)), '../')
    require(dir + 'dist/cli.js')
