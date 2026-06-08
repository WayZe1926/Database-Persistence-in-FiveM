# Database & Persistence in FiveM

This is one of those topics that doesn’t look important at first… until your server starts growing.

When you're building a FiveM server, you quickly realize something:

> Everything depends on the database.

Money, inventories, vehicles, characters, jobs, housing, gangs, progression… all of it only exists because the database remembers it when the server doesn’t.

And if it’s done wrong, everything eventually breaks in ways that are painful to debug.

---

## Why Persistence Actually Matters

Persistence is not just "saving data".

It’s what makes your server feel like a real world.

Without it:

- Players lose progress every restart
- Vehicles disappear
- Items reset
- Money wipes
- Entire economies collapse

With it:

- Players build long-term progress
- Servers feel alive
- Decisions matter
- Time investment is respected

The database is what turns a temporary session into a persistent world.

---

## Loading Player Data

Every player joins → load their data → store it in memory → sync back when needed.

```
-- Load player from database
function LoadPlayer(identifier, source)
    MySQL.query('SELECT * FROM users WHERE identifier = ?', {identifier}, function(result)
        if result[1] then
            PlayerData[source] = result[1]
        else
            MySQL.insert('INSERT INTO users (identifier, money) VALUES (?, ?)', {identifier, 500}, function(id)
                PlayerData[source] = {
                    id = id,
                    identifier = identifier,
                    money = 500,
                    inventory = {},
                    position = {}
                }
            end)
        end
    end)
end
```
Notice how we check if the player exists first, and insert if not. This prevents errors when a new player joins.

# Saving Data Safely
We never want to just overwrite everything blindly. Always save what matters, preferably on disconnect or key events:
```
function SavePlayer(source)
    local player = PlayerData[source]
    if not player then return end

    MySQL.update('UPDATE users SET money = ?, inventory = ?, position = ? WHERE identifier = ?', {
        player.money,
        json.encode(player.inventory),
        json.encode(player.position),
        player.identifier
    })
end
```
Saving everything constantly is tempting but kills performance on large servers.

# Transactions Prevent Partial Updates
A common problem: player sells an item and gets money. Server crashes mid-update. Dupes or loss occur.

Solution: transaction-like logic.
```
-- Simulate atomic transaction
Transaction({
    { query = 'UPDATE users SET money = money - ? WHERE identifier = ?', values = {amount, sender.identifier} },
    { query = 'UPDATE users SET money = money + ? WHERE identifier = ?', values = {amount, receiver.identifier} }
}, function(success)
    if success then
        sender.money = sender.money - amount
        receiver.money = receiver.money + amount
        print("Transaction completed safely")
    else
        print("Transaction failed, rolled back")
    end
end)
```
Either everything succeeds, or nothing does. This prevents half-applied updates, which are the main cause of duplication and desync.
# Avoiding Duplication Bugs
Most dupes are not inventory bugs, they’re timing bugs.

# Example scenario:

- Player sells item
- Server starts DB update
- Player disconnects instantly
- Inventory rollback triggers
- DB update still completes

# Result: item returned + money given → dupe.

# The solution:

- Server validates every action
- Updates memory first
- Writes to DB in controlled events
- Uses transactions for multi-step operations
# Logging Everything
When issues arise, logs are your best friend:
```
function LogAction(identifier, action, value)
    print(("[LOG] %s -> %s : %s"):format(identifier, action, tostring(value)))

    MySQL.update('INSERT INTO logs (identifier, action, value) VALUES (?, ?, ?)', {
        identifier,
        action,
        json.encode(value)
    })
end
```
Logs help track:

- Money changes
- Inventory transactions
- Vehicle ownership
- Admin actions

Without logs, you’re guessing in the dark when problems happen.

# Scaling Changes Everything
A system that works for 10 players may completely fail at 200+.

Why?

- More simultaneous DB writes
- More overlapping events
- More race conditions
- More edge cases

This is where weak persistence systems collapse.

# Final Thoughts
Persistence isn’t just saving player stats.

It’s what makes the world real.

It’s the difference between:

- A server where progress is respected
- A server where everything can vanish or duplicate

When designing your server, think like this:

“Can my system survive hundreds of players spamming actions, without corrupting data?”

If the answer is yes, you’re on the right path.
