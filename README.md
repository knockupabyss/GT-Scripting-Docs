# GT-Scripting-Docs

----------

# GT_CustomMapScripting

Scripting for compatible Gorilla Tag maps is done using a [Luau](https://luau.org/) script, executed by each client. Maps can have a single script that overrides the default game modes (e.g., infection, casual). This script manages custom game logic, including interactions such as opening/closing doors, tagging, and movement.

----------

## **Setup**

Scripting is done in the **Luau** programming language and starts with a simple text file. We recommend using **VS Code** with the extensions:

-   `stylua` – for code formatting.
-   `luau-lsp` – a language server providing real-time feedback.

Type files are provided to enhance these extensions with up-to-date bindings between Gorilla Tag and Luau.

### **Steps to Add Scripts to Maps**:

1.  Once your script is complete, add it to your map's Unity project as a **Text Resource**.
2.  Supply the text file in the **Custom Game Mode** field of the Map Descriptor (where you define the map's name, etc.).
3.  After exporting and uploading your map, run your script by selecting **CUSTOM** in the game mode selector inside the Virtual Stump. Then load up your map.
    
    > **Note:** If CUSTOM is not selected, the script won't execute, and the default game mode will run.
    

----------

## **Important Notes**

-   **Client-Side Execution**:  
    All scripts currently run on the client side. You are responsible for managing synchronization between clients. For instance:
    
    -   When opening a door, ensure you network the state to other clients.
    -   If a player gets tagged, sync the change and update their materials accordingly.
-   **Printing/Logging**:  
    Logging is done via Unity's log system.
    
    -   On **PC VR**, logs can be accessed in the Unity Player logs.
    -   On **Quest**, use `logcat` via `adb` to retrieve logs.

----------

## **Debugging**

A built-in `print()` function allows you to log messages to Unity logs. 

In the event that your script crashes or fails to run, the logs will include the error message and the line number where the issue occurred.

----------

## **Networking**

Networking is currently event-driven. Each client can send up to **20 events per second**.

-   **Sending Events**:  
    Use the `emitEvent()` function, which takes:
    
    -   `tag` (string): The event name.
    -   `data` (table): A table containing the data to be sent (must have at least one entry).
    
    ```lua
    emitEvent("eventName", { key = "value" })
    ```
    
    **Limitations**:
    
    -   Tables are limited to **10 entries**.
    -   Every event must send some data, even if it's just an empty table with one element.
    -   Events can only send, `Vec3`'s, `Quat`'s, `Player`'s, and Numbers.
-   **Receiving Events**:  
    Implement an `onEvent()` function in your script to handle incoming events. The function should accept:
    
    -   `tag` (string): The event name.
    -   `data` (table): The data received.
    
    ```lua
    function onEvent(tag, data)
        if tag == "exampleEvent" then
            -- Process the data
        end
    end
    ```

----------

## **Players**

Scripts have simple access to each player in the lobby, allowing you to do things like:

-   Check if a player is the **master client**.
-   Get their **position**, **rotation**, and **player materials**.

### **Player Materials**

You can control how players appear using the `playerMaterial` property. This property is an integer corresponding to different material types (normal, tagged, infected, frozen, etc.):

Example:
```lua
localPlayer.playerMaterial = 0
```
----------

## **Touch Events**

When a player tags/touches another player, the `onEvent()` function receives an event named `"touchedPlayer"` with the touched player instance as data.


----------

## **GameObjects**

Scripts can access and manipulate **GameObjects** within a custom map's scene. Available properties include:

-   **position**: `Vec3`
-   **rotation**: `Quat`
-   **scale**: `Vec3`

Additional methods allow you to toggle:

-   **Collision**: `setCollision(enabled: boolean)`
-   **Visibility**: `setVisibility(enabled: boolean)`

> **Note:** Static objects cannot be moved, as they are combined into a single mesh upon export.

### **Finding GameObjects**

Use the `findGameObject()` function to retrieve GameObjects by name:

```lua
local door = findGameObject("DoorName")
door:setVisibility(false)
```

Fetching GameObjects can be slow, so store references when possible.

----------

## **Sounds**

You can play sounds using the `playSound()` function, which accepts:

-   `idx` (number): The sound ID (similar to player materials).
-   `pos` (Vec3): The position to play the sound.
-   `vol` (number): Volume percentage (0 to 1).

----------

## **Vibrations**

The `startVibration()` function lets you trigger controller vibrations. Parameters include:

-   `leftHand` (boolean): Whether to vibrate the left hand.
-   `strength` (number): Vibration strength (0 to 1).
-   `duration` (number): Duration in seconds.

```lua
startVibration(true, 0.8, 2.0)
```

----------

## **Movement Control**

Scripts can modify player movement settings through the `playerSettings` instance. 

Example usage:

```lua
playerSettings.maxJumpSpeed = 10
playerSettings.jumpMultiplier = 1.5
```
----------


## **Tick Function**

The `tick()` function is called every frame, making it essential for continuous processing. Use this function to handle tasks like:

-   Updating game logic for non event based player interactions.
-   Handling movement over time.

Since `tick()` is called frequently, it's a good practice to prefetch any GameObjects outside the function to avoid performance issues.

Example usage:

```lua
local buttonObject = findGameObject("Button")
local platform= findGameObject("Platform")

function isButtonPressed(gameObject)
    local leftHand = LocalPlayer.leftHandPosition
    local rightHand = LocalPlayer.rightHandPosition

    local minBounds = gameObject.position - (gameObject.scale * 0.5)
    local maxBounds = gameObject.position + (gameObject.scale * 0.5)

    local function isWithinBounds(hand)
        return hand.x >= minBounds.x and hand.x <= maxBounds.x and
               hand.y >= minBounds.y and hand.y <= maxBounds.y and
               hand.z >= minBounds.z and hand.z <= maxBounds.z
    end

    return isWithinBounds(leftHand) or isWithinBounds(rightHand)
end

function tick()
    if isButtonPressed(buttonObject) then
        platform.position = platform.position + Vec3.new(0, 0.1, 0)
    end
end

```

----------

# Simple Tag Example
```lua
timer = os.clock()
tagCoolDown = 0
lastPlayerCount = 0

function allNotTagged()
    for index, ply in ipairs(Players) do
        if ply.PlayerMaterial ~= 0 then
            return false
        end
    end
    return true
end

function tick(dt)
    if InRoom then
        if os.clock() - timer >= 2 then
            if #Players == 1 then
                LocalPlayer.playerMaterial = 1
            else
                emitEvent("whoTagged", { 1 })
            end
        end
        if tagCoolDown ~= 0 and os.clock() - tagCoolDown >= 2 then
            if LocalPlayer.playerMaterial == 2 then
                LocalPlayer.playerMaterial = 1
            end
            tagCoolDown = 0
        end
    end

    if LocalPlayer.isMasterClient and lastPlayerCount ~= #Players then
        if allNotTagged() then
            local n = math.random(1, #Players)
            emitEvent("tagEvent", { Players[n], LocalPlayer })
            onEvent("tagEvent", { Players[n], LocalPlayer })
        end
    end

    lastPlayerCount = #Players
end

function onEvent(event, data)
    if event == "touchedPlayer" then
        if LocalPlayer.playerMaterial == 1 then
            emitEvent("tagEvent", { data, LocalPlayer })
            onEvent("tagEvent", { data, LocalPlayer })
        end
    end

    if event == "tagEvent" then
        data[1].playerMaterial = 1
        if data[2] ~= nil then
            data[2].playerMaterial = 0
        end
        if data[1] == LocalPlayer then
            tagCoolDown = os.clock()
            data[1].playerMaterial = 2
        end
    end

    if event == "imTagged" then
        data[1].playerMaterial = 1
    end

    if event == "whoTagged" then
        if LocalPlayer.playerMaterial == 1 then
            emitEvent("imTagged", { LocalPlayer })
        end
    end
end

```
