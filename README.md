# Node Create Guidelines

Guidelines for Creating Nodes for Node-RED

## About This Project

This project includes a sample node (`example-node`) that adheres to the recommended structure for Node-RED node development. It serves as a reference implementation for creating well-structured and localized nodes.

## Project Structure

```
node-create-guidelines/  
├── README.md                # This document  
├── package.json             # Package settings including "node-red.nodes" field  
├── docs/                    # Development guides  
│   ├── folder-structure.md  # Folder structure guide  
│   ├── node-definition.md   # Node definition template explanation  
│   └── publishing.md        # NPM publishing steps  
├── node/                    # Template implementation directory for nodes  
│   ├── <node-name>/         
│   │   ├── <node-name>.js   # Runtime implementation template  
│   │   ├── <node-name>.html # Editor UI definition template  
│   │   └── locales/         # i18n dictionary folder (recommended)  
│   └── index.js             # Entry point for registering all nodes  
├── manifest.json            # Required configuration file for MCU nodes  
└── examples/                # Sample flows (JSON)  
```

## ⚠️ Critical Information for Node-RED MCU

**If you're creating MCU-specific nodes, pay special attention to:**

1. **Dual Implementation Required**:
   - `<node-name>.js`: Stub for standard Node-RED (CommonJS only, no ESM)
   - `<node-name>.mcu.js`: Actual MCU implementation (ESM, runs on Moddable)

2. **manifest.json Must Use Object Format**:
   ```json
   {
     "modules": {
       "node-name": "./node/node-name/node-name.mcu"
     },
     "preload": "node-name"
   }
   ```
   ⚠️ Do NOT use array format or include `.js` extension!

3. **NO `export default` in MCU Implementation**:
   ```javascript
   // ❌ WRONG
   export default class MyNode extends Node { }
   
   // ✅ CORRECT
   class MyNode extends Node {
     static type = "my-node"
     static { RED.nodes.registerType(this.type, this) }
   }
   ```

4. **package.json Points to Stub Only**:
   ```json
   {
     "node-red": {
       "nodes": {
         "my-node": "node/my-node/my-node.js"
       }
     }
   }
   ```

See `docs/node-definition.md` for complete MCU-specific guidelines.

## Documentation

For more detailed information, please refer to the `docs/` folder, which includes:

- `folder-structure.md`: Details on the recommended project structure (including MCU dual-file pattern).
- `node-definition.md`: Guidelines for defining Node-RED nodes (includes critical MCU requirements).
- `publishing.md`: Steps for publishing to NPM and updating versions.

## Adding This Project as a Git Submodule

1. Navigate to your repository:
   ```bash
   cd /path/to/your/repository
   ```

2. Add this project:
   ```bash
   git submodule add https://github.com/404background/node-create-guidelines.git node-create-guidelines
   ```

3. Initialize and update:
   ```bash
   git pull
   ```
