title: 'Javascript Integration Tests Coverage with Istanbul'
id: 115
categories:
  - Node.js
date: 2014-07-05 14:20:13
tags:
---

We've all heard about the importance of tests. Especially in javascript when even a minor code refactoring is becoming an issue because of the dynamic nature of the language.

But even if the project has a lot of tests, we're not sure if every line in the code is covered. That's where coverage tools come in handy.

What is the problem then? While running coverage for unit tests is pretty easy and works out of the box (e.g. [blanket.js](http://blanketjs.org "Blanket.js")), coverage for integration tests is a bit harder, because the tool doesn't know what is the code that is being tested.

<!--more-->It took me a few hours to realize that [Istanbul](https://github.com/gotwarlost/istanbul "istanbul") already solved this problem, they are giving the option to instrument code, although the documentation about this feature is pretty shallow. Maybe this post will help if you encounter the same issue.

Instrumenting your code is to convert your source code to a code with coverage data, so the coverage tool will be able to generate a report with the relevant coverage.

In order to get the report for all of our tests, assuming the unit tests and the integration tests are in the same folder, run the following:

1. Instrument your core source code + the tests source code
  ```
  istanbul instrument . -o .instrument
  ```

1. Run the server with istanbul cover
  ```
  istanbul cover --report none [main-file.js]
  ```

  Additional parameters can be added after two dashes:
  ```
  istanbul cover --report none [.instrument/main-file.js] -- [params]
  ```

1. Run the tests -
  ```
  istanbul cover --report none --dir coverage/unit node_modules/.bin/_mocha -- -R spec [./instrument/tests folder] --recursive
  ```

1. Generate the report
  ```
  istanbul report
  ```

On the first step, we instrumented the code into a special folder called `.instrument`. Later on the second one, we just ran the _instrumented_ server code.

The third step is interesting, here we executed all of the project tests, including unit tests, integration tests and system tests, assuming that all of them are in the same folder. If they are not, you can run this step for each tests folder, but make sure that the output of the coverage will be different (`--dir coverage/unit` parameter).
Notice the underscore before mocha, that's because the regular mocha forks _mocha that and detaches the coverage tool. [More about this issue](https://github.com/gotwarlost/istanbul/issues/44#issuecomment-16093330 "_mocha vs mocha").
The two dashes are as before, the values after them are the parameters to the executed command.

On the last step, we generated the report, the magic here is that istanbul knows to take all the files that matched the pattern `**/coverage*.json` ([from the source here](https://github.com/gotwarlost/istanbul/blob/master/lib/command/report.js "Istanbul source code")).

Happy Coveraging!
Noam.