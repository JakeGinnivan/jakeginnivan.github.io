---
layout: post
published: true
title: Running individual karma test file in VS Code
comments: true
---

I have setup to have only failures printed to my console and to not capture any log output. This is great most of the time but when writing a test sometimes you just need the console output.

So I wanted to run an individual file in VS Code, just the one I was working on.

The first step is to setup a VS Code task to define a task which will run my tests for the file. We do this by creating `.vscode/tasks.json` with contents:

``` js
{
  "suppressTaskName": true,
  "taskName": "test",
  "isTestCommand": true,
  "isWatching": false,
  "args": [
    "node",
    "./node_modules/.bin/karma",
    "start",
    "karma.vscode.conf.js",
    "--file",
    "${file}"
  ]
}
```

The interesting part of this is the `${file}` variable which will be the file the editor has focused at the time.

Next we need to create `karma.vscode.conf.js`.

``` js
const getBaseConfig = require('./karma.conf.js')
const argv = require('yargs').argv

if (!argv.file) {
  throw new Error('Expected to recieve file argument')
}

module.exports = (config) => {
  getBaseConfig(config)
  config.set({
    files: [
      './node_modules/babel-polyfill/browser.js',
      argv.file
    ],
    preprocessors: {
      [argv.file]: ['webpack', 'sourcemap']
    },
    client: {
      captureConsole: true
    },
    reporters: ['mocha'],
    mochaReporter: {
      output: 'full'
    }
  })
}
```

This config extends your base config, you may need to change the pre-processors and reporters but the idea will stay the same. We grab the filename passed to karma through `argv.file`, make sure it is setup as the only file and that it has the appropriate preprocessor setup.

Now if you open the command palette (ctrl + shift + p), then type `test` and run the `Run Test Task` command VSCode will start watching your current file and running the tests in that file. Not quite the same as running an individual test but being able to continually run the test file I am working on really helped me.
