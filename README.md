# Access Control and Sandboxing for VS Code Extensions
[Link to .pdf of paper](/Access_Control_and_Sandboxing_for_VS_Code_Extensions.pdf)
## Abstract
To decouple extensions from the main editor, Visual Studio (VS) Code has an extension system in which all extensions run within a single separate extension host process and communicate with the main process via IPC. This process isolation only protects VS Code from misbehaving extensions and provides little to no user protection. Despite demonstrated attacks targeting the extension system, proposals for extension permission control are not on the current agenda. Therefore, we perform an exploratory research in applying access control policy to VS Code extensions and demonstrate how one can integrate existing Node.js sandboxing tools into VS Code to deter malicious extensions.
## Authors
- Fan Jin
- Junhao Zeng
- Dhiren Lad
- Aymaan Ashhab Ahmed
- Tianrui Chen
