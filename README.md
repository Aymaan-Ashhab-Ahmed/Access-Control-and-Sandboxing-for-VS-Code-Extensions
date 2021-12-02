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

## 1. Background
### 1.1. VS Code Extensions
VS Code is an open source text editor written in Typescript and developed based on the Electron framework. VS Code exposes APIs for developers to build customized extensions in order to add new features. Microsoft operates a marketplace of VS Code extensions, but extensions can also be manually installed using the pre-built VSIX file. 

To manage extensions, VS Code creates a separate extension host process during startup which loads all built-in/third-party extensions while restricting their direct access to the DOM [14]. Communications between the main VS Code process and the extension host process are conducted via an internal IPC protocol. 

The extension host process provides VS Code APIs to extensions which can be used to request information from the main process or submit changes to the DOM. Under this model, misbehaving extensions can be prevented from impacting code editor’s startup performance and stability.
