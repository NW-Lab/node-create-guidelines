# Folder Structure Guide

This document explains the recommended folder structure for Node-RED node development projects, including special considerations for Node-RED MCU nodes.

## Standard Node-RED Structure

```
node-create-guidelines/  
├── README.md                # Main documentation  
├── package.json             # Package settings including "node-red.nodes" field  
├── node/                    # Implementation directory for nodes  
│   ├── <node-name>/         
│   │   ├── <node-name>.js   # Runtime implementation  
│   │   ├── <node-name>.html # Editor UI definition  
│   │   └── locales/         # i18n dictionary folder (recommended)  
│   │       ├── en-US/       # English translations
│   │       └── ja/          # Japanese translations
│   └── index.js             # Entry point for registering all nodes  
├── examples/                # Sample flows (JSON)  
├── images/                  # Visual representations of sample flows  
```

## Node-RED MCU Structure (Dual Implementation)

For MCU-specific nodes that run on Moddable SDK, you need **two separate implementations**:

```
node-create-guidelines/  
├── README.md                # Main documentation  
├── package.json             # Package settings (points to stub .js)  
├── manifest.json            # ⚠️ MCU module definition (object format!)  
├── node/                    
│   ├── <node-name>/         
│   │   ├── <node-name>.js       # ⚠️ STUB for standard Node-RED (CommonJS)
│   │   ├── <node-name>.mcu.js   # ⚠️ ACTUAL MCU implementation (ESM)
│   │   ├── <node-name>.html     # Editor UI (includes moddable_manifest)
│   │   └── locales/              
│   │       ├── en-US/       
│   │       │   ├── <node-name>.html
│   │       │   └── <node-name>.json
│   │       └── ja/          
│   │           ├── <node-name>.html
│   │           └── <node-name>.json
│   └── index.js             # Entry point (minimal for MCU nodes)  
├── examples/                # Sample flows  
├── images/                  
```

### Critical Files for MCU Nodes

#### 1. `package.json` (Points to Stub)
```json
{
  "node-red": {
    "nodes": {
      "my-node": "node/my-node/my-node.js"
    }
  }
}
```
⚠️ **Must point to `.js` stub, NOT `.mcu.js`!**

#### 2. `manifest.json` (Object Format)
```json
{
  "modules": {
    "my-node": "./node/my-node/my-node.mcu"
  },
  "preload": "my-node"
}
```
⚠️ **Must use object format, NOT array! No `.js` extension!**

#### 3. `<node-name>.js` (Stub - CommonJS Only)
```javascript
module.exports = function(RED) {
    function MyNode(config) {
        RED.nodes.createNode(this, config);
        this.status({fill: "red", shape: "ring", text: "MCU only"});
        this.error("This node runs only on Node-RED MCU");
    }
    RED.nodes.registerType("my-node", MyNode);
};
```
⚠️ **No ESM, no `import`, no `nodered` module reference!**

#### 4. `<node-name>.mcu.js` (MCU Implementation - ESM)
```javascript
import {Node} from "nodered";

class MyNode extends Node {
    onStart(config) {
        super.onStart(config);
        // MCU-specific implementation
    }
    
    onMessage(msg, done) {
        // Handle messages
        done();
    }
    
    static type = "my-node"
    static {
        RED.nodes.registerType(this.type, this)
    }
}
```
⚠️ **NO `export default`! Must use `static { ... }` registration!**

#### 5. `<node-name>.html` (Editor UI with moddable_manifest)
```html
<script type="text/javascript">
RED.nodes.registerType('my-node', {
    category: 'MCU',
    defaults: {
        name: {value: ""},
        // ... other properties ...
        moddable_manifest: {
            value: {
                include: [
                    {"git": "https://github.com/your-org/your-repo.git"}
                ]
            }
        }
    },
    // ... rest of configuration ...
});
</script>
```

## Why Dual Implementation?

| File | Purpose | Runtime | Format |
|------|---------|---------|--------|
| `.js` (stub) | Node-RED editor palette | Node.js host | CommonJS |
| `.mcu.js` | Actual MCU execution | Moddable SDK | ESM |
| `.html` | Editor UI | Browser | HTML/JS |

The stub ensures the node appears in the palette, while the MCU implementation runs on the device.

## Common Mistakes to Avoid

❌ Using array format in `manifest.json`:
```json
{"modules": ["path/to/file.js"]}  // WRONG
```

❌ Using `export default` in MCU implementation:
```javascript
export default class MyNode  // WRONG
```

❌ Pointing `package.json` to `.mcu.js`:
```json
{"my-node": "node/my-node/my-node.mcu.js"}  // WRONG
```

❌ Including `.js` extension in `manifest.json`:
```json
{"my-node": "./node/my-node/my-node.mcu.js"}  // WRONG
```

This structure ensures clarity, maintainability, and correct MCU operation.
