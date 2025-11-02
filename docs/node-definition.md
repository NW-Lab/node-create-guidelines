# Node Definition Template Guide

This document explains how to define a Node-RED node using the provided templates, following the guidelines from the [Node-RED documentation](https://nodered.org/docs/creating-nodes).

## Naming Conventions
- Node names should be lowercase and use hyphens (`-`) to separate words (e.g., `example-node`).
- Avoid using underscores (`_`) or camelCase in node names.
- The name should clearly reflect the node's purpose.

## General Guidelines
Nodes should:
- Be well-defined in their purpose. Avoid creating nodes that expose every possible option of an API. Instead, focus on the most commonly used and essential features to keep the node simple and user-friendly.
- Be simple to use, regardless of the underlying functionality. Hide complexity and avoid jargon.
- Be forgiving in what types of message properties they accept (e.g., strings, numbers, booleans, Buffers, objects, arrays, or nulls).
- Be consistent in what they send and document the properties they add to messages.
- Catch errors to prevent the entire flow from stopping. Register error handlers for any asynchronous calls.
- Support multiple instances of the same node in a flow. If necessary, use a configuration node to manage shared resources or common settings across multiple nodes.

## Runtime Implementation (`<node-name>.js`)
- Use `RED.nodes.createNode(this, config)` to initialize the node.
- Implement the node's functionality in the `on('input', ...)` handler.
- Use `node.error()` to handle errors gracefully.
- Add status indicators using `node.status()` to provide feedback to the user.

## Editor UI Definition (`<node-name>.html`)
- Define the node's properties and UI elements using `<script type="text/html">`.
- Include help text and descriptions for each property.
- Use `RED.nodes.registerType()` to register the node and define its category, color, inputs, outputs, and defaults.

## Internationalization
- Use the `locales/` folder to provide translations for the node's labels and messages.
- Follow the [Node-RED Internationalisation Guide](https://nodered.org/docs/creating-nodes/internationalisation) for best practices.

## Examples
- Provide sample flows in the `examples/` folder to demonstrate the node's functionality.
- Include screenshots and import instructions in the `README.md` file.

---

## 🚨 Node-RED MCU Specific Guidelines

### Overview

Node-RED MCU nodes run on **Moddable SDK** (embedded JavaScript runtime), not Node.js. This requires a **dual implementation approach**:

1. **Stub** (`.js`): CommonJS file for Node-RED editor (palette display only)
2. **MCU Implementation** (`.mcu.js`): ESM file for actual device execution

### Critical Rules for MCU Nodes

#### ✅ Rule 1: Separate Stub and MCU Files

**Stub (`<node-name>.js`)** - CommonJS, Node-RED host:
```javascript
module.exports = function(RED) {
    function MyNode(config) {
        RED.nodes.createNode(this, config);
        this.status({fill: "red", shape: "ring", text: "MCU only"});
        this.error("This node runs only on Node-RED MCU (Moddable)");
    }
    RED.nodes.registerType("my-node", MyNode);
};
```

**MCU Implementation (`<node-name>.mcu.js`)** - ESM, Moddable:
```javascript
import {Node} from "nodered";
import Timer from "timer";

class MyNode extends Node {
    onStart(config) {
        super.onStart(config);
        // Initialize hardware (I2C, GPIO, etc.)
    }
    
    onMessage(msg, done) {
        // Process message
        done();
    }
    
    onStop() {
        // Cleanup resources
    }
    
    static type = "my-node"
    static {
        RED.nodes.registerType(this.type, this)
    }
}
```

⚠️ **Common Mistakes:**
- ❌ Using `import` in stub → Use `module.exports` only
- ❌ Using `export default` in MCU → Use `static { ... }` registration
- ❌ Referencing `nodered` in stub → Keep stub minimal

---

#### ✅ Rule 2: manifest.json Must Use Object Format

❌ **WRONG (Array Format):**
```json
{
  "modules": [
    "node/my-node/my-node.mcu.js"
  ],
  "preload": []
}
```

✅ **CORRECT (Object Format):**
```json
{
  "modules": {
    "my-node": "./node/my-node/my-node.mcu"
  },
  "preload": "my-node"
}
```

**Key Points:**
- `modules` is an **object** with `key: path` pairs
- Path has **NO `.js` extension**
- `preload` is a **string** (module name), not array

---

#### ✅ Rule 3: package.json Points to Stub Only

```json
{
  "node-red": {
    "nodes": {
      "my-node": "node/my-node/my-node.js"
    }
  }
}
```

⚠️ **Never point to `.mcu.js`!** This causes `ERR_MODULE_NOT_FOUND: 'nodered'` error.

---

#### ✅ Rule 4: NO `export default` in MCU Implementation

❌ **WRONG:**
```javascript
export default class MyNode extends Node {
  // ...
}
```

✅ **CORRECT:**
```javascript
class MyNode extends Node {
  // ...
  
  static type = "my-node"
  static {
    RED.nodes.registerType(this.type, this)
  }
}
```

**Reason:** Node-RED MCU does not recognize `export default`. Use static registration block.

---

#### ✅ Rule 5: Editor HTML Must Include moddable_manifest

```html
<script type="text/javascript">
RED.nodes.registerType('my-node', {
    category: 'MCU',
    color: '#a6bbcf',
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
    inputs: 1,
    outputs: 0,
    icon: "bridge.svg",
    label: function() {
        return this.name || "My Node";
    }
});
</script>
```

**Options for `include`:**
- Git repository: `{"git": "https://github.com/..."}`
- Local path: `{"path": "./submodules/your-repo/manifest.json"}`

⚠️ **Important:** Use your **Fork URL** if you forked the repository!

---

#### ✅ Rule 6: Type Name Must Match Everywhere

The node type string must be **identical** in all these places:

| File | Location | Example |
|------|----------|---------|
| `.html` | `<script data-template-name="...">` | `"my-node"` |
| `.html` | `RED.nodes.registerType('...', {})` | `"my-node"` |
| `.js` (stub) | `RED.nodes.registerType("...", ...)` | `"my-node"` |
| `.mcu.js` | `static type = "..."` | `"my-node"` |
| `package.json` | `"node-red": {"nodes": {"...": ...}}` | `"my-node"` |
| `manifest.json` | `"modules": {"...": ...}` | `"my-node"` |

❌ One mismatch → "Disabling unsupported node type" error!

---

### MCU Hardware Integration Examples

#### I2C Device Control

```javascript
import {Node} from "nodered";

class I2CNode extends Node {
    #i2c;
    
    onStart(config) {
        super.onStart(config);
        
        const io = globalThis.device?.io;
        if (!io?.I2C) {
            this.status({fill: "red", shape: "ring", text: "no I2C"});
            return;
        }
        
        this.#i2c = new io.I2C({
            address: config.address || 0x64,
            data: config.sda || 21,
            clock: config.scl || 22,
            hz: config.hz || 400000
        });
        
        this.status({fill: "green", shape: "dot", text: "ready"});
    }
    
    onMessage(msg, done) {
        try {
            // Write to I2C register
            this.#i2c.write(Uint8Array.of(0x00, msg.payload));
            done();
        } catch (e) {
            this.error(`I2C error: ${e.message}`);
            done(e);
        }
    }
    
    onStop() {
        this.#i2c?.close();
    }
    
    static type = "i2c-node"
    static { RED.nodes.registerType(this.type, this) }
}
```

#### Using Timers

```javascript
import {Node} from "nodered";
import Timer from "timer";

class TimerNode extends Node {
    onMessage(msg, done) {
        Timer.set(() => {
            this.send({payload: "timeout"});
            done();
        }, 1000);
    }
    
    static type = "timer-node"
    static { RED.nodes.registerType(this.type, this) }
}
```

#### Debug Logging

```javascript
trace(`MyNode: Processing value=${value}\n`);
```

Use `trace()` for debug output visible in xsbug debugger.

---

### Common MCU Errors and Solutions

| Error | Cause | Solution |
|-------|-------|----------|
| `Disabling unsupported node type` | manifest.json array format | Use object format |
| | Missing `export default` → wrong! | Remove `export default`, use `static { ... }` |
| | Type name mismatch | Check all 6 locations for consistency |
| | Missing moddable_manifest | Add to HTML defaults |
| `ERR_MODULE_NOT_FOUND: 'nodered'` | package.json points to `.mcu.js` | Point to stub `.js` instead |
| | Stub uses `import` or ESM | Use CommonJS `module.exports` only |
| `No 'creation' found` | Running `mcconfig` manually | Use Node-RED MCU Deploy button |
| Module not found | Wrong path in manifest.json | Use `./node/...` and no `.js` extension |

---

### Deployment

1. **Standard Node-RED** (for palette display):
   ```bash
   npm install /path/to/your/package
   node-red
   ```
   Node should appear in palette with "MCU only" status.

2. **Node-RED MCU** (for device execution):
   - Open Node-RED MCU interface
   - Add your node to flow
   - Click **Deploy** button
   - MCU automatically builds and flashes device
   - Monitor output in xsbug debugger

⚠️ **Do NOT run `mcconfig` manually** - Node-RED MCU handles the build process.

---

### Testing Checklist

- [ ] Stub appears in palette (standard Node-RED)
- [ ] Stub shows "MCU only" status
- [ ] manifest.json uses object format
- [ ] manifest.json has no `.js` extension
- [ ] MCU implementation has no `export default`
- [ ] Type names match in all 6 locations
- [ ] moddable_manifest includes your repository
- [ ] Device builds and runs (Deploy from MCU interface)
- [ ] xsbug shows debug logs
- [ ] Hardware functions correctly

---

## References

- [Node-RED Documentation](https://nodered.org/docs/creating-nodes)
- [Node-RED MCU](https://github.com/phoddie/node-red-mcu)
- [Moddable SDK](https://github.com/Moddable-OpenSource/moddable)
- [Node-RED Internationalisation](https://nodered.org/docs/creating-nodes/internationalisation)
