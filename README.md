# [super-shell](https://www.npmjs.com/package/super-shell) [![Build status](https://ci.appveyor.com/api/projects/status/m8e3h53vvxg5wb2q?svg=true)](https://ci.appveyor.com/project/Almenon/super-shell)

A simple way to run scripts from Node.js with basic but efficient inter-process communication and better error handling.

## Features

+ Reliably spawn scripts in a child process
+ Built-in text, JSON and binary modes
+ Custom parsers and formatters
+ Simple and efficient data transfers through stdin and stdout streams
+ Extended stack traces when an error is thrown

## Installation

```bash
npm install super-shell
```

To run the tests:
```bash
npm test
```

## Documentation

### Running super code:

```typescript
import {SuperShell} from 'super-shell';
SuperShell.defaultPath = "python" //or any other language/program
SuperShell.runString('x=1+1;print(x)', null, function (err) {
  if (err) throw err;
  console.log('finished');
});
```

If the script exits with a non-zero code, an error will be thrown.

Note the use of imports! If you're not using typescript ಠ_ಠ you can [still get imports to work with this guide](https://github.com/extrabacon/super-shell/issues/148#issuecomment-419120209).

Or you can use require like so: 
```javascript
let {SuperShell} = require('super-shell')
```

### Running a script:

```typescript
import {SuperShell} from 'super-shell';
SuperShell.defaultPath = "python"
SuperShell.run('my_script.py', null, function (err) {
  if (err) throw err;
  console.log('finished');
});
```

If the script exits with a non-zero code, an error will be thrown.

### Running a Super script with arguments and options:

```typescript
import {SuperShell} from 'super-shell';

let options = {
  mode: 'text',
  Path: 'path/to/super',
  Options: ['-u'], // get print results in real-time
  scriptPath: 'path/to/my/scripts',
  args: ['value1', 'value2', 'value3']
};

SuperShell.run('my_script.py', options, function (err, results) {
  if (err) throw err;
  // results is an array consisting of messages collected during execution
  console.log('results: %j', results);
});
```

### Exchanging data between Node and Super:

```typescript
import {SuperShell} from 'super-shell';
let pyshell = new SuperShell('my_script.py');

// sends a message to the Super script via stdin
pyshell.send('hello');

pyshell.on('message', function (message) {
  // received a message sent from the Super script (a simple "print" statement)
  console.log(message);
});

// end the input stream and allow the process to exit
pyshell.end(function (err,code,signal) {
  if (err) throw err;
  console.log('The exit code was: ' + code);
  console.log('The exit signal was: ' + signal);
  console.log('finished');
  console.log('finished');
});
```

Use `.send(message)` to send a message to the Super script. Attach the `message` event to listen to messages emitted from the Super script.

Use `options.mode` to quickly setup how data is sent and received between your Node and Super applications.

  * use `text` mode for exchanging lines of text
  * use `json` mode for exchanging JSON fragments
  * use `binary` mode for anything else (data is sent and received as-is)

For more details and examples including Super source code, take a look at the tests.

### Error Handling and extended stack traces

An error will be thrown if the process exits with a non-zero exit code. Additionally, if "stderr" contains a formatted Super traceback, the error is augmented with Super exception details including a concatenated stack trace.

Sample error with traceback (from test/super/error.py):

```
Traceback (most recent call last):
  File "test/super/error.py", line 6, in <module>
    divide_by_zero()
  File "test/super/error.py", line 4, in divide_by_zero
    print 1/0
ZeroDivisionError: integer division or modulo by zero
```

would result into the following error:

```typescript
{ [Error: ZeroDivisionError: integer division or modulo by zero]
  traceback: 'Traceback (most recent call last):\n  File "test/super/error.py", line 6, in <module>\n    divide_by_zero()\n  File "test/super/error.py", line 4, in divide_by_zero\n    print 1/0\nZeroDivisionError: integer division or modulo by zero\n',
  executable: 'super',
  options: null,
  script: 'test/super/error.py',
  args: null,
  exitCode: 1 }
```

and `err.stack` would look like this:

```
Error: ZeroDivisionError: integer division or modulo by zero
    at SuperShell.parseError (super-shell/index.js:131:17)
    at ChildProcess.<anonymous> (super-shell/index.js:67:28)
    at ChildProcess.EventEmitter.emit (events.js:98:17)
    at Process.ChildProcess._handle.onexit (child_process.js:797:12)
    ----- Super Traceback -----
    File "test/super/error.py", line 6, in <module>
      divide_by_zero()
    File "test/super/error.py", line 4, in divide_by_zero
      print 1/0
```

## API Reference

#### `SuperShell(script, options)` constructor

Creates an instance of `SuperShell` and starts the Super process

