# Using 'say hello'

A dependencies experiment to understand error output.

Following a [tutorial on YouTube](https://www.youtube.com/watch?v=h1z2qGV4KPI&ab_channel=Ashotofcode) (~18 mins)

Here's an interesting explanation of a similar error in text: [Stack Overflow](https://stackoverflow.com/questions/76039613/how-do-i-read-npm-dependency-conflict-errors)


This app consumes packages we created in other repos:

- [deps-experiment-sayhello](https://github.com/evalarumbe/deps-experiment-sayhello)
- [deps-experiment-sayhelloplugin](https://github.com/evalarumbe/deps-experiment-sayhelloplugin)

With those installed, use `npx`:

```sh
$ npx @evalarumbe/deps-experiment-sayhello
```


## Starting scenario

3 repos:
### @evalarumbe/deps-experiment-sayhello

  If no plugin is installed alongside it, this app will:

  - Print a string "no plugins"
  - Print a string "Say Hi from App"

  Otherwise if a plugin is found, it will:
  - Call the function exported by the plugin, to print a different string.

### @evalarumbe/deps-experiment-sayhelloplugin
  - Exports `SayHi()` that prints "hi from the plugin"

### @evalarumbe/deps-experiment-sayhello
  - Consumes both the above repos, using `npx` to run things and see how dependencies interact.

The details will change as we test different scenarios.

## Steps in the tutorial

- 00:55 Create Test Application + Publish to NPM
  - @evalarumbe/deps-experiment-sayhello v1.0.1

- 04:10 Install and Test Application
  - Create new project @evalarumbe/deps-experiment-usingsayhello
  - From this project dir, run:
    - `npm i @evalarumbe/deps-experiment-sayhello`
    - `npx @evalarumbe/deps-experiment-sayhello`
    - Output: 'no plugins. Say Hi from App'

- 05:00 Create Test Application Plugin + Publish to NPM
  - @evalarumbe/deps-experiment-sayhelloplugin v1.0.1

- 06:30 Scenario 1 - Happy Case App and Plugin work together successfully
  - Again in the `usingsayhello` project, run:
    - `npm i @evalarumbe/deps-experiment-sayhelloplugin`
    - `npx @evalarumbe/deps-experiment-sayhello`
    - Output: 'hi from the plugin'

- 07:15 Scenario 2 - App API is updated with breaking change - no Peer Dependency to help us
  - Author of the app changes the API (`app.SayHi()` is renamed to `app.SayHello()`) in v2.0.1
  - This means it will be looking for a `SayHello()` in the plugin, which still only provides `SayHi()`

- 08:30 Install V1 of App - No Errors or Warnings
  - As a consumer, we hear about the update and decide to install the latest version.
    - In the `usingsayhello` project, run:

      `npm i @evalarumbe/deps-experiment-sayhello@latest`

      This updates `package.json` to require `^2.0.1`, and installs the latest version

      **Note: No errors or warnings even though this was a breaking change.**

      **Peer dependencies would have given us a warning at this point (install time).**

- 08:50 Runtime error!
  - Run the app, which tries to call the plugin via the updated API:
    - `npx @evalarumbe/deps-experiment-sayhello`
    - Runtime error: The updated app tries to run a function that's not available on the old plugin! Output:
      ```sh
      no plugins
      file://.../deps-experiment-sayhelloplugin/node_modules/@evalarumbe/deps-experiment-sayhello/bin/index.js:21
      console.log(app.SayHello())
                      ^

      TypeError: app.SayHello is not a function
          at file://.../deps-experiment-sayhelloplugin/node_modules/@evalarumbe/deps-experiment-sayhello/bin/index.js:19:17

      Node.js v18.18.0
      ```

- 09:40 Scenario 3 - Using Peer Dependency on Plugin
  To prevent the runtime error in future:
  - As the author of the plugin, we want to list which applications the plugin has been tested with. We do that in `package.json` > `peerDependencies`. E.g.
  ```json
  "peerDependencies": {
    "@evalarumbe/deps-experiment-sayhello": "^1.0.1"
  }
  ```

- 10:20 Update version of Plugin to 1.0.2
  - Publish plugin v1.0.2
  - Let's see how it would've been different if we'd had the peer dependencies defined from the start:

  - In the `usingsayhello` project:
    - Reset your file system: delete `package-lock.json` and `node_modules/`
    - Edit `package.json` so `sayhello` is back to an earlier version, and `sayhelloplugin` is at the newer version (with peer dependency):
      ```json
      "dependencies": {
        "@evalarumbe/deps-experiment-sayhello": "^1.0.1",
        "@evalarumbe/deps-experiment-sayhelloplugin": "^1.0.2"
      }
      ```
    - Install: `npm i`
    - Run the app: `npx @evalarumbe/deps-experiment-sayhello`
    - Happy path: This works, because the app matches the version required by the plugin.

  - 11:30 Peer Dependency Warning on attempting to install app against plugin with `peerDependencies`

    - As before, run this one more to update `package.json` so it requires the latest version of the app. This will conflict with the earlier version required by the plugin's peer dependencies:
    `npm i @evalarumbe/deps-experiment-sayhello@latest`

    - Completes 'successfully' but with warning:

      ```sh
      npm warn ERESOLVE overriding peer dependency
      npm warn While resolving: @evalarumbe/deps-experiment-usingsayhello@1.0.0
      npm warn Found: @evalarumbe/deps-experiment-sayhello@1.0.1
      npm warn node_modules/@evalarumbe/deps-experiment-sayhello
      npm warn   peer @evalarumbe/deps-experiment-sayhello@"^1.0.1" from @evalarumbe/deps-experiment-sayhelloplugin@1.0.2
      npm warn   node_modules/@evalarumbe/deps-experiment-sayhelloplugin
      npm warn     @evalarumbe/deps-experiment-sayhelloplugin@"^1.0.1" from the root project
      npm warn   1 more (the root project)
      npm warn
      npm warn Could not resolve dependency:
      npm warn peer @evalarumbe/deps-experiment-sayhello@"^1.0.1" from @evalarumbe/deps-experiment-sayhelloplugin@1.0.2
      npm warn node_modules/@evalarumbe/deps-experiment-sayhelloplugin
      npm warn   @evalarumbe/deps-experiment-sayhelloplugin@"^1.0.1" from the root project

      changed 1 package, and audited 3 packages in 3s

      found 0 vulnerabilities
      ```

      I.e. it "Found" a requirement for a peer dependency, and it "Could not resolve" that dependency.

    - That was just a _warning_ because we were trying to install the _app_. If we try to install the _plugin_ (which has the requirement defined for `peerDependencies`) we should get a blocking _error_.

    - In `usingsayhello`:
      - Run `npm i` (installs both app and plugin)

        ```sh
        npm error code ERESOLVE
        npm error ERESOLVE could not resolve
        npm error
        npm error While resolving: @evalarumbe/deps-experiment-sayhelloplugin@1.0.2
        npm error Found: @evalarumbe/deps-experiment-sayhello@2.0.1
        npm error node_modules/@evalarumbe/deps-experiment-sayhello
        npm error   @evalarumbe/deps-experiment-sayhello@"^2.0.1" from the root project
        npm error
        npm error Could not resolve dependency:
        npm error peer @evalarumbe/deps-experiment-sayhello@"^1.0.1" from @evalarumbe/deps-experiment-sayhelloplugin@1.0.2
        npm error node_modules/@evalarumbe/deps-experiment-sayhelloplugin
        npm error   @evalarumbe/deps-experiment-sayhelloplugin@"^1.0.1" from the root project
        npm error
        npm error Conflicting peer dependency: @evalarumbe/deps-experiment-sayhello@1.0.1
        npm error node_modules/@evalarumbe/deps-experiment-sayhello
        npm error   peer @evalarumbe/deps-experiment-sayhello@"^1.0.1" from @evalarumbe/deps-experiment-sayhelloplugin@1.0.2
        npm error   node_modules/@evalarumbe/deps-experiment-sayhelloplugin
        npm error     @evalarumbe/deps-experiment-sayhelloplugin@"^1.0.1" from the root project
        npm error
        npm error Fix the upstream dependency conflict, or retry
        npm error this command with --force or --legacy-peer-deps
        npm error to accept an incorrect (and potentially broken) dependency resolution.
        npm error
        npm error
        npm error For a full report see:
        npm error /Users/.../.npm/_logs/2025-08-17T18_40_47_395Z-eresolve-report.txt
        npm error A complete log of this run can be found in: /Users/.../.npm/_logs/2025-08-17T18_40_47_395Z-debug-0.log
        ```

- 13:20 Scenario 4 - Installing Plugin with App V1 - peer dependency should warn/error


- 14:40 Install Plugin shows Error and blocks install
- 15:40 Force install of Plugin


## Notes from the tutorial

- To be able to run the file from the CLI, place `index.js` in the `bin` dir and use a shebang (first line)

- Publish to NPM by using the CLI from within your project dir: 
  `npm publish --access=public`
  You may need to log into your NPM account first. This command will let you authenticate in a browser:
  `npm adduser`

### When might we run into problems with dependencies?

- Say the author of this app now updates to v2.0.0 with a change in the API
- The [consumer](https://github.com/evalarumbe/deps-experiment-usingsayhello) updates to v2.0.0:
  ```sh
  npm i @evalarumbe/deps-experiment-sayhello@latest
  ```
  This works without errors, but all is not as it seems 👀 We'll get a runtime error (see 8:50 above)

- 