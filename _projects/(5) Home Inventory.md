---
name: Raspberry Pi Home Inventory
tools: [Raspberry Pi, Barcode Scanner, Python, Flask]
image: https://i.postimg.cc/yx2HKZ8N/IMG-20250823-154049.jpg
description: Keeping track of the pantry inventory with a barcode scanner and a Raspberry pi.
---

# This is a barcode-based home inventory manager.

## The problem

We keep a lot of canned goods and other products in the basement on a shelf. Unfortunately, the inconvenience required in manually checking it when we need something leads to us not knowing what we do or don't have, most of the time. The reasonable solution would be to build an inventory tracker! This system uses a Honeywell Xenon 1902 2D Barcode Scanner and a classic Raspberry pi Model B+ I had from 2014. 

![Scanner Setup in the Dark](https://i.postimg.cc/YCY5gNWw/IMG-20250926-001057.jpg)

### How to use it?

1. Scan item barcodes:
    - By default, you are in "ADD MODE" (each barcode you scan, it adds 1 to that item's quantity).
2. Special (control) barcodes:

![Basic barcodes](https://i.postimg.cc/xdZTYHwJ/newestbcpage.jpg)

> The top 4 datamatrix codes are **modes**, meaning they persist through scans. The Bottom 4 are **actions**, either they fire when you scan, or they need you to scan one item before returning to add mode.

> The Add/Remove quantity modes have you scan an item, then prompt you to enter a quantity. 

> I also added two special barcodes for individual rolls of paper towels or toilet paper (since they come in different sized packs). These are handled by the quantity modes.



---

### How does it work? (Architecture basics)

- Inventory is a file (`inventory.json`), barcode to quantity/name.
- Barcode input is your main interaction (scan or type).
- Control barcodes change how the next scan behaves (e.g. to remove instead of add).
- OpenFoodFacts API: If the scanned barcode is new, the code tries to fetch its product name; otherwise, you are asked for a manual name.
- StateMachine: Remembers if you’re in ADD or REMOVE mode, and if you triggered a one-shot action (like delete).


---

![Example Output](https://i.postimg.cc/yx2HKZ8N/IMG-20250823-154049.jpg)
> Example Output

![The Raspberry pi](https://i.postimg.cc/k4X9vCH7/IMG-20250823-154146.jpg)
> Raspberry pi

---

### Local Web Page

- I also have a Flask app that runs as a daemon on the raspberry pi, which hosts the json file to your browser, allowing you to sort by category and name, and allows you to increment and decrement the quantity from your computer.

![Local website frontend](https://i.postimg.cc/YjBv7gmf/frontend.png)
> Local website frontend

---

### Flask Code:

```python
from flask import Flask, jsonify, render_template_string, request
import json
import os
import threading
import pathlib

app = Flask(__name__)

INVENTORY_FILE = 'inventory.json'
inventory_lock = threading.Lock()

def safe_filename(requested):
    requested = os.path.basename((requested or '').strip())
    if not requested.lower().endswith('.json'):
        return INVENTORY_FILE
    # Prevent directory traversal
    p = pathlib.Path(requested)
    if p.parent != pathlib.Path('.'):
        return INVENTORY_FILE
    return str(p)

HTML_PAGE = '''
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Inventory Table</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 2em; }
        table { border-collapse: collapse; width: 100%; }
        th, td { border: 1px solid #ddd; padding: 8px; }
        th { background: #f2f2f2; cursor: pointer; }
        th.sorted-asc::after { content: " ▲"; }
        th.sorted-desc::after { content: " ▼"; }
        tr:nth-child(even) { background-color: #f9f9f9; }
        button { padding: 0 7px; font-size: 14px; line-height: 1; }
        button:active { background: #eee; }
        span.qty { min-width:2em; display:inline-block; text-align:center;}
    </style>
</head>
<body>
    <div>
        <label for="file-select"><b>Inventory List:</b></label>
        <select id="file-select"></select>
    </div>
    <div id="table-container">Loading...</div>
    <script>
let inventory = {};
let tableData = [];
let columns = [];
let sortCol = null;
let sortDir = 1;
let currentFile = "inventory.json";

function fetchFileList() {
    fetch('/list_json')
        .then(resp => resp.json())
        .then(files => {
            files.sort();
            let sel = document.getElementById('file-select');
            sel.innerHTML = "";
            files.forEach(f => {
                let opt = document.createElement("option");
                opt.value = f;
                opt.innerText = f;
                sel.appendChild(opt);
            });
            // Set currentFile to inventory.json, else first file
            if (!files.includes("inventory.json") && files.length > 0) {
                currentFile = files[0];
            } else {
                currentFile = "inventory.json";
            }
            sel.value = currentFile;
            sel.onchange = function() {
                currentFile = this.value;
                fetchInventory();
            };
            fetchInventory(); // Only call after dropdown is ready
        });
}

function fetchInventory() {
    fetch('/inventory.json?filename=' + encodeURIComponent(currentFile))
        .then(response => response.json())
        .then(data => {
            inventory = data;
            tableData = Object.entries(inventory).map(([upc, details]) => {
                return { "UPC": upc, ...details };
            });
            columns = Object.keys(tableData[0] || {});
            sortCol = null;
            renderTable();
        });
}

function renderTable() {
    if (columns.length === 0) {
        document.getElementById('table-container').innerHTML = "<p>No data in this file.</p>";
        return;
    }
    let html = '<table><thead><tr>';
    columns.forEach((col, idx) => {
        let className = (sortCol === col) ? ('sorted-' + (sortDir > 0 ? 'asc' : 'desc')) : '';
        html += `<th class="${className}" onclick="sortTable('${col}')">${col}</th>`;
    });
    html += '</tr></thead><tbody>';
    tableData.forEach(row => {
        html += '<tr>';
        columns.forEach(col => {
            if (col === "Quantity") {
                html += `<td>
                    <button onclick="updateQuantity('${row['UPC']}', -1)">&#8595;</button>
                    <span id="qty-${row['UPC']}" class="qty">${row[col]}</span>
                    <button onclick="updateQuantity('${row['UPC']}', 1)">&#8593;</button>
                </td>`;
            } else {
                html += `<td>${row[col] !== undefined ? row[col] : ''}</td>`;
            }
        });
        html += '</tr>';
    });
    html += '</tbody></table>';
    document.getElementById('table-container').innerHTML = html;
}

function sortTable(col) {
    if (sortCol === col) {
        sortDir *= -1;
    } else {
        sortCol = col;
        sortDir = 1;
    }
    tableData.sort((a, b) => {
        let x = a[col], y = b[col];
        if (!isNaN(parseFloat(x)) && !isNaN(parseFloat(y))) {
            return (parseFloat(x) - parseFloat(y)) * sortDir;
        }
        return (String(x).localeCompare(String(y))) * sortDir;
    });
    renderTable();
}

function updateQuantity(upc, delta) {
    let endpoint = '/api/' + (delta > 0 ? "increment" : "decrement");
    fetch(endpoint + '?filename=' + encodeURIComponent(currentFile), {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ upc: upc })
    })
    .then(response => response.json())
    .then(data => {
        if (data.success) {
            document.getElementById("qty-" + upc).innerText = data.quantity;
        } else {
            alert("Update failed");
        }
    });
}

window.sortTable = sortTable;
window.updateQuantity = updateQuantity;

fetchFileList();

    </script>
</body>
</html>
'''

@app.route('/')
def index():
    return render_template_string(HTML_PAGE)

@app.route('/inventory.json')
def inv():
    file = safe_filename(request.args.get('filename', INVENTORY_FILE))
    with inventory_lock:
        if not os.path.exists(file):
            return jsonify({})
        with open(file, "r", encoding="utf-8") as f:
            inv_data = json.load(f)
    return jsonify(inv_data)

@app.route('/api/increment', methods=['POST'])
def api_increment():
    file = safe_filename(request.args.get('filename', INVENTORY_FILE))
    data = request.get_json(force=True)
    upc = data.get('upc')
    with inventory_lock:
        if not os.path.exists(file):
            return jsonify({"success": False}), 400
        with open(file, "r", encoding="utf-8") as f:
            inv_data = json.load(f)
        if upc in inv_data:
            inv_data[upc]['Quantity'] = int(inv_data[upc].get('Quantity', 0)) + 1
            with open(file, "w", encoding="utf-8") as f:
                json.dump(inv_data, f, indent=4)
            return jsonify({"success": True, "quantity": inv_data[upc]['Quantity']})
    return jsonify({"success": False}), 400

@app.route('/api/decrement', methods=['POST'])
def api_decrement():
    file = safe_filename(request.args.get('filename', INVENTORY_FILE))
    data = request.get_json(force=True)
    upc = data.get('upc')
    with inventory_lock:
        if not os.path.exists(file):
            return jsonify({"success": False}), 400
        with open(file, "r", encoding="utf-8") as f:
            inv_data = json.load(f)
        if upc in inv_data:
            inv_data[upc]['Quantity'] = max(0, int(inv_data[upc].get('Quantity', 0)) - 1)
            with open(file, "w", encoding="utf-8") as f:
                json.dump(inv_data, f, indent=4)
            return jsonify({"success": True, "quantity": inv_data[upc]['Quantity']})
    return jsonify({"success": False}), 400

@app.route('/list_json')
def list_json():
    files = [f for f in os.listdir(".") if f.lower().endswith(".json")]
    files.sort()
    return jsonify(files)

if __name__ == "__main__":
    app.run(host='192.168.1.180', port=5000, debug=True)
```

### Scanner Code:

```python
from __future__ import annotations

import json
import logging
import os
import tempfile
from dataclasses import dataclass, asdict
from enum import Enum, auto
from typing import Dict, Optional, Callable

import openfoodfacts


#  Configuration 
APP_NAME = "HomeInventory"
APP_VERSION = "2.1"
INVENTORY_FILE = "inventory.json"

# Control barcodes
ADD_MODE_BARCODE = "400000111117"         # Persistent mode: add
REMOVE_MODE_BARCODE = "400000333335"      # Persistent mode: remove
DELETE_ITEM_BARCODE = "400000555553"      # One-shot action
ADDQTY_MODE_BARCODE = "400000222226"      # One-shot action
REMOVEQTY_MODE_BARCODE = "400000444444"   # One-shot action
PRINT_INVENTORY_BARCODE = "400000666662"  # Function call

PRINT_QTY_MODE_BARCODE = "420088557732"   # Persistent mode: print_qty
RENAME_NEXT_BARCODE = "499775224322"      # One-shot action

# Control barcode sets/maps (single source of truth)
CONTROL_BARCODES = {
    ADD_MODE_BARCODE,
    REMOVE_MODE_BARCODE,
    DELETE_ITEM_BARCODE,
    ADDQTY_MODE_BARCODE,
    REMOVEQTY_MODE_BARCODE,
    PRINT_INVENTORY_BARCODE,
    PRINT_QTY_MODE_BARCODE,
    RENAME_NEXT_BARCODE,
}

PERSISTENT_MODES = {
    ADD_MODE_BARCODE: "add",
    REMOVE_MODE_BARCODE: "remove",
    PRINT_QTY_MODE_BARCODE: "printqty",
}

ONE_SHOT_ACTIONS = {
    DELETE_ITEM_BARCODE: "delete",
    ADDQTY_MODE_BARCODE: "addqty",
    REMOVEQTY_MODE_BARCODE: "removeqty",
    RENAME_NEXT_BARCODE: "rename",
}

# Logging setup
logging.basicConfig(
    level=logging.INFO,
    format="%(message)s",
)
logger = logging.getLogger(APP_NAME)

# OpenFoodFacts API
API = openfoodfacts.API(user_agent=f"{APP_NAME}/{APP_VERSION}")


#  I/O abstraction 
class IO:
    def input(self, prompt: str) -> str:
        return input(prompt)

    def print(self, message: str) -> None:
        print(message)

    def prompt_int(self, prompt: str, default: int = 1, min_value: int = 1) -> int:
        raw = self.input(prompt).strip()
        if not raw:
            return default
        try:
            value = int(raw)
            return value if value >= min_value else default
        except ValueError:
            return default


#  Domain models 
@dataclass
class Item:
    name: str
    quantity: int = 0
    category: str = "Other"
    display_name: Optional[str] = None

    def to_json(self) -> Dict:
        return {
            "Name": self.name,
            "Display Name": self.display_name if self.display_name is not None else self.name,
            "Quantity": self.quantity,
            "Category": self.category,
        }

    @staticmethod
    def from_json(data: Dict) -> Item:
        name = data.get("Name", "Unknown")
        # Use the display_name from json if present, else fallback to name
        display_name = data.get("Display Name", name)
        return Item(
            name=name,
            quantity=int(data.get("Quantity", 0)),
            category=data.get("Category", "Other"),
            display_name=display_name,
        )

class Mode(Enum):
    ADD = auto()
    REMOVE = auto()
    PRINT_QTY = auto()


class Action(Enum):
    NONE = auto()
    DELETE = auto()
    ADD_QTY = auto()
    REMOVE_QTY = auto()
    RENAME = auto()


#  Inventory 
class Inventory:
    def __init__(self, filename: str, api=API, io: Optional[IO] = None):
        self.filename = filename
        self._api = api
        self._io = io or IO()
        self.data: Dict[str, Item] = {}
        self.load_inventory()

    def is_control_barcode(self, barcode: str) -> bool:
        return barcode in CONTROL_BARCODES

    def load_inventory(self) -> None:
        if not os.path.exists(self.filename):
            self.data = {}
            self.save_inventory()
            logger.info(f"{self.filename} not found. Created a new inventory file.")
            return

        try:
            with open(self.filename, "r", encoding="utf-8") as f:
                raw = json.load(f) or {}
            self.data = {bc: Item.from_json(info) for bc, info in raw.items()}
            logger.info(f"Loaded inventory from {self.filename}.")
        except (json.JSONDecodeError, OSError) as e:
            logger.warning(f"Error reading {self.filename} ({e}). Starting with empty inventory.")
            self.data = {}

    def save_inventory(self) -> None:
        # Atomic write to reduce the chance of file corruption
        tmp_fd, tmp_path = tempfile.mkstemp(prefix="inv_", suffix=".json", dir=os.path.dirname(self.filename) or None)
        try:
            with os.fdopen(tmp_fd, "w", encoding="utf-8") as tmp_file:
                #payload = {bc: {"Name": item.name, "Quantity": item.quantity, "Category": item.category} for bc, item in self.data.items()}
                payload = {
                    bc: item.to_json()
                    for bc, item in self.data.items()
                }
                json.dump(payload, tmp_file, indent=4)
            os.replace(tmp_path, self.filename)
            logger.debug(f"Inventory saved to {self.filename}.")
        except OSError as e:
            logger.error(f"Error saving inventory: {e}")
            # Clean up temp file if replace failed
            try:
                os.remove(tmp_path)
            except OSError:
                pass

    def fetch_product_name(self, barcode: str) -> Optional[str]:
        try:
            product = self._api.product.get(barcode, fields=["code", "product_name"])
            if product and product.get("product_name"):
                return str(product["product_name"])
        except Exception as e:
            logger.debug(f"OpenFoodFacts API error for {barcode}: {e}")
        return None

    def resolve_product_name(self, barcode: str) -> (str, str):
        # Use cached name/category if present
        if barcode in self.data and self.data[barcode].name:
            return self.data[barcode].name, self.data[barcode].category

        # Try to fetch from API
        fetched = self.fetch_product_name(barcode)
        if fetched:
            return fetched, "Food"

        # Ask user for a friendly name and food prompt
        user_input = self._io.input(
            f"Unknown product for barcode {barcode}. Enter a name (default: Miscellaneous): "
        ).strip()
        name = user_input or "Miscellaneous"

        is_food_str = self._io.input(
            "Is this a food item? y/n (default n): "
        ).strip().lower()
        category = "Food" if is_food_str == "y" else "Other"

        return name, category

    def add_item(self, barcode: str, qty: int) -> None:
        self.load_inventory()
        if self.is_control_barcode(barcode):
            logger.info("Scanned barcode is reserved and cannot be added to inventory.")
            return

        if barcode in self.data:
            item = self.data[barcode]
            item.quantity += qty
            logger.info(f"Added {qty} to {item.name} [{barcode}]. New quantity: {item.quantity}.")
        else:
            name, category = self.resolve_product_name(barcode)
            #self.data[barcode] = Item(name=name, quantity=qty, category=category)
            self.data[barcode] = Item(name=name, quantity=qty, category=category, display_name=name)
            logger.info(f"Added new item {name} [{barcode}] (Category: {category}) with quantity {qty}.")

        self.save_inventory()

    def remove_item(self, barcode: str, qty: int) -> None:
        self.load_inventory()
        if self.is_control_barcode(barcode):
            logger.info("Scanned barcode is reserved and cannot be removed from inventory.")
            return

        item = self.data.get(barcode)
        if not item:
            logger.info(f"Barcode {barcode} not found in inventory. No action taken.")
            return

        prev = item.quantity
        item.quantity = max(0, item.quantity - qty)
        logger.info(f"Removed {min(qty, prev)} from {item.name} [{barcode}]. New quantity: {item.quantity}.")
        self.save_inventory()

    def delete_item(self, barcode: str) -> None:
        self.load_inventory()
        if self.is_control_barcode(barcode):
            logger.info("Scanned barcode is reserved and cannot be deleted from inventory.")
            return

        item = self.data.pop(barcode, None)
        if item:
            logger.info(f"Deleted {item.name} [{barcode}] from inventory.")
            self.save_inventory()
        else:
            logger.info(f"Barcode {barcode} not found in inventory. No action taken.")

    def print_inventory(self) -> None:
        if not self.data:
            self._io.print("\n--- Inventory is empty ---\n")
            return

        self._io.print("\n--- Inventory ---")
        #for barcode, item in sorted(self.data.items(), key=lambda x: x[1].name.lower()):
        #    self._io.print(f"{item.name:30s} qty: {item.quantity:4d}  [{barcode}]")
        for barcode, item in sorted(self.data.items(), key=lambda x: (x[1].display_name or x[1].name).lower()):
            self._io.print(f"{item.display_name or item.name:30s} qty: {item.quantity:4d}  [{barcode}]")
        self._io.print("--- End ---\n")

    def print_item_qty(self, barcode: str, io_: IO = None) -> None:
        io = io_ or self._io
        self.load_inventory()
        item = self.data.get(barcode)
        if item:
            io.print(f"{item.display_name or item.name}: qty = {item.quantity}  [{barcode}]")
        else:
            io.print(f"Barcode {barcode} not found in inventory.")
            
    def rename_item(self, barcode: str) -> None:
        self.load_inventory()
        item = self.data.get(barcode)
        if not item:
            logger.info(f"Barcode {barcode} not found in inventory. No action taken.")
            return
        new_name = self._io.input(f"Enter new display name for '{item.display_name or item.name}' [{barcode}]: ").strip()
        if new_name:
            item.display_name = new_name
            logger.info(f"Renamed item [{barcode}] to '{new_name}'.")
            self.save_inventory()
        else:
            logger.info("Empty name entered. No changes made.")


#  State machine 
class StateMachine:
    def __init__(self):
        self.mode: Mode = Mode.ADD
        self.pending_action: Action = Action.NONE

    def set_persistent_mode(self, barcode: str) -> bool:
        if barcode not in PERSISTENT_MODES:
            return False

        name = PERSISTENT_MODES[barcode]
        if name == "add":
            self.mode = Mode.ADD
        elif name == "remove":
            self.mode = Mode.REMOVE
        elif name == "printqty":
            self.mode = Mode.PRINT_QTY
        else:
            return False

        logger.info(f"Switched to {name.upper()} mode.")
        self.pending_action = Action.NONE
        return True

    def set_one_shot_action(self, barcode: str) -> bool:
        if barcode not in ONE_SHOT_ACTIONS:
            return False

        mapping = {
            "delete": Action.DELETE,
            "addqty": Action.ADD_QTY,
            "removeqty": Action.REMOVE_QTY,
            "rename": Action.RENAME,
        }
        self.pending_action = mapping[ONE_SHOT_ACTIONS[barcode]]

        messages = {
            Action.DELETE: "DELETE mode: Next scan deletes the item, then returning to ADD mode.",
            Action.ADD_QTY: "ADD QTY mode: Next scan adds specified quantity, then returning to ADD mode.",
            Action.REMOVE_QTY: "REMOVE QTY mode: Next scan removes specified quantity, then returning to ADD mode.",
            Action.RENAME: "RENAME mode: Next scan renames the item, then returning to ADD mode.",   # <-- NEW
        }
        logger.warning(messages[self.pending_action])
        return True

    def complete_action_and_reset(self) -> None:
        self.pending_action = Action.NONE
        self.mode = Mode.ADD
        logger.info("Returned to ADD mode.")


#  Main loop 
def main() -> None:
    io = IO()
    inventory = Inventory(INVENTORY_FILE, io=io)
    machine = StateMachine()

    logger.info(f"Starting in {machine.mode.name} mode.")
    logger.info("Awaiting barcode scans. Scan special barcodes to switch modes or print inventory.")

    try:
        while True:
            # Determine prompt mode for display
            if machine.pending_action == Action.DELETE:
                prompt_mode = "DELETE"
            elif machine.pending_action == Action.ADD_QTY:
                prompt_mode = "ADDQTY"
            elif machine.pending_action == Action.REMOVE_QTY:
                prompt_mode = "REMOVEQTY"
            elif machine.pending_action == Action.RENAME:
                prompt_mode = "RENAME"
            elif machine.mode == Mode.PRINT_QTY:
                prompt_mode = "PRINTQTY"
            else:
                prompt_mode = machine.mode.name   # "ADD" or "REMOVE"
        
            barcode = io.input(f"Scanner - {prompt_mode} > ").strip()
            if not barcode:
                continue

            # Print inventory
            if barcode == PRINT_INVENTORY_BARCODE:
                inventory.print_inventory()
                continue

            # Mode switching
            if machine.set_persistent_mode(barcode):
                continue

            # One-shot modes (delete/addqty/removeqty)
            if machine.set_one_shot_action(barcode):
                continue

            # Handle pending one-shot action
            if machine.pending_action == Action.DELETE:
                inventory.delete_item(barcode)
                machine.complete_action_and_reset()
                continue

            if machine.pending_action == Action.ADD_QTY:
                qty = io.prompt_int("Enter quantity to add (default 1): ", default=1, min_value=1)
                inventory.add_item(barcode, qty=qty)
                machine.complete_action_and_reset()
                continue

            if machine.pending_action == Action.REMOVE_QTY:
                qty = io.prompt_int("Enter quantity to remove (default 1): ", default=1, min_value=1)
                inventory.remove_item(barcode, qty=qty)
                machine.complete_action_and_reset()
                continue

            if machine.pending_action == Action.RENAME:
                inventory.rename_item(barcode)
                machine.complete_action_and_reset()
                continue

            # Normal add/remove
            if barcode in CONTROL_BARCODES:
                logger.info("This barcode is reserved for control. No action taken.")
                continue

            if machine.mode == Mode.ADD:
                inventory.add_item(barcode, qty=1)
            elif machine.mode == Mode.REMOVE:
                inventory.remove_item(barcode, qty=1)
            elif machine.mode == Mode.PRINT_QTY:
                inventory.print_item_qty(barcode, io)
            else:
                logger.error("Unknown mode. Please restart the application.")
                break

    except KeyboardInterrupt:
        logger.info("\nExiting inventory scanner. Goodbye!")


if __name__ == "__main__":
    main()
```
