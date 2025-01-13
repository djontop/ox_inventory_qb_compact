# ox_inventory Setup and Conversion Guide

This guide provides step-by-step instructions for integrating and converting data for **ox_inventory** with QBCore. Follow the instructions carefully to ensure proper functionality.

---

## Prerequisites
- Ensure **MySQL** is set up and accessible.
- Install **ox_inventory** and QBCore framework.

---

## Files to Modify

### 1. **ox_inventory/setup/convert.lua**
This script is responsible for converting QBCore data into the new format compatible with **ox_inventory**. Place the script in the following path:
```
ox_inventory/setup/convert.lua - line  137
```
```lua
local function ConvertQB()
	if started then
		return warn('Data is already being converted, please wait..')
	end

	started = true
	local users = MySQL.query.await('SELECT citizenid, inventory, money FROM players')
	if not users then return end
	local count = 0
	local parameters = {}

	for i = 1, #users do
		local inventory, slot = {}, 0
		local user = users[i]
		local items = user.inventory and json.decode(user.inventory) or {}
		local accounts = user.money and json.decode(user.money) or {}

		for k, v in pairs(accounts) do
			if type(v) == 'table' then break end
			if k == 'cash' then k = 'money' end

			if server.accounts[k] and Items(k) and v > 0 then
				slot += 1
				inventory[slot] = {slot=slot, name=k, count=v}
			end
		end

		local shouldConvert = false

		for _, v in pairs(items) do
			if Items(v?.name) then
				slot += 1
				inventory[slot] = {slot=slot, name=v.name, count=v.amount, metadata = type(v.info) == 'table' and v.info or {}}
				if v.type == "weapon" then
					inventory[slot].metadata.durability = v.info.quality or 100
					inventory[slot].metadata.ammo = v.info.ammo or 0
					inventory[slot].metadata.components = {}
					inventory[slot].metadata.serial = v.info.serie or GenerateSerial()
					inventory[slot].metadata.quality = nil
				end
			end

			shouldConvert = v.amount and true
		end

		if shouldConvert then
			count += 1
			parameters[count] = { 'UPDATE players SET inventory = ? WHERE citizenid = ?', { json.encode(inventory), user.citizenid } }
		end
	end

	if count > 0 then
	    Print(('Converting %s user inventories to new data format'):format(count))

		if not MySQL.transaction.await(parameters) then
			return Print('An error occurred while converting player inventories')
		end
		Wait(100)
	else
        print('literally why are you even running the convert command if you have no inventory data to convert? real 500iq move there')
    end

	local plates = MySQL.query.await('SELECT plate, citizenid FROM player_vehicles')

	if plates then
		for i = 1, #plates do
			plates[plates[i].plate] = plates[i].citizenid
		end

		local oldqbdumpsterfire, trunk = pcall(MySQL.query.await, 'SELECT plate, items FROM trunkitems')

		if oldqbdumpsterfire and trunk then
			table.wipe(parameters)
			count = 0
			local vehicles = {}

			for _, v in pairs(trunk) do
				local owner = plates[v.plate]

				if owner then
					if not vehicles[owner] then
						vehicles[owner] = {}
					end

					if not vehicles[owner][v.plate] then
						local items = json.decode(v.items) or {}
						local inventory, slot = {}, 0

						for _, v in pairs(items) do
							if Items(v?.name) then
								slot += 1
								inventory[slot] = {slot=slot, name=v.name, count=v.amount, metadata = type(v.info) == 'table' and v.info or {}}
								if v.type == "weapon" then
									inventory[slot].metadata.durability = v.info.quality or 100
									inventory[slot].metadata.ammo = v.info.ammo or 0
									inventory[slot].metadata.components = {}
									inventory[slot].metadata.serial = v.info.serie or GenerateSerial()
									inventory[slot].metadata.quality = nil
								end
							end
						end

						vehicles[owner][v.plate] = true
						count += 1
						parameters[count] = { 'UPDATE player_vehicles SET trunk = ? WHERE plate = ? AND citizenid = ?', { json.encode(inventory), v.plate, owner } }
					end
				end
			end

			Print(('Moving ^3%s^0 trunks to the player_vehicles table'):format(count))

			if count > 0 then
				if not MySQL.transaction.await(parameters) then
					return Print('An error occurred while converting trunk inventories')
				end

				Wait(100)
			end
		end

		local glovebox = oldqbdumpsterfire and MySQL.query.await('SELECT plate, items FROM gloveboxitems')

		if glovebox then
			table.wipe(parameters)
			count = 0
			local vehicles = {}

			for _, v in pairs(glovebox) do
				local owner = plates[v.plate]

				if owner then
					if not vehicles[owner] then
						vehicles[owner] = {}
					end

					if not vehicles[owner][v.plate] then
						local items = json.decode(v.items) or {}
						local inventory, slot = {}, 0

						for _, v in pairs(items) do
							if Items(v?.name) then
								slot += 1
								inventory[slot] = {slot=slot, name=v.name, count=v.amount, metadata = type(v.info) == 'table' and v.info or {}}

								if v.type == "weapon" then
									inventory[slot].metadata.durability = v.info.quality or 100
									inventory[slot].metadata.ammo = v.info.ammo or 0
									inventory[slot].metadata.components = {}
									inventory[slot].metadata.serial = v.info.serie or GenerateSerial()
									inventory[slot].metadata.quality = nil
								end
							end
						end

						vehicles[owner][v.plate] = true
						count += 1
						parameters[count] = { 'UPDATE player_vehicles SET glovebox = ? WHERE plate = ? AND citizenid = ?', { json.encode(inventory), v.plate, owner } }
					end
				end
			end

			Print(('Moving ^3%s^0 gloveboxes to the player_vehicles table'):format(count))

			if count > 0 then
				if not MySQL.transaction.await(parameters) then
					return Print('An error occurred while converting glovebox inventories')
				end
			end
		end
	end

	local qbEvidence = MySQL.query.await("SELECT stash, items FROM stashitems WHERE stash LIKE '% | Drawer%'")

	if qbEvidence then
		table.wipe(parameters)
		count = 0

		---@type table<string, OxItem[]> maps stash name to inventory
		local oxEvidence = {}

		for i = 1, #qbEvidence do
			local qbStash = qbEvidence[i]
			local name = 'evidence-'..qbStash.stash:sub(12)
			local items = server.convertInventory(nil, (qbStash.items and json.decode(qbStash.items) or {}))

			--- evidence numbers can be shared between locations, so need to maintain map and merge.
			if oxEvidence[name] then
				for k = 1, #items do
					oxEvidence[name][#oxEvidence[name]+1] = items[k]
				end
			else
				oxEvidence[name] = items
			end
		end

		for name, items in pairs(oxEvidence) do
			count += 1
			parameters[count] = { "INSERT INTO ox_inventory (owner, name, data) VALUES ('', ?, ?) ON DUPLICATE KEY UPDATE name = VALUES(name), data = VALUES(data)", {
				name, json.encode(items)
			}}
		end

		Print(('Creating ^3%s^0 evidence lockers from ^3%s^0 lockers by merging duplicate locker numbers'):format(count, #qbEvidence))
		if count > 0 then
			if not MySQL.transaction.await(parameters) then
				return Print('An error occurred while converting evidence lockers')
			end
		end
	end

	Print('Successfully converted user and vehicle inventories')
	started = false
end
```
---

