# Access Control and Sandboxing for VS Code Extensions

[Link to .pdf](/Access_Control_and_Sandboxing_for_VS_Code_Extensions.pdf)

## Authors
- Fan Jin
- Junhao Zeng
- Dhiren Lad
- Aymaan Ashhab Ahmed
- Tianrui Chen

## Abstract
To decouple extensions from the main editor, Visual Studio (VS) Code has an extension system in which all extensions run within a single separate extension host process and communicate with the main process via IPC. This process isolation only protects VS Code from misbehaving extensions and provides little to no user protection. Despite demonstrated attacks targeting the extension system, proposals for extension permission control are not on the current agenda. Therefore, we perform an exploratory research in applying access control policy to VS Code extensions and demonstrate how one can integrate existing Node.js sandboxing tools into VS Code to deter malicious extensions.

## 1 Background
### 1.1 VS Code Extensions
VS Code is an open source text editor written in Typescript and developed based on the Electron framework. VS Code exposes APIs for developers to build customized extensions in order to add new features. Microsoft operates a marketplace of VS Code extensions, but extensions can also be manually installed using the pre-built VSIX file. 

To manage extensions, VS Code creates a separate extension host process during startup which loads all built-in/third-party extensions while restricting their direct access to the DOM [14]. Communications between the main VS Code process and the extension host process are conducted via an internal IPC protocol. 

The extension host process provides VS Code APIs to extensions which can be used to request information from the main process or submit changes to the DOM. Under this model, misbehaving extensions can be prevented from impacting code editor’s startup performance and stability.

### 1.2 How are Extensions Launched
In order to conserve system resources, VS Code lazily loads extensions as they are needed based on activation events specified by the extension [14]. Upon the activation event, VS Code loads the extension as a package from the path specified in the extension’s package.json. If the extension is successfully loaded, the extension host trigger’s the activate function exported by the extension. Within this function, the extension then subscribes to all further events that it wishes to be notified about by registering callbacks with the extension host. In VS Code’s source code (release/1.35), the function that does the actual loading can be found in src/vs/workbench/api/node/extHostExtensionService.ts.

### 1.3 Vulnerability
#### 1.3.1 Python Extension
On Oct. 2, 2019, a code execution vulnerability in the VS Code Python extension was found by Filippo Cremonese [13]. At the time of writing, it is the most popular extension on the marketplace with over 20M downloads [3]. 

The extension would inspect the project folder and automatically select the Python environment (e.g. a virtualenv) without user consent. With this, for instance, one could run the calculator app on Apple’s OSX simply by adding the line os.exec("/Applications/Calculator.app") to the pylint sources in the environment. 

Moving on, attackers may simply forge a Python virtualenv that contains a compromised pylint and place it into the workspace. VS Code persists the path to this virtualenv in the .vscode/settings.json file and naively trusts it without any user interaction. Therefore, when VS Code opens a Python file, it automatically sources and uses that environment, and the malicious pylint will execute. This issue is still open and has not been resolved [12].

### 1.4 Motivation
Regarding the vulnerability in the Python extension, we believe the root cause comes down to the design that all extensions are running as a user-domain process with no proper access control mechanism. As of today, such proposals are available as GitHub issues on VS Code’s repository but are not on the agenda. This specific proposal [19] details a plan for sandboxing, permission modification, safety reporting, and other mechanisms. The vulnerability, along with this proposal, formed the primary motivating factors behind our work on this project. 

The remainder of this paper is structured as follows: Section 2 discusses our efforts to profile popular extensions and Section 3 details our proposed access control policies based on those results. Section 4 steps through our work on sandboxing extensions at both the OS level and the Node.js level. We describe our success at sandboxing the Prettier extension at the Node.js level in Section 5 and list pending items for future work in Section 6.

## 2 Extension Behavior Monitoring
We utilize the strace command [8] to monitor the behavior of some of the most popular VS Code extensions. Although it is possible to manually examine the extension’s source code to find static behavior, we prefer runtime tracing because it exposes dynamic behaviors.

The strace command traces all system calls invoked by the target process, i.e. the extension host process in our case. With respect to our profiling, we focus on file system calls, and in particular open and openat. Our goal is to trace all paths that the extension host process attempts to access on behalf of the activated extensions. The paths include not only physical file paths, such as the home folder or library directories, but also virtual files like /proc.

The extensions we choose are those that provide official language support for C/C++, Java, JavaScript, TypeScript, and Python, as well as a third-party code formatter Prettier. Table 1 lists the result.

Most of the file system behavior is as expected. However, the official C/C++ extension enumerates all process information under /proc, including process status, which is not something we expected a language support extension to access. Based on our examination of the C/C++ extension’s source code, the C/C++ extension is shipped with a debugger which needs to attach itself to the target process [2].

## 3 Access Control Policy
Based on our discoveries while monitoring extensions, we propose the following file system access policy. The policy defines the paths that an extension is allowed (an allowlist) or disallowed (a blocklist) to access. It is defined in a separate manifest file intended to be shipped with the extension.

|   Ext.   | Usr config |   Sys config  |    Lib   |       Other       |
|----------|------------|---------------|----------|-------------------|
|   C/C++  |     No     | Linker cache  |  C libs  |     /proc/pid     |
|   Java   |     No     | Network hosts |    JDK   |                   |
|    JS    |     No     |      /etc     |    NPM   | /proc/filesystems |
|    TS    |     Yes    |      /etc     |    NPM   |  CPU info & /tmp  |
|  Python  |     No     |       No      | Py 2 & 3 |                   |
| Prettier | prettierrc | Network hosts |    NPM   |      CPU info     |
