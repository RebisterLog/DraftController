# DraftController
<img width="90" height="20" alt="image" src="https://github.com/user-attachments/assets/3099a728-bd84-44b2-984c-d5aad9268990" />
<img width="101" height="20" alt="image" src="https://github.com/user-attachments/assets/05045452-667c-4a2c-bfac-7d3ecae0b7c0" />

DraftController module provides utilities for manipulating nested tables, including path-based value access, table merging based on patterns, and generating change logs between table versions. Ideal for data migration and state management scenarios where schema evolution is required.

<h2>Core Features</h2>

- 🌳 Traverse deeply nested tables with dot-path notation
- 🔀 Merge tables while preserving default structures
- 📝 Generate detailed change logs between table versions
- ⚡ Efficient path-based get/set operations
- 🔍 Detect additions, removals, and modifications automatically

# <h1>Function Reference</h1>

## getTabSize(table: any): number

Counts all keys in a table (including non-array keys).<br>
<b>Example:</b>

```luau
local myTable = {a = 1, b = 2, [3] = "three"}
local count = DraftController.getTabSize(myTable)  -- Returns 3
```

## getTabIndexesTree(table: {[any]: any}, prefix: string?): {string}

Recursively collects all dot-separated paths to leaf values in a table. Empty subtables are treated as leaves.<br>
<b>Example:</b>

```luau
local data = {
    health = 100,
    inventory = {
        weapons = {"sword"},
        potions = {}
    }
}

local paths = DraftController.getTabIndexesTree(data)
-- Returns: {"health", "inventory.weapons.1", "inventory.potions"}
```
## setValueOnPath(table: {[any]: any}, path: string, value: any): ()

Sets a value at a dot-separated path, creating intermediate tables as needed.<br>
<b>Example:</b>

```luau
local playerData = {}
DraftController.setValueOnPath(playerData, "stats.level", 5)
DraftController.setValueOnPath(playerData, "inventory.weapons[1].name", "Excalibur")

-- Result:
-- {
--     stats = {level = 5},
--     inventory = {
--         weapons = {
--             [1] = {name = "Excalibur"}
--         }
--     }
-- }
```

## getValueOnPath(table: {[any]: any}, path: string, maxLevel: number?): (any?, number)

Retrieves value at a dot-separated path with optional depth limiting.<br>
<b>Basic Usage:</b>

```luau
local player = {stats = {level = 5, xp = 100}}
local level = DraftController.getValueOnPath(player, "stats.level")  -- Returns 5, 2
```

<b>With Depth Limit</b>

```luau
local player = {
    inventory = {
        weapons = {sword = true},
        armor = {chest = "iron"}
    }
}

-- Get only the weapons table (depth=2)
local weapons, depth = DraftController.getValueOnPath(player, "inventory.weapons.sword", 2)
-- Returns: {sword = true}, 2
```

## merge(pattern: {[any]: any}, template: {[any]: any}): {[any]: any}

Merges two tables where:
- Values from template override pattern at matching paths
- Missing paths in template use values from pattern
- Only paths existing in pattern are considered

<b>Example:</b>

```luau
local defaultPattern = {
    coins = 0,
    items = {potion = 1},
    settings = {volume = 50}
}

local userData = {
    coins = 10,
    items = {elixir = 1},  -- Replaces entire items subtree
    settings = {brightness = 75}  -- Merges with existing settings
}

local result = DraftController.merge(defaultPattern, userData)

-- Result:
-- {
--     coins = 10,           -- Overridden by userData
--     items = {elixir = 1}, -- Entire subtree replaced
--     settings = {          -- Partial merge
--         volume = 50,      -- From pattern (not in userData)
--         brightness = 75   -- From userData
--     }
-- }
```

## conversion(oldVersion: {[any]: any}, newVersion: {[any]: any}): {{action: string, path: string, value: any}}

Generates a changelog comparing two table versions. Detects added, removed, and modified leaf values.<br>
<b>Example:</b>

```luau
local oldData = {
    gold = 100,
    pets = {
        ["cat"] = {name = "Whiskers", level = 3}
    }
}

local newData = {
    gold = 200,  -- Modified
    energy = 50, -- Added
    pets = {
        ["cat"] = {name = "Whiskers", level = 5},  -- Modified
        ["dog"] = {name = "Rex"}                   -- Added
    }
}

local changes = DraftController.conversion(oldData, newData)

--[[
Returns:
{
    {
        action = "change",
        path = "gold",
        value = 200
    },
    {
        action = "add",
        path = "energy",
        value = 50
    },
    {
        action = "change",
        path = "pets.cat.level",
        value = 5
    },
    {
        action = "add",
        path = "pets.dog.name",
        value = "Rex"
    }
}
--]]
```

# Full Practical Example

```luau
local DraftController = require(path.to.DraftController)

-- Default data structure with fallback values
local DEFAULT_PATTERN = {
    Coins = 0,
    Inventory = {},
    Stats = {
        Level = 1,
        XP = 0
    },
    Settings = {
        Volume = 50,
        Notifications = true
    }
}

-- User's saved data (old version)
local OLD_SAVE = {
    Coins = 150,
    Inventory = {
        {Name = "Wooden Sword", Quantity = 1}
    },
    Stats = {
        Level = 5,
        XP = 2500
    }
}

-- Updated game schema (new version)
local NEW_SCHEMA = {
    Coins = 0,  -- Reset coins on update
    Crystals = 100,  -- New currency type
    Inventory = {},  -- Wipe inventory
    Stats = {
        Level = 1,
        XP = 0,
        Stamina = 100  -- New stat
    },
    Settings = {
        Volume = 75,  -- Changed default
        Brightness = 50  -- New setting
    }
}

-- 1. Generate upgrade changes
local changes = DraftController.conversion(OLD_SAVE, NEW_SCHEMA)

-- 2. Apply defaults to existing data
local upgradedSave = DraftController.merge(DEFAULT_PATTERN, OLD_SAVE)

-- 3. Apply schema changes
for _, change in ipairs(changes) do
    if change.action == "add" then
        DraftController.setValueOnPath(upgradedSave, change.path, change.value)
    elseif change.action == "change" then
        DraftController.setValueOnPath(upgradedSave, change.path, change.value)
    end
    -- We ignore "remove" actions for this upgrade scenario
end

--[[
upgradedSave now contains:
{
    Coins = 150,  -- Preserved from old save
    Crystals = 100,  -- Added from new schema
    Inventory = {},  -- Wiped per new schema
    Stats = {
        Level = 5,    -- Preserved
        XP = 2500,    -- Preserved
        Stamina = 100 -- Added from new schema
    },
    Settings = {
        Volume = 50,          -- From DEFAULT_PATTERN (user's old value wasn't set)
        Notifications = true, -- From DEFAULT_PATTERN
        Brightness = 50       -- Added from new schema
    }
}
--]]
```
