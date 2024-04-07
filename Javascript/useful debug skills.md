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
