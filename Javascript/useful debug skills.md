## Preface

As a developer,bug is something unavoidable for us. To solve those bugs effectively, we need to master some efficient debug skills, this is why this article exists.

## The principle of debug

### Chrome Devtool

Chrome devtools is composed of four parts, explained by following graph:

![](https://github.com/NealST/my-system/assets/17682407/cc6077c5-8eef-4236-a79e-f9438d4bff03)

### VSCode Debugger

In a project opened in vscode, you can start debugging by adding some configuration in .vscode/launch.json file. For example:
```json
{
  "version": "0.2.0",
  "configurations": [
  	{
      "type": "chrome",
      "request": "launch", // debug mode: launch ｜ attach
      "name": "example",
      "file": "${workspcaceFolder}/start-from-scratch.html"
    }
  ]
}
```

The basic principle is similar to chrome devtool. But there exists some differences that vscode debugger need extra adaptor to process a variety of programing language.

![](https://github.com/NealST/my-system/assets/17682407/1ba4ef4e-79d2-4224-86d4-f6dfebc20fc2)

## Breakpoint Debugging

Add breakpoint is a common way to pause the execument of a program, but normal breakpoint is not satisfied in some cases like exceptions. We need some other types of breakpoint to help us.

### Exception Breakpoint

you can use it like this:
![exception breakpoint](https://github.com/NealST/my-system/assets/17682407/8be5a35f-f115-400a-9883-a7e27c58272b)


it will pause the program at the place of throwing exception. If you want to locate the exception quickly, this type of breakpoint is very helpful.

### Condition Breakpoint

It will pause the program when meeting some conditions. You can use it like this:
![condition breakpoint](https://github.com/NealST/my-system/assets/17682407/db4591da-6ae3-4f20-ac08-798ba6d37849)


It's useful in some scenarios that we want to control the execution of the program without modifying the code and rerun the app.

### Log Breakpoint

It's similar to 'console.log' which is useful to print the value of some variables but we do not need to write a 'console.log' expression in our codebase.
![log breakpoint](https://github.com/NealST/my-system/assets/17682407/40fc786b-163f-4479-aa42-82eec5a34ab8)


### EventListener Breakpoint

If we want to pause the program after some events are fired but we hate to cost time to find the concret code that processing the event, eventListener breakpoint will be a good choice. It's able to manage the global event.
![eventlistener breakpoint](https://github.com/NealST/my-system/assets/17682407/785712a9-31cf-4df6-a640-c8ca95d53c5e)


### URL Request Breakpoint

This type of breakpoint has ability to pause the program before sending a url request. So it's useful for us to modify the request body to verify some assumptions,which will boost our productivity when coordinating with the backend.

### DOM Breakpoint

It will pause the program when a dom is removed or some attributes are changed. It's useful for us in some scenarios that we directly manipulate the dom.

## Chrome Devtools Skills

### Using function keys to change 'px' value

Aside from using the key '↑' or '↓' to add or subtract one, we can use these function keys to change value quickly.

* cmd + ↑/↓: add or descrease value per 100
* shift + ↑/↓: add or decrease value per 10
* option + ↑/↓: add or decrease value per 0.1
![function key](https://github.com/NealST/my-system/assets/17682407/4eac2e51-b197-41a8-b4a4-ec75c57b592f)


### Switch color quickly

You could select and switch color quickly by using 'shift' key:
![switch color](https://github.com/NealST/my-system/assets/17682407/e510470b-f268-43a2-a5b7-97558e4cc212)


### Check the request initiator

In network panel of chrome devtools, you could 
have an insight for the call stack of every network request through the initiator tab like this:

So we can locate the code efficiently.

### Filter requests

If we have the desire to find a concret request in our website, other than keywords, the filter is also a nice method. All we need to do is pressing the ctrl + space compose key, when adding a '-' prefix before the filter, we can toggle the filter result, it's equal to click the invert checkbox.
![request filter](https://github.com/NealST/my-system/assets/17682407/1c9a65a2-9b42-4d67-be09-8e9559ce94eb)


### Ignore some scripts in stack

When we debug the program step by step, we may step into some common libaries such as react, but we just care about the code of ourselves. At this time, we could press the right button of the mouse to add the library script to ignore list.
