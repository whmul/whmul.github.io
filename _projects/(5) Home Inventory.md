---
name: Raspberry Pi Home Inventory
tools: [Raspberry Pi, Barcode Scanner, Python]
image: https://i.postimg.cc/yx2HKZ8N/IMG-20250823-154049.jpg
description: Keeping track of the pantry inventory with a barcode scanner and a Raspberry pi.
---

# This is a barcode-based home inventory manager.

## The problem

We keep a lot of canned goods and other products in the basement on a shelf. Unfortunately, the inconvenience required in manually checking it when we need something leads to us not knowing what we do or don't have, most of the time. The reasonable solution would be to build an inventory tracker! This system uses a Symbol LS2208 handheld USB Barcode Scanner and a classic Raspberry pi Model B+ I had from 2014. 

### How to use it?

1. Scan item barcodes:
    - By default, you are in "ADD MODE" (each barcode you scan, it adds 1 to that item's quantity).
2. Special (control) barcodes:

![Basic barcodes](https://i.postimg.cc/wvJPNGq9/basic.jpg)

> Switch to ADD mode: Scan code `400000111117`

> Switch to REMOVE mode: Scan code `400000333335`

> Print inventory: Scan code `400000666662`

> Delete an item: Scan code `400000555553`, then scan the product's barcode

> Add custom quantity: Scan code `400000222226`, then scan item and enter quantity

> Remove custom quantity: Scan code `400000444444`, then scan item and enter quantity

- These Barcodes were chosen specifically following UPC guidelines where the first digit includes reserved product codes for internal use.

3. If an item is new, it tries to fetch a name. If it can't, you're prompted to enter a name.

![Photo of barcodes](https://i.postimg.cc/zvyr4j5M/IMG-20250823-154101.jpg)

> I added two special barcodes for individual rolls of paper towels or toilet paper. These are handled by the quantity modes.

---

### How does it work? (Architecture basics)

- Inventory is a file (`inventory.json`), barcode to quantity/name.
- Barcode input is your main interaction (scan or type).
- Control barcodes change how the next scan behaves (e.g. to remove instead of add).
- OpenFoodFacts API: If the scanned barcode is new, the code tries to fetch its product name; otherwise, you are asked for a manual name.
- StateMachine: Remembers if you’re in ADD or REMOVE mode, and if you triggered a one-shot action (like delete).

---

![Symbol](https://i.postimg.cc/76s4Bcv0/IMG-20250823-154132.jpg)
> Symbol Barcode Scanner

![Example Output](https://i.postimg.cc/yx2HKZ8N/IMG-20250823-154049.jpg)
> Example Output

![The Raspberry pi](https://i.postimg.cc/k4X9vCH7/IMG-20250823-154146.jpg)
> Raspberry pi

---

### Code:

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


# Configuration
APP_NAME = "HomeInventory"
APP_VERSION = "1.0"
INVENTORY_FILE = "inventory.json"

# Control barcodes
ADD_MODE_BARCODE = "400000111117" # Persistent mode: add
REMOVE_MODE_BARCODE = "400000333335" # Persistent mode: remove
DELETE_ITEM_BARCODE = "400000555553" # One-shot action
ADDQTY_MODE_BARCODE = "400000222226" # One-shot action
REMOVEQTY_MODE_BARCODE = "400000444444" # One-shot action
PRINT_INVENTORY_BARCODE = "400000666662" # Function call

# Control barcode sets/maps (single source of truth)
CONTROL_BARCODES = {
    ADD_MODE_BARCODE,
    REMOVE_MODE_BARCODE,
    DELETE_ITEM_BARCODE,
    ADDQTY_MODE_BARCODE,
    REMOVEQTY_MODE_BARCODE,
    PRINT_INVENTORY_BARCODE,
}

PERSISTENT_MODES = {
    ADD_MODE_BARCODE: "add",
    REMOVE_MODE_BARCODE: "remove",
}

ONE_SHOT_ACTIONS = {
    DELETE_ITEM_BARCODE: "delete",
    ADDQTY_MODE_BARCODE: "addqty",
    REMOVEQTY_MODE_BARCODE: "removeqty",
}

# Logging setup
logging.basicConfig(
    level=logging.INFO,
    format="%(message)s",
)
logger = logging.getLogger(APP_NAME)

# OpenFoodFacts API
API = openfoodfacts.API(user_agent=f"{APP_NAME}/{APP_VERSION}")


# I/O abstraction
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


# Domain models
@dataclass
class Item:
    name: str
    quantity: int = 0

    def to_json(self) -> Dict:
        return asdict(self)

    @staticmethod
    def from_json(data: Dict) -> Item:
        return Item(name=data.get("Name", "Unknown"), quantity=int(data.get("Quantity", 0)))


class Mode(Enum):
    ADD = auto()
    REMOVE = auto()


class Action(Enum):
    NONE = auto()
    DELETE = auto()
    ADD_QTY = auto()
    REMOVE_QTY = auto()


# Inventory 
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
                payload = {bc: {"Name": item.name, "Quantity": item.quantity} for bc, item in self.data.items()}
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

    def resolve_product_name(self, barcode: str) -> str:
        # Use cached name if present
        if barcode in self.data and self.data[barcode].name:
            return self.data[barcode].name

        # Try to fetch from API
        fetched = self.fetch_product_name(barcode)
        if fetched:
            return fetched

        # Ask user for a friendly name
        user_input = self._io.input(
            f"Unknown product for barcode {barcode}. Enter a name (default: Miscellaneous): "
        ).strip()
        return user_input or "Miscellaneous"

    def add_item(self, barcode: str, qty: int) -> None:
        if self.is_control_barcode(barcode):
            logger.info("Scanned barcode is reserved and cannot be added to inventory.")
            return

        if barcode in self.data:
            item = self.data[barcode]
            item.quantity += qty
            logger.info(f"Added {qty} to {item.name} [{barcode}]. New quantity: {item.quantity}.")
        else:
            name = self.resolve_product_name(barcode)
            self.data[barcode] = Item(name=name, quantity=qty)
            logger.info(f"Added new item {name} [{barcode}] with quantity {qty}.")

        self.save_inventory()

    def remove_item(self, barcode: str, qty: int) -> None:
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
        for barcode, item in sorted(self.data.items(), key=lambda x: x[1].name.lower()):
            self._io.print(f"{item.name:30s} qty: {item.quantity:4d}  [{barcode}]")
        self._io.print("--- End ---\n")


# State machine
class StateMachine:
    def __init__(self):
        self.mode: Mode = Mode.ADD
        self.pending_action: Action = Action.NONE

    def set_persistent_mode(self, barcode: str) -> bool:
        if barcode not in PERSISTENT_MODES:
            return False

        name = PERSISTENT_MODES[barcode]
        self.mode = Mode.ADD if name == "add" else Mode.REMOVE
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
        }
        self.pending_action = mapping[ONE_SHOT_ACTIONS[barcode]]

        messages = {
            Action.DELETE: "DELETE mode: Next scan deletes the item, then returning to ADD mode.",
            Action.ADD_QTY: "ADD QTY mode: Next scan adds specified quantity, then returning to ADD mode.",
            Action.REMOVE_QTY: "REMOVE QTY mode: Next scan removes specified quantity, then returning to ADD mode.",
        }
        logger.warning(messages[self.pending_action])
        return True

    def complete_action_and_reset(self) -> None:
        self.pending_action = Action.NONE
        self.mode = Mode.ADD
        logger.info("Returned to ADD mode.")


# Main loop 
def main() -> None:
    io = IO()
    inventory = Inventory(INVENTORY_FILE, io=io)
    machine = StateMachine()

    logger.info(f"Starting in {machine.mode.name} mode.")
    logger.info("Awaiting barcode scans. Scan special barcodes to switch modes or print inventory.")

    try:
        while True:
            barcode = io.input("Scanner > ").strip()
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

            # Normal add/remove
            if barcode in CONTROL_BARCODES:
                logger.info("This barcode is reserved for control. No action taken.")
                continue

            if machine.mode == Mode.ADD:
                inventory.add_item(barcode, qty=1)
            elif machine.mode == Mode.REMOVE:
                inventory.remove_item(barcode, qty=1)
            else:
                logger.error("Unknown mode. Please restart the application.")
                break

    except KeyboardInterrupt:
        logger.info("\nExiting inventory scanner. Goodbye!")


if __name__ == "__main__":
    main()
```