### 2. **ox_inventory/modules/item/server.lua**
```
include QBCore item handling. Insert the provided code snippet between lines 116 and 217.
```
#### Code Snippet:
```lua
elseif shared.framework == 'qb' then
  local QBCore = exports['qb-core']:GetCoreObject()
  local items = QBCore.Shared.Items

  if items and table.type(items) ~= 'empty' then
    local dump = {}
    local count = 0
    local ignoreList = {
      "weapon_",
      "pistol_",
      "pistol50_",
      "revolver_",
      "smg_",
      "combatpdw_",
      "shotgun_",
      "rifle_",
      "carbine_",
      "gusenberg_",
      "sniper_",
      "snipermax_",
      "tint_",
      "_ammo"
    }

    local function checkIgnoredNames(name)
      for i = 1, #ignoreList do
        if string.find(name, ignoreList[i]) then
          return true
        end
      end
      return false
    end

    for k, item in pairs(items) do
      -- Explain why this wouldn't be table to me, because numerous people have been getting "attempted to index number" here
      if type(item) == 'table' then
        -- Some people don't assign the name property, but it seemingly always matches the index anyway.
        if not item.name then item.name = k end

        if not ItemList[item.name] and not checkIgnoredNames(item.name) then
          item.close = item.shouldClose == nil and true or item.shouldClose
          item.stack = not item.unique and true
          item.description = item.description
          item.weight = item.weight or 0
          dump[k] = item
          count += 1
        end
      end
    end

    if table.type(dump) ~= 'empty' then
      local file = {string.strtrim(LoadResourceFile(shared.resource, 'data/items.lua'))}
      file[1] = file[1]:gsub('}$', '')

      ---@todo separate into functions for reusability, properly handle nil values
      local itemFormat = [[

[%q] = {
  label = %q,
  weight = %s,
  stack = %s,
  close = %s,
  description = %q,
  client = {
    status = {
      hunger = %s,
      thirst = %s,
      stress = %s
    },
    image = %q,
  }
},
]]

      local fileSize = #file

      for _, item in pairs(dump) do
        if not ItemList[item.name] then
          fileSize += 1

          ---@todo cry
          local itemStr = itemFormat:format(item.name, item.label, item.weight, item.stack, item.close, item.description or 'nil', item.hunger or 'nil', item.thirst or 'nil', item.stress or 'nil', item.image or 'nil')
          -- temporary solution for nil values
          itemStr = itemStr:gsub('[%s]-[%w]+ = "?nil"?,?', '')
          -- temporary solution for empty status table
          itemStr = itemStr:gsub('[%s]-[%w]+ = %{[%s]+%},?', '')
          -- temporary solution for empty client table
          itemStr = itemStr:gsub('[%s]-[%w]+ = %{[%s]+%},?', '')
          file[fileSize] = itemStr
          ItemList[item.name] = item
        end
      end

      file[fileSize+1] = '}'

      SaveResourceFile(shared.resource, 'data/items.lua', table.concat(file), -1)
      shared.info(count, 'items have been copied from the QBCore.Shared.Items.')
      shared.info('You should restart the resource to load the new items.')
    end
  end
		Wait(500)
```

---

### 3. **ox_inventory/modules/mysql/server.lua**
```
define table and column names for QBCore framework. Add the following lines between line 30 and 34.
```
#### Code Snippet:
```lua
    elseif shared.framework == 'qb' then
        playerTable = 'players'
        playerColumn = 'citizenid'
        vehicleTable = 'player_vehicles'
        vehicleColumn = 'plate'
```

---

### 4. Add QBCore Folder in `ox_inventory/bridge`
Create a new folder named `qb` inside `ox_inventory/bridge` and add relevant shared files for QBCore integration.

---
