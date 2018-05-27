--- 
layout: post
title: Visual Studio extensions issues
comments: true
date: 2018-05-29
---

Imagine situation you've just installed new shiny extension to your Visual Studio 2017 - you restart and ... nothing happens VS is not starting. What to do?

There is couple of options:

1. Figure out what went wrong. If you start Visual Studio with `/log` switch you should get details [log file][1]. I've tried this with mixed results in the past.

2. Uninstall problematic extension - `.vsix` file is just zip archive. When you unpack it you get `extension.vsixmanifest` file in which you find, extension id:

    ``` xml
    <Identity Id="extension-id-1234" />
    ```

    Open your `Developer Command Prompt` and execute:

    ```
    vsixinstaller /u:extention-id-1234
    ```

    That should do the trick - the next two options are rather last resort.

3. If you for whatever reason cannot obtain extension id, you can try to delete plugin folder. It should be located in `c:\program files (x86)\Microsoft\VisualStudio[Community/Pro/Enterprise]\Common7\IDE\Extensions`. The trick here is to find proper folder - all the plugin folder names are randomly generated. You have to make educated guess - files inside folder can give you some clues.

4. If all the above fail - you can use atomic option - reset all settings for your instance of VS. To do that navigate to `%APPDATA%\Microsoft\VisualStudio\` and delete folder `15.0_???????` where question marks are some random instance characters. After that you get default VS settings and you will have to install and configure all your extensions from scratch.

[1]:https://msdn.microsoft.com/en-us/library/ms241272.aspx