* `script`: the path of the script to execute
* `options`: the execution options, consisting of:
  * `mode`: Configures how data is exchanged when data flows through stdin and stdout. The possible values are:
    * `text`: each line of data (ending with "\n") is emitted as a message (default)
    * `json`: each line of data (ending with "\n") is parsed as JSON and emitted as a message
    * `binary`: data is streamed as-is through `stdout` and `stdin`
  * `formatter`: each message to send is transformed using this method, then appended with "\n"
  * `parser`: each line of data (ending with "\n") is parsed with this function and its result is emitted as a message
  * `stderrParser`: each line of logs (ending with "\n") is parsed with this function and its result is emitted as a message
  * `encoding`: the text encoding to apply on the child process streams (default: "utf8")
  * `Path`: The path where to locate the "super" executable. Default: "super"
  * `Options`: Array of option switches to pass to "super"
  * `scriptPath`: The default path where to look for scripts. Default is the current working directory.
  * `args`: Array of arguments to pass to the script

Other options are forwarded to `child_process.spawn`.

SuperShell instances have the following properties:
* `script`: the path of the script to execute
* `command`: the full command arguments passed to the Super executable
* `stdin`: the Super stdin stream, used to send data to the child process
* `stdout`: the Super stdout stream, used for receiving data from the child process
* `stderr`: the Super stderr stream, used for communicating logs & errors
* `childProcess`: the process instance created via `child_process.spawn`
* `terminated`: boolean indicating whether the process has exited
* `exitCode`: the process exit code, available after the process has ended

Example:

```typescript
// create a new instance
let shell = new SuperShell('script.py', options);
```

#### `#defaultOptions`

Configures default options for all new instances of SuperShell.

Example:

```typescript
// setup a default "scriptPath"
SuperShell.defaultOptions = { scriptPath: '../scripts' };
```

#### `#run(script, options, callback)`

Runs the Super script and invokes `callback` with the results. The callback contains the execution error (if any) as well as an array of messages emitted from the Super script.

This method is also returning the `SuperShell` instance.

Example:

```typescript
// run a simple script
SuperShell.run('script.py', null, function (err, results) {
  // script finished
});
```

#### `#runString(code, options, callback)`

Runs the Super code and invokes `callback` with the results. The callback contains the execution error (if any) as well as an array of messages emitted from the Super script.

This method is also returning the `SuperShell` instance.

Example:

```typescript
// run a simple script
SuperShell.runString('x=1;print(x)', null, function (err, results) {
  // script finished
});
```

#### `#checkSyntax(code:string)`

Checks the syntax of the code and returns a promise.
Promise is rejected if there is a syntax error.

#### `#checkSyntaxFile(filePath:string)`

Checks the syntax of the file and returns a promise.
Promise is rejected if there is a syntax error.

#### `#getVersionSync(Path?:string)`

Returns the super version. Optional Path param to get the version 
of a specific super interpreter.

#### `.send(message)`

Sends a message to the Super script via stdin. The data is formatted according to the selected mode (text or JSON), or through a custom function when `formatter` is specified.

Example:

```typescript
// send a message in text mode
let shell = new SuperShell('script.py', { mode: 'text '});
shell.send('hello world!');

// send a message in JSON mode
let shell = new SuperShell('script.py', { mode: 'json '});
shell.send({ command: "do_stuff", args: [1, 2, 3] });
```

#### `.receive(data)`

Parses incoming data from the Super script written via stdout and emits `message` events. This method is called automatically as data is being received from stdout.

#### `.receiveStderr(data)`

Parses incoming logs from the Super script written via stderr and emits `stderr` events. This method is called automatically as data is being received from stderr.

#### `.end(callback)`

Closes the stdin stream, allowing the Super script to finish and exit. The optional callback is invoked when the process is terminated.

#### `.terminate(signal)`

Terminates the super script, the optional end callback is invoked if specified. A kill signal may be provided by `signal`, if `signal` is not specified SIGTERM is sent.

#### event: `message`

Fires when a chunk of data is parsed from the stdout stream via the `receive` method. If a `parser` method is specified, the result of this function will be the message value. This event is not emitted in binary mode.

Example:

```typescript
// receive a message in text mode
let shell = new SuperShell('script.py', { mode: 'text '});
shell.on('message', function (message) {
  // handle message (a line of text from stdout)
});

// receive a message in JSON mode
let shell = new SuperShell('script.py', { mode: 'json '});
shell.on('message', function (message) {
  // handle message (a line of text from stdout, parsed as JSON)
});
```

#### event: `stderr`

Fires when a chunk of logs is parsed from the stderr stream via the `receiveStderr` method. If a `stderrParser` method is specified, the result of this function will be the message value. This event is not emitted in binary mode.

Example:

```typescript
// receive a message in text mode
let shell = new SuperShell('script.py', { mode: 'text '});
shell.on('stderr', function (stderr) {
  // handle stderr (a line of text from stderr)
});
```

#### event: `close`

Fires when the process has been terminated, with an error or not.

#### event: `error`

Fires when the process terminates with a non-zero exit code, or if data is written to the stderr stream.