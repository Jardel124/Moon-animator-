--new
print("load")

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")
local HttpService = game:GetService("HttpService")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")
local mouse = player:GetMouse()

-- System variables
local selectedRig = nil
local selectedPart = nil
local savedPositions = {}
local bodyPartPositions = {}
local currentKeyframe = 0
local isPlaying = false
local isLooping = false
local loopConnection = nil
local startFromKeyframe = 1
local highlight = nil
local previewConnections = {}
local rigCheckConnection = nil
local keyframeDuration = 0.5
local editingKeyframe = nil
local exportedScripts = {}
local currentMode = "rig"
local currentRigType = "static"
local currentPlayerType = "static"

-- Persistent data key
local DATA_KEY = "MoonAnimatorX_SaveData_" .. tostring(player.UserId)

-- Theme colors
local THEME = {
    ORANGE = Color3.fromRGB(255, 140, 0),
    BLACK = Color3.fromRGB(0, 0, 0),
    WHITE = Color3.fromRGB(255, 255, 255),
    GRAY = Color3.fromRGB(128, 128, 128),
    DARK_GRAY = Color3.fromRGB(64, 64, 64),
    LIGHT_GRAY = Color3.fromRGB(192, 192, 192),
    GREEN = Color3.fromRGB(46, 204, 113),
    BLUE = Color3.fromRGB(52, 152, 219),
    RED = Color3.fromRGB(231, 76, 60),
    PURPLE = Color3.fromRGB(155, 89, 182)
}

-- Save data to WritableComputerScript or clipboard as fallback
local function saveDataPersistent()
    local dataToSave = {
        exportedScripts = exportedScripts,
        savedPositions = savedPositions,
        bodyPartPositions = bodyPartPositions,
        settings = {
            keyframeDuration = keyframeDuration,
            currentMode = currentMode,
            currentRigType = currentRigType,
            currentPlayerType = currentPlayerType
        },
        timestamp = tick()
    }
    
    local jsonData = HttpService:JSONEncode(dataToSave)
    
    -- Try to save using writefile (if supported)
    local success, err = pcall(function()
        if writefile then
            writefile(DATA_KEY .. ".json", jsonData)
            return true
        end
        return false
    end)
    
    if not success then
        -- Fallback: save to clipboard
        if setclipboard then
            setclipboard("-- MOON ANIMATOR X SAVE DATA --\n-- Copy this and save it somewhere safe!\nlocal saveData = [[\n" .. jsonData .. "\n]]\nreturn saveData")
            print("Data saved to clipboard as backup!")
        else
            print("-- MOON ANIMATOR X SAVE DATA --")
            print("local saveData = [[")
            print(jsonData)
            print("]]")
            print("return saveData")
        end
    end
    
    return success
end

-- Load data from file or return empty data
local function loadDataPersistent()
    local success, jsonData = pcall(function()
        if readfile and isfile and isfile(DATA_KEY .. ".json") then
            return readfile(DATA_KEY .. ".json")
        end
        return nil
    end)
    
    if success and jsonData then
        local loadSuccess, data = pcall(function()
            return HttpService:JSONDecode(jsonData)
        end)
        
        if loadSuccess and data then
            exportedScripts = data.exportedScripts or {}
            savedPositions = data.savedPositions or {}
            bodyPartPositions = data.bodyPartPositions or {}
            
            if data.settings then
                keyframeDuration = data.settings.keyframeDuration or 0.5
                currentMode = data.settings.currentMode or "rig"
                currentRigType = data.settings.currentRigType or "static"
                currentPlayerType = data.settings.currentPlayerType or "static"
            end
            
            print("Moon Animator X: Data loaded successfully!")
            return true
        end
    end
    
    print("Moon Animator X: No save data found, starting fresh.")
    return false
end

-- Detect rig type
local function detectRigType(rig)
    if not rig or not rig:FindFirstChild("Humanoid") then 
        return nil 
    end
    
    if rig:FindFirstChild("UpperTorso") and rig:FindFirstChild("LowerTorso") then
        return "R15"
    elseif rig:FindFirstChild("Torso") then
        return "R6"
    end
    
    return nil
end

-- Check if character/rig is moving
local function isCharacterMoving(character)
    local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
    local humanoid = character:FindFirstChild("Humanoid")
    
    if humanoidRootPart and humanoid then
        -- Check velocity and humanoid movement
        local velocity = humanoidRootPart.AssemblyLinearVelocity.Magnitude
        local moveDirection = humanoid.MoveDirection.Magnitude
        local isAnchored = humanoidRootPart.Anchored
        
        return (velocity > 2 or moveDirection > 0) and not isAnchored
    end
    return false
end

-- Generate export scripts based on mode, type, and loop setting
local function generateExportScript(mode, scriptType, infiniteLoop)
    local baseCode = ""
    
    if mode == "rig" then
        if scriptType == "static" then
            baseCode = [[-- RIG STATIC ANIMATION SCRIPT
-- This script works when the rig has very low velocity (almost stopped)
local TweenService = game:GetService('TweenService')
local RunService = game:GetService("RunService")
local rig = script.Parent

local function isRigStatic()
    local rootPart = rig:FindFirstChild("HumanoidRootPart")
    local humanoid = rig:FindFirstChild("Humanoid")
    
    if not rootPart or not humanoid then return false end
    
    local velocity = rootPart.AssemblyLinearVelocity.Magnitude
    local moveDirection = humanoid.MoveDirection.Magnitude
    
    -- Static means very low velocity (almost stopped)
    return velocity < 0.5 and moveDirection < 0.1
end

local function playStaticAnimation()
    if isRigStatic() then
        local rootPart = rig:FindFirstChild("HumanoidRootPart")
        if not rootPart then return end
        
]]
        else -- walking
            baseCode = [[-- RIG WALKING ANIMATION SCRIPT  
-- This script works when the rig has high velocity (moving)
local TweenService = game:GetService('TweenService')
local RunService = game:GetService("RunService")
local rig = script.Parent

local function isRigMoving()
    local rootPart = rig:FindFirstChild("HumanoidRootPart")
    local humanoid = rig:FindFirstChild("Humanoid")
    
    if not rootPart or not humanoid then return false end
    
    local velocity = rootPart.AssemblyLinearVelocity.Magnitude
    local moveDirection = humanoid.MoveDirection.Magnitude
    
    -- Moving means higher velocity
    return velocity > 0.5 or moveDirection > 0.1
end

local function playWalkingAnimation()
    if isRigMoving() then
        local rootPart = rig:FindFirstChild("HumanoidRootPart")
        if not rootPart then return end
        
]]
        end
    else -- player mode
        if scriptType == "static" then
            baseCode = [[-- PLAYER STATIC ANIMATION SCRIPT
-- This script works when the player has very low velocity (almost stopped)
local Players = game:GetService("Players")
local TweenService = game:GetService('TweenService')
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local rootPart = character:WaitForChild("HumanoidRootPart")

local function isPlayerStatic()
    local velocity = rootPart.AssemblyLinearVelocity.Magnitude
    local moveDirection = humanoid.MoveDirection.Magnitude
    
    return velocity < 0.5 and moveDirection < 0.1
end

local connection
connection = RunService.Heartbeat:Connect(function()
    if isPlayerStatic() then
        connection:Disconnect()
        
]]
        else -- walking
            baseCode = [[-- PLAYER WALKING ANIMATION SCRIPT
-- This script works when the player has high velocity (moving)
local Players = game:GetService("Players")
local TweenService = game:GetService('TweenService')
local RunService = game:GetService("RunService")

local player = Players.LocalPlayer  
local character = player.Character or player.CharacterAdded:Wait()
local humanoid = character:WaitForChild("Humanoid")
local rootPart = character:WaitForChild("HumanoidRootPart")

local function isPlayerMoving()
    local velocity = rootPart.AssemblyLinearVelocity.Magnitude
    local moveDirection = humanoid.MoveDirection.Magnitude
    
    return velocity > 0.5 or moveDirection > 0.1
end

local connection
connection = RunService.Heartbeat:Connect(function()
    if isPlayerMoving() then
        connection:Disconnect()
        
]]
        end
    end
    
    -- Add keyframes data
    if #savedPositions > 0 or next(bodyPartPositions) then
        -- Add keyframes with relative positioning
        for i, savedData in ipairs(savedPositions) do
            baseCode = baseCode .. "        -- Keyframe " .. i .. "\n"
            baseCode = baseCode .. "        local tweenInfo" .. i .. " = TweenInfo.new(" .. (savedData.duration or keyframeDuration) .. ", Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)\n"
            baseCode = baseCode .. "        local tweens" .. i .. " = {}\n"
            
            local partIndex = 1
            for partName, data in pairs(savedData.parts) do
                local x, y, z = data.CFrame.Position.X, data.CFrame.Position.Y, data.CFrame.Position.Z
                local rx, ry, rz = data.CFrame:toEulerAnglesXYZ()
                
                baseCode = baseCode .. "        local part" .. partIndex .. " = " .. (mode == "rig" and "rig" or "character") .. ":FindFirstChild(\"" .. partName .. "\")\n"
                baseCode = baseCode .. "        if part" .. partIndex .. " then\n"
                baseCode = baseCode .. "            local targetCFrame" .. partIndex .. " = rootPart.CFrame * CFrame.new(" .. string.format("%.3f, %.3f, %.3f", x, y, z) .. ") * CFrame.Angles(" .. string.format("%.3f, %.3f, %.3f", rx, ry, rz) .. ")\n"
                baseCode = baseCode .. "            local tween" .. partIndex .. " = TweenService:Create(part" .. partIndex .. ", tweenInfo" .. i .. ", {\n"
                baseCode = baseCode .. "                CFrame = targetCFrame" .. partIndex .. ",\n"
                baseCode = baseCode .. string.format("                Size = Vector3.new(%.3f, %.3f, %.3f)\n", data.Size.X, data.Size.Y, data.Size.Z)
                baseCode = baseCode .. "            })\n"
                baseCode = baseCode .. "            table.insert(tweens" .. i .. ", tween" .. partIndex .. ")\n"
                baseCode = baseCode .. "        end\n"
                
                partIndex = partIndex + 1
            end
            
            baseCode = baseCode .. "        \n"
            baseCode = baseCode .. "        for _, tween in pairs(tweens" .. i .. ") do\n"
            baseCode = baseCode .. "            tween:Play()\n"
            baseCode = baseCode .. "        end\n"
            baseCode = baseCode .. "        if #tweens" .. i .. " > 0 then\n"
            baseCode = baseCode .. "            tweens" .. i .. "[1].Completed:Wait()\n"
            baseCode = baseCode .. "        end\n\n"
        end
    else
        baseCode = baseCode .. "        -- No animation data available\n"
        baseCode = baseCode .. "        print('No keyframes saved for animation')\n"
    end
    
    -- Close the conditional blocks and add infinite loop option
    if mode == "rig" then
        if scriptType == "static" then
            baseCode = baseCode .. "    end\nend\n\n"
            if infiniteLoop then
                baseCode = baseCode .. "-- Infinite loop\nwhile true do\n    playStaticAnimation()\n    wait(0.5)\nend"
            else
                baseCode = baseCode .. "-- Single execution\nplayStaticAnimation()"
            end
        else
            baseCode = baseCode .. "    end\nend\n\n"
            if infiniteLoop then
                baseCode = baseCode .. "-- Infinite loop\nwhile true do\n    playWalkingAnimation()\n    wait(0.5)\nend"
            else
                baseCode = baseCode .. "-- Single execution\nplayWalkingAnimation()"
            end
        end
    else
        baseCode = baseCode .. "    end\nend)"
        if infiniteLoop then
            baseCode = baseCode .. "\n\n-- Auto-restart for infinite loop\nlocal function startLoop()\n    spawn(function()\n        wait(1)\n        if " .. (scriptType == "static" and "isPlayerStatic()" or "isPlayerMoving()") .. " then\n            startLoop()\n        end\n    end)\nend\n\nstartLoop()"
        end
    end
    
    return baseCode
end

-- Save exported script
local function saveExportedScript(name, code, mode, scriptType, infiniteLoop)
    local scriptData = {
        name = name,
        code = code,
        mode = mode,
        scriptType = scriptType,
        infiniteLoop = infiniteLoop or false,
        timestamp = tick(),
        keyframeCount = #savedPositions,
        partAnimations = 0
    }
    
    for _, _ in pairs(bodyPartPositions) do
        scriptData.partAnimations = scriptData.partAnimations + 1
    end
    
    table.insert(exportedScripts, scriptData)
    saveDataPersistent() -- Save after adding new script
end

-- Load exported script  
local function loadExportedScript(scriptData)
    currentMode = scriptData.mode
    currentRigType = scriptData.mode == "rig" and scriptData.scriptType or currentRigType
    currentPlayerType = scriptData.mode == "player" and scriptData.scriptType or currentPlayerType
    
    print("Loaded script: " .. scriptData.name)
    print("Mode: " .. scriptData.mode .. ", Type: " .. scriptData.scriptType)
    saveDataPersistent() -- Save settings
end

-- Find rig from part
local function findRigFromPart(part)
    local current = part
    
    if current:FindFirstChild("Humanoid") then
        return current
    end
    
    while current.Parent do
        current = current.Parent
        
        if current:FindFirstChild("Humanoid") then
            return current
        end
        
        if current == workspace then
            break
        end
    end
    
    return nil
end

-- Create highlight
local function createHighlight(part)
    if highlight then
        highlight:Destroy()
    end
    
    highlight = Instance.new("Highlight")
    highlight.Parent = part
    highlight.FillColor = THEME.ORANGE
    highlight.OutlineColor = THEME.WHITE
    highlight.FillTransparency = 1
    highlight.OutlineTransparency = 0
end

-- Remove highlight
local function removeHighlight()
    if highlight then
        highlight:Destroy()
        highlight = nil
    end
end

-- Clear preview
local function clearKeyframePreview()
    for _, connection in pairs(previewConnections) do
        if connection.part then
            connection.part:Destroy()
        end
        if connection.highlight then
            connection.highlight:Destroy()
        end
    end
    previewConnections = {}
end

-- Get part display name
local function getPartType(partName)
    local partTypes = {
        ["Head"] = "Head",
        ["Torso"] = "Torso", 
        ["HumanoidRootPart"] = "Root",
        ["Left Arm"] = "L.Arm",
        ["Right Arm"] = "R.Arm", 
        ["Left Leg"] = "L.Leg",
        ["Right Leg"] = "R.Leg",
        ["LeftUpperArm"] = "L.U.Arm",
        ["RightUpperArm"] = "R.U.Arm",
        ["LeftLowerArm"] = "L.L.Arm", 
        ["RightLowerArm"] = "R.L.Arm",
        ["LeftUpperLeg"] = "L.U.Leg",
        ["RightUpperLeg"] = "R.U.Leg",
        ["LeftLowerLeg"] = "L.L.Leg",
        ["RightLowerLeg"] = "R.L.Leg",
        ["LeftHand"] = "L.Hand",
        ["RightHand"] = "R.Hand",
        ["LeftFoot"] = "L.Foot",
        ["RightFoot"] = "R.Foot",
        ["UpperTorso"] = "U.Torso",
        ["LowerTorso"] = "L.Torso"
    }
    
    return partTypes[partName] or string.sub(partName, 1, 6)
end

-- Save position with relative positioning
local function savePosition(rig, specificPart, updateExisting)
    if not rig or not rig:FindFirstChild("Humanoid") then 
        return 
    end
    
    local rootPart = rig:FindFirstChild("HumanoidRootPart")
    if not rootPart then
        print("Warning: No HumanoidRootPart found for relative positioning")
        return
    end
    
    if specificPart then
        if not bodyPartPositions[specificPart.Name] then
            bodyPartPositions[specificPart.Name] = {}
        end
        
        local relativeCFrame = rootPart.CFrame:Inverse() * specificPart.CFrame
        
        local newData = {
            CFrame = relativeCFrame,
            Size = specificPart.Size,
            duration = keyframeDuration,
            timestamp = tick(),
            position = specificPart.Position
        }
        
        if updateExisting and editingKeyframe and editingKeyframe.type == "part" and editingKeyframe.partName == specificPart.Name then
            bodyPartPositions[specificPart.Name][editingKeyframe.index] = newData
            local index = editingKeyframe.index
            editingKeyframe = nil
            saveDataPersistent() -- Auto-save
            return index, specificPart.Name, true
        else
            table.insert(bodyPartPositions[specificPart.Name], newData)
            saveDataPersistent() -- Auto-save
            return #bodyPartPositions[specificPart.Name], specificPart.Name, false
        end
    else
        local savedData = {
            rigName = rig.Name,
            duration = keyframeDuration,
            parts = {},
            timestamp = tick(),
            rootCFrame = rootPart.CFrame
        }
        
        for _, part in pairs(rig:GetChildren()) do
            if part:IsA("BasePart") and part ~= rootPart then
                local relativeCFrame = rootPart.CFrame:Inverse() * part.CFrame
                savedData.parts[part.Name] = {
                    CFrame = relativeCFrame,
                    Size = part.Size,
                    position = part.Position
                }
            elseif part == rootPart then
                savedData.parts[part.Name] = {
                    CFrame = CFrame.new(0, 0, 0),
                    Size = part.Size,
                    position = part.Position
                }
            end
        end
        
        local function savePartsRecursively(parent)
            for _, child in pairs(parent:GetChildren()) do
                if child:IsA("BasePart") and not savedData.parts[child.Name] and child ~= rootPart then
                    local relativeCFrame = rootPart.CFrame:Inverse() * child.CFrame
                    savedData.parts[child.Name] = {
                        CFrame = relativeCFrame,
                        Size = child.Size,
                        position = child.Position
                    }
                elseif child:IsA("Model") then
                    savePartsRecursively(child)
                end
            end
        end
        
        savePartsRecursively(rig)
        
        if updateExisting and editingKeyframe and editingKeyframe.type == "fullbody" then
            savedPositions[editingKeyframe.index] = savedData
            local index = editingKeyframe.index
            editingKeyframe = nil
            saveDataPersistent() -- Auto-save
            return index, "Full Body", true
        else
            table.insert(savedPositions, savedData)
            currentKeyframe = #savedPositions
            saveDataPersistent() -- Auto-save
            return currentKeyframe, "Full Body", false
        end
    end
end

-- Update body parts list
local function updateBodyPartsList(bodyPartsList, bodyPartsLayout)
    for _, child in pairs(bodyPartsList:GetChildren()) do
        if child ~= bodyPartsLayout then
            child:Destroy()
        end
    end
    
    for i, savedData in ipairs(savedPositions) do
        local keyframeButton = Instance.new("TextButton")
        keyframeButton.Size = UDim2.new(0, 50, 1, 0)
        local isSelected = (editingKeyframe and editingKeyframe.type == "fullbody" and editingKeyframe.index == i)
        keyframeButton.BackgroundColor3 = isSelected and THEME.WHITE or THEME.ORANGE
        keyframeButton.TextColor3 = isSelected and THEME.BLACK or THEME.WHITE
        keyframeButton.Text = "F" .. tostring(i)
        keyframeButton.TextScaled = true
        keyframeButton.Font = Enum.Font.GothamBold
        keyframeButton.BorderSizePixel = 0
        keyframeButton.Parent = bodyPartsList
        
        keyframeButton.MouseButton1Click:Connect(function()
            editingKeyframe = {type = "fullbody", index = i}
            updateBodyPartsList(bodyPartsList, bodyPartsLayout)
        end)
    end
    
    for partName, positions in pairs(bodyPartPositions) do
        for i, data in ipairs(positions) do
            local keyframeButton = Instance.new("TextButton")
            keyframeButton.Size = UDim2.new(0, 60, 1, 0)
            local isSelected = (editingKeyframe and editingKeyframe.type == "part" and editingKeyframe.partName == partName and editingKeyframe.index == i)
            keyframeButton.BackgroundColor3 = isSelected and THEME.WHITE or THEME.GRAY
            keyframeButton.TextColor3 = isSelected and THEME.BLACK or THEME.WHITE
            keyframeButton.Text = getPartType(partName) .. tostring(i)
            keyframeButton.TextScaled = true
            keyframeButton.Font = Enum.Font.Gotham
            keyframeButton.BorderSizePixel = 0
            keyframeButton.Parent = bodyPartsList
            
            keyframeButton.MouseButton1Click:Connect(function()
                editingKeyframe = {type = "part", partName = partName, index = i}
                updateBodyPartsList(bodyPartsList, bodyPartsLayout)
            end)
        end
    end
    
    bodyPartsLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        bodyPartsList.CanvasSize = UDim2.new(0, bodyPartsLayout.AbsoluteContentSize.X, 0, 0)
    end)
    bodyPartsList.CanvasSize = UDim2.new(0, bodyPartsLayout.AbsoluteContentSize.X, 0, 0)
end

-- Check if rig exists
local function checkRigExists(rig, rigInfoLabel, bodyPartsList, bodyPartsLayout)
    if not rig or not rig.Parent or not rig:FindFirstChild("Humanoid") then
        selectedRig = nil
        selectedPart = nil
        editingKeyframe = nil
        rigInfoLabel.Text = "Rig no longer exists - data cleared"
        removeHighlight()
        clearKeyframePreview()
        
        if isLooping and loopConnection then
            isLooping = false
            isPlaying = false
            loopConnection:Disconnect()
            loopConnection = nil
        end
        
        updateBodyPartsList(bodyPartsList, bodyPartsLayout)
        
        if rigCheckConnection then
            rigCheckConnection:Disconnect()
            rigCheckConnection = nil
        end
        
        spawn(function()
            wait(2)
            if not selectedRig then
                rigInfoLabel.Text = "No rig selected - Click on a character"
            end
        end)
        
        return false
    end
    return true
end

-- Delete keyframe
local function deleteSpecificKeyframe()
    if not editingKeyframe then 
        return false 
    end
    
    if editingKeyframe.type == "fullbody" then
        table.remove(savedPositions, editingKeyframe.index)
        if currentKeyframe > editingKeyframe.index then
            currentKeyframe = currentKeyframe - 1
        elseif currentKeyframe == editingKeyframe.index and currentKeyframe > #savedPositions then
            currentKeyframe = #savedPositions
        end
    elseif editingKeyframe.type == "part" then
        if bodyPartPositions[editingKeyframe.partName] then
            table.remove(bodyPartPositions[editingKeyframe.partName], editingKeyframe.index)
            if #bodyPartPositions[editingKeyframe.partName] == 0 then
                bodyPartPositions[editingKeyframe.partName] = nil
            end
        end
    end
    
    editingKeyframe = nil
    saveDataPersistent() -- Auto-save after deletion
    return true
end

-- Create export selection GUI with infinite loop option
local function createExportSelectionGui(callback)
    local exportGui = Instance.new("ScreenGui")
    exportGui.Name = "ExportSelectionGui"
    exportGui.ResetOnSpawn = false
    exportGui.Parent = playerGui
    
    local bg = Instance.new("Frame")
    bg.Size = UDim2.new(1, 0, 1, 0)
    bg.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    bg.BackgroundTransparency = 0.5
    bg.Parent = exportGui
    
    local mainFrame = Instance.new("Frame")
    mainFrame.Size = UDim2.new(0, 650, 0, 500)
    mainFrame.Position = UDim2.new(0.5, -325, 0.5, -250)
    mainFrame.BackgroundColor3 = THEME.BLACK
    mainFrame.BorderSizePixel = 2
    mainFrame.BorderColor3 = THEME.ORANGE
    mainFrame.Parent = bg
    
    local titleBar = Instance.new("Frame")
    titleBar.Size = UDim2.new(1, 0, 0, 40)
    titleBar.BackgroundColor3 = THEME.ORANGE
    titleBar.BorderSizePixel = 0
    titleBar.Parent = mainFrame
    
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -40, 1, 0)
    title.Position = UDim2.new(0, 10, 0, 0)
    title.BackgroundTransparency = 1
    title.Text = "EXPORT SCRIPT - Choose Settings"
    title.TextColor3 = THEME.WHITE
    title.TextScaled = true
    title.Font = Enum.Font.GothamBold
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = titleBar
    
    local closeBtn = Instance.new("TextButton")
    closeBtn.Size = UDim2.new(0, 40, 0, 40)
    closeBtn.Position = UDim2.new(1, -40, 0, 0)
    closeBtn.BackgroundColor3 = THEME.RED
    closeBtn.Text = "X"
    closeBtn.TextColor3 = THEME.WHITE
    closeBtn.TextScaled = true
    closeBtn.Font = Enum.Font.GothamBold
    closeBtn.BorderSizePixel = 0
    closeBtn.Parent = titleBar
    
    -- Mode selection
    local modeLabel = Instance.new("TextLabel")
    modeLabel.Size = UDim2.new(1, -20, 0, 30)
    modeLabel.Position = UDim2.new(0, 10, 0, 60)
    modeLabel.BackgroundTransparency = 1
    modeLabel.Text = "CHOOSE MODE:"
    modeLabel.TextColor3 = THEME.WHITE
    modeLabel.TextScaled = true
    modeLabel.Font = Enum.Font.GothamBold
    modeLabel.TextXAlignment = Enum.TextXAlignment.Left
    modeLabel.Parent = mainFrame
    
    local rigButton = Instance.new("TextButton")
    rigButton.Size = UDim2.new(0, 280, 0, 80)
    rigButton.Position = UDim2.new(0, 40, 0, 100)
    rigButton.BackgroundColor3 = THEME.BLUE
    rigButton.Text = "RIG\n(For NPCs/Models)"
    rigButton.TextColor3 = THEME.WHITE
    rigButton.TextScaled = true
    rigButton.Font = Enum.Font.GothamBold
    rigButton.BorderSizePixel = 0
    rigButton.Parent = mainFrame
    
    local playerButton = Instance.new("TextButton")
    playerButton.Size = UDim2.new(0, 280, 0, 80)
    playerButton.Position = UDim2.new(0, 330, 0, 100)
    playerButton.BackgroundColor3 = THEME.GREEN
    playerButton.Text = "PLAYER\n(For Players)"
    playerButton.TextColor3 = THEME.WHITE
    playerButton.TextScaled = true
    playerButton.Font = Enum.Font.GothamBold
    playerButton.BorderSizePixel = 0
    playerButton.Parent = mainFrame
    
    -- Type selection frame (initially hidden)
    local typeFrame = Instance.new("Frame")
    typeFrame.Size = UDim2.new(1, -20, 0, 180)
    typeFrame.Position = UDim2.new(0, 10, 0, 200)
    typeFrame.BackgroundColor3 = THEME.DARK_GRAY
    typeFrame.BorderSizePixel = 1
    typeFrame.BorderColor3 = THEME.GRAY
    typeFrame.Visible = false
    typeFrame.Parent = mainFrame
    
    local typeLabel = Instance.new("TextLabel")
    typeLabel.Size = UDim2.new(1, -10, 0, 30)
    typeLabel.Position = UDim2.new(0, 5, 0, 10)
    typeLabel.BackgroundTransparency = 1
    typeLabel.Text = "CHOOSE TYPE:"
    typeLabel.TextColor3 = THEME.WHITE
    typeLabel.TextScaled = true
    typeLabel.Font = Enum.Font.GothamBold
    typeLabel.TextXAlignment = Enum.TextXAlignment.Left
    typeLabel.Parent = typeFrame
    
    local staticButton = Instance.new("TextButton")
    staticButton.Size = UDim2.new(0, 280, 0, 60)
    staticButton.Position = UDim2.new(0, 40, 0, 50)
    staticButton.BackgroundColor3 = THEME.PURPLE
    staticButton.TextColor3 = THEME.WHITE
    staticButton.TextScaled = true
    staticButton.Font = Enum.Font.GothamBold
    staticButton.BorderSizePixel = 0
    staticButton.Parent = typeFrame
    
    local walkingButton = Instance.new("TextButton")
    walkingButton.Size = UDim2.new(0, 280, 0, 60)
    walkingButton.Position = UDim2.new(0, 330, 0, 50)
    walkingButton.BackgroundColor3 = THEME.ORANGE
    walkingButton.TextColor3 = THEME.WHITE
    walkingButton.TextScaled = true
    walkingButton.Font = Enum.Font.GothamBold
    walkingButton.BorderSizePixel = 0
    walkingButton.Parent = typeFrame
    
    -- Loop selection frame (initially hidden)
    local loopFrame = Instance.new("Frame")
    loopFrame.Size = UDim2.new(1, -20, 0, 120)
    loopFrame.Position = UDim2.new(0, 10, 0, 390)
    loopFrame.BackgroundColor3 = THEME.DARK_GRAY
    loopFrame.BorderSizePixel = 1
    loopFrame.BorderColor3 = THEME.GRAY
    loopFrame.Visible = false
    loopFrame.Parent = mainFrame
    
    local loopLabel = Instance.new("TextLabel")
    loopLabel.Size = UDim2.new(1, -10, 0, 30)
    loopLabel.Position = UDim2.new(0, 5, 0, 10)
    loopLabel.BackgroundTransparency = 1
    loopLabel.Text = "ANIMATION LOOP:"
    loopLabel.TextColor3 = THEME.WHITE
    loopLabel.TextScaled = true
    loopLabel.Font = Enum.Font.GothamBold
    loopLabel.TextXAlignment = Enum.TextXAlignment.Left
    loopLabel.Parent = loopFrame
    
    local onceButton = Instance.new("TextButton")
    onceButton.Size = UDim2.new(0, 280, 0, 60)
    onceButton.Position = UDim2.new(0, 40, 0, 50)
    onceButton.BackgroundColor3 = THEME.BLUE
    onceButton.Text = "PLAY ONCE\n(Single Execution)"
    onceButton.TextColor3 = THEME.WHITE
    onceButton.TextScaled = true
    onceButton.Font = Enum.Font.GothamBold
    onceButton.BorderSizePixel = 0
    onceButton.Parent = loopFrame
    
    local infiniteButton = Instance.new("TextButton")
    infiniteButton.Size = UDim2.new(0, 280, 0, 60)
    infiniteButton.Position = UDim2.new(0, 330, 0, 50)
    infiniteButton.BackgroundColor3 = THEME.GREEN
    infiniteButton.Text = "INFINITE LOOP\n(Repeat Forever)"
    infiniteButton.TextColor3 = THEME.WHITE
    infiniteButton.TextScaled = true
    infiniteButton.Font = Enum.Font.GothamBold
    infiniteButton.BorderSizePixel = 0
    infiniteButton.Parent = loopFrame
    
    local selectedMode = nil
    local selectedType = nil
    
    -- Mode selection handlers
    rigButton.MouseButton1Click:Connect(function()
        selectedMode = "rig"
        rigButton.BackgroundColor3 = THEME.WHITE
        rigButton.TextColor3 = THEME.BLACK
        playerButton.BackgroundColor3 = THEME.GREEN
        playerButton.TextColor3 = THEME.WHITE
        
        staticButton.Text = "STATIC\n(Very Low Velocity < 0.5)"
        walkingButton.Text = "WALKING\n(Higher Velocity > 0.5)"
        
        typeFrame.Visible = true
        loopFrame.Visible = false
        selectedType = nil
    end)
    
    playerButton.MouseButton1Click:Connect(function()
        selectedMode = "player"
        playerButton.BackgroundColor3 = THEME.WHITE
        playerButton.TextColor3 = THEME.BLACK
        rigButton.BackgroundColor3 = THEME.BLUE
        rigButton.TextColor3 = THEME.WHITE
        
        staticButton.Text = "STATIC\n(Very Low Velocity < 0.5)"
        walkingButton.Text = "WALKING\n(Higher Velocity > 0.5)"
        
        typeFrame.Visible = true
        loopFrame.Visible = false
        selectedType = nil
    end)
    
    -- Type selection handlers
    staticButton.MouseButton1Click:Connect(function()
        if selectedMode then
            selectedType = "static"
            staticButton.BackgroundColor3 = THEME.WHITE
            staticButton.TextColor3 = THEME.BLACK
            walkingButton.BackgroundColor3 = THEME.PURPLE
            walkingButton.TextColor3 = THEME.WHITE
            
            loopFrame.Visible = true
        end
    end)
    
    walkingButton.MouseButton1Click:Connect(function()
        if selectedMode then
            selectedType = "walking"
            walkingButton.BackgroundColor3 = THEME.WHITE
            walkingButton.TextColor3 = THEME.BLACK
            staticButton.BackgroundColor3 = THEME.PURPLE
            staticButton.TextColor3 = THEME.WHITE
            
            loopFrame.Visible = true
        end
    end)
    
    -- Loop selection handlers
    onceButton.MouseButton1Click:Connect(function()
        if selectedMode and selectedType then
            callback(selectedMode, selectedType, false)
            exportGui:Destroy()
        end
    end)
    
    infiniteButton.MouseButton1Click:Connect(function()
        if selectedMode and selectedType then
            callback(selectedMode, selectedType, true)
            exportGui:Destroy()
        end
    end)
    
    closeBtn.MouseButton1Click:Connect(function()
        exportGui:Destroy()
    end)
    
    bg.MouseButton1Click:Connect(function()
        exportGui:Destroy()
    end)
end")
    bg.Size = UDim2.new(1, 0, 1, 0)
    bg.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    bg.BackgroundTransparency = 0.5
    bg.Parent = exportGui
    
    local mainFrame = Instance.new("Frame")
    mainFrame.Size = UDim2.new(0, 600, 0, 400)
    mainFrame.Position = UDim2.new(0.5, -300, 0.5, -200)
    mainFrame.BackgroundColor3 = THEME.BLACK
    mainFrame.BorderSizePixel = 2
    mainFrame.BorderColor3 = THEME.ORANGE
    mainFrame.Parent = bg
    
    local titleBar = Instance.new("Frame")
    titleBar.Size = UDim2.new(1, 0, 0, 40)
    titleBar.BackgroundColor3 = THEME.ORANGE
    titleBar.BorderSizePixel = 0
    titleBar.Parent = mainFrame
    
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -40, 1, 0)
    title.Position = UDim2.new(0, 10, 0, 0)
    title.BackgroundTransparency = 1
    title.Text = "EXPORT SCRIPT - Choose Type"
    title.TextColor3 = THEME.WHITE
    title.TextScaled = true
    title.Font = Enum.Font.GothamBold
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = titleBar
    
    local closeBtn = Instance.new("TextButton")
    closeBtn.Size = UDim2.new(0, 40, 0, 40)
    closeBtn.Position = UDim2.new(1, -40, 0, 0)
    closeBtn.BackgroundColor3 = THEME.RED
    closeBtn.Text = "X"
    closeBtn.TextColor3 = THEME.WHITE
    closeBtn.TextScaled = true
    closeBtn.Font = Enum.Font.GothamBold
    closeBtn.BorderSizePixel = 0
    closeBtn.Parent = titleBar
    
    -- Mode selection
    local modeLabel = Instance.new("TextLabel")
    modeLabel.Size = UDim2.new(1, -20, 0, 30)
    modeLabel.Position = UDim2.new(0, 10, 0, 60)
    modeLabel.BackgroundTransparency = 1
    modeLabel.Text = "CHOOSE MODE:"
    modeLabel.TextColor3 = THEME.WHITE
    modeLabel.TextScaled = true
    modeLabel.Font = Enum.Font.GothamBold
    modeLabel.TextXAlignment = Enum.TextXAlignment.Left
    modeLabel.Parent = mainFrame
    
    local rigButton = Instance.new("TextButton")
    rigButton.Size = UDim2.new(0, 250, 0, 80)
    rigButton.Position = UDim2.new(0, 50, 0, 100)
    rigButton.BackgroundColor3 = THEME.BLUE
    rigButton.Text = "RIG\n(For NPCs/Models)"
    rigButton.TextColor3 = THEME.WHITE
    rigButton.TextScaled = true
    rigButton.Font = Enum.Font.GothamBold
    rigButton.BorderSizePixel = 0
    rigButton.Parent = mainFrame
    
    local playerButton = Instance.new("TextButton")
    playerButton.Size = UDim2.new(0, 250, 0, 80)
    playerButton.Position = UDim2.new(0, 320, 0, 100)
    playerButton.BackgroundColor3 = THEME.GREEN
    playerButton.Text = "PLAYER\n(For Players)"
    playerButton.TextColor3 = THEME.WHITE
    playerButton.TextScaled = true
    playerButton.Font = Enum.Font.GothamBold
    playerButton.BorderSizePixel = 0
    playerButton.Parent = mainFrame
    
    -- Type selection frame (initially hidden)
    local typeFrame = Instance.new("Frame")
    typeFrame.Size = UDim2.new(1, -20, 0, 180)
    typeFrame.Position = UDim2.new(0, 10, 0, 200)
    typeFrame.BackgroundColor3 = THEME.DARK_GRAY
    typeFrame.BorderSizePixel = 1
    typeFrame.BorderColor3 = THEME.GRAY
    typeFrame.Visible = false
    typeFrame.Parent = mainFrame
    
    local typeLabel = Instance.new("TextLabel")
    typeLabel.Size = UDim2.new(1, -10, 0, 30)
    typeLabel.Position = UDim2.new(0, 5, 0, 10)
    typeLabel.BackgroundTransparency = 1
    typeLabel.Text = "CHOOSE TYPE:"
    typeLabel.TextColor3 = THEME.WHITE
    typeLabel.TextScaled = true
    typeLabel.Font = Enum.Font.GothamBold
    typeLabel.TextXAlignment = Enum.TextXAlignment.Left
    typeLabel.Parent = typeFrame
    
    local staticButton = Instance.new("TextButton")
    staticButton.Size = UDim2.new(0, 250, 0, 60)
    staticButton.Position = UDim2.new(0, 40, 0, 50)
    staticButton.BackgroundColor3 = THEME.PURPLE
    staticButton.TextColor3 = THEME.WHITE
    staticButton.TextScaled = true
    staticButton.Font = Enum.Font.GothamBold
    staticButton.BorderSizePixel = 0
    staticButton.Parent = typeFrame
    
    local walkingButton = Instance.new("TextButton")
    walkingButton.Size = UDim2.new(0, 250, 0, 60)
    walkingButton.Position = UDim2.new(0, 310, 0, 50)
    walkingButton.BackgroundColor3 = THEME.ORANGE
    walkingButton.TextColor3 = THEME.WHITE
    walkingButton.TextScaled = true
    walkingButton.Font = Enum.Font.GothamBold
    walkingButton.BorderSizePixel = 0
    walkingButton.Parent = typeFrame
    
    local selectedMode = nil
    
    -- Mode selection handlers
    rigButton.MouseButton1Click:Connect(function()
        selectedMode = "rig"
        rigButton.BackgroundColor3 = THEME.WHITE
        rigButton.TextColor3 = THEME.BLACK
        playerButton.BackgroundColor3 = THEME.GREEN
        playerButton.TextColor3 = THEME.WHITE
        
        staticButton.Text = "STATIC\n(Low Velocity/Anchored)"
        walkingButton.Text = "WALKING\n(High Velocity/Moving)"
        
        typeFrame.Visible = true
    end)
    
    playerButton.MouseButton1Click:Connect(function()
        selectedMode = "player"
        playerButton.BackgroundColor3 = THEME.WHITE
        playerButton.TextColor3 = THEME.BLACK
        rigButton.BackgroundColor3 = THEME.BLUE
        rigButton.TextColor3 = THEME.WHITE
        
        staticButton.Text = "STATIC\n(Check Velocity)"
        walkingButton.Text = "WALKING\n(Check Velocity)"
        
        typeFrame.Visible = true
    end)
    
    -- Type selection handlers
    staticButton.MouseButton1Click:Connect(function()
        if selectedMode then
            callback(selectedMode, "static")
            exportGui:Destroy()
        end
    end)
    
    walkingButton.MouseButton1Click:Connect(function()
        if selectedMode then
            callback(selectedMode, "walking")
            exportGui:Destroy()
        end
    end)
    
    closeBtn.MouseButton1Click:Connect(function()
        exportGui:Destroy()
    end)
    
    bg.MouseButton1Click:Connect(function()
        exportGui:Destroy()
    end)
end

-- Create load scripts GUI
local function createLoadScriptsGui()
    local loadGui = Instance.new("ScreenGui")
    loadGui.Name = "LoadScriptsGui"
    loadGui.ResetOnSpawn = false
    loadGui.Parent = playerGui
    
    local bg = Instance.new("Frame")
    bg.Size = UDim2.new(1, 0, 1, 0)
    bg.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    bg.BackgroundTransparency = 0.5
    bg.Parent = loadGui
    
    local mainFrame = Instance.new("Frame")
    mainFrame.Size = UDim2.new(0, 700, 0, 500)
    mainFrame.Position = UDim2.new(0.5, -350, 0.5, -250)
    mainFrame.BackgroundColor3 = THEME.BLACK
    mainFrame.BorderSizePixel = 2
    mainFrame.BorderColor3 = THEME.ORANGE
    mainFrame.Parent = bg
    
    local titleBar = Instance.new("Frame")
    titleBar.Size = UDim2.new(1, 0, 0, 40)
    titleBar.BackgroundColor3 = THEME.ORANGE
    titleBar.BorderSizePixel = 0
    titleBar.Parent = mainFrame
    
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -40, 1, 0)
    title.Position = UDim2.new(0, 10, 0, 0)
    title.BackgroundTransparency = 1
    title.Text = "LOAD EXPORTED SCRIPTS"
    title.TextColor3 = THEME.WHITE
    title.TextScaled = true
    title.Font = Enum.Font.GothamBold
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = titleBar
    
    local closeBtn = Instance.new("TextButton")
    closeBtn.Size = UDim2.new(0, 40, 0, 40)
    closeBtn.Position = UDim2.new(1, -40, 0, 0)
    closeBtn.BackgroundColor3 = THEME.RED
    closeBtn.Text = "X"
    closeBtn.TextColor3 = THEME.WHITE
    closeBtn.TextScaled = true
    closeBtn.Font = Enum.Font.GothamBold
    closeBtn.BorderSizePixel = 0
    closeBtn.Parent = titleBar
    
    local scrollFrame = Instance.new("ScrollingFrame")
    scrollFrame.Size = UDim2.new(1, -20, 1, -60)
    scrollFrame.Position = UDim2.new(0, 10, 0, 50)
    scrollFrame.BackgroundColor3 = THEME.DARK_GRAY
    scrollFrame.BorderSizePixel = 1
    scrollFrame.BorderColor3 = THEME.GRAY
    scrollFrame.ScrollBarThickness = 8
    scrollFrame.ScrollBarImageColor3 = THEME.ORANGE
    scrollFrame.Parent = mainFrame
    
    local listLayout = Instance.new("UIListLayout")
    listLayout.Padding = UDim.new(0, 5)
    listLayout.Parent = scrollFrame
    
    if #exportedScripts == 0 then
        local noScriptsLabel = Instance.new("TextLabel")
        noScriptsLabel.Size = UDim2.new(1, -10, 0, 100)
        noScriptsLabel.Position = UDim2.new(0, 5, 0, 10)
        noScriptsLabel.BackgroundTransparency = 1
        noScriptsLabel.Text = "No exported scripts yet.\nExport some animations first!"
        noScriptsLabel.TextColor3 = THEME.LIGHT_GRAY
        noScriptsLabel.TextScaled = true
        noScriptsLabel.Font = Enum.Font.Gotham
        noScriptsLabel.Parent = scrollFrame
    else
        for i, scriptData in ipairs(exportedScripts) do
            local scriptFrame = Instance.new("Frame")
            scriptFrame.Size = UDim2.new(1, -10, 0, 80)
            scriptFrame.BackgroundColor3 = THEME.GRAY
            scriptFrame.BorderSizePixel = 1
            scriptFrame.BorderColor3 = THEME.ORANGE
            scriptFrame.Parent = scrollFrame
            
            local scriptName = Instance.new("TextLabel")
            scriptName.Size = UDim2.new(0.4, 0, 0.5, 0)
            scriptName.Position = UDim2.new(0, 10, 0, 5)
            scriptName.BackgroundTransparency = 1
            scriptName.Text = scriptData.name
            scriptName.TextColor3 = THEME.WHITE
            scriptName.TextScaled = true
            scriptName.Font = Enum.Font.GothamBold
            scriptName.TextXAlignment = Enum.TextXAlignment.Left
            scriptName.Parent = scriptFrame
            
            local scriptInfo = Instance.new("TextLabel")
            scriptInfo.Size = UDim2.new(0.4, 0, 0.5, 0)
            scriptInfo.Position = UDim2.new(0, 10, 0.5, 0)
            scriptInfo.BackgroundTransparency = 1
            scriptInfo.Text = string.format("%s | %s | %d keyframes%s", 
                scriptData.mode:upper(), 
                scriptData.scriptType:upper(), 
                scriptData.keyframeCount,
                scriptData.infiniteLoop and " | INFINITE" or " | ONCE")
            scriptInfo.TextColor3 = THEME.LIGHT_GRAY
            scriptInfo.TextScaled = true
            scriptInfo.Font = Enum.Font.Gotham
            scriptInfo.TextXAlignment = Enum.TextXAlignment.Left
            scriptInfo.Parent = scriptFrame
            
            local loadButton = Instance.new("TextButton")
            loadButton.Size = UDim2.new(0, 100, 0, 30)
            loadButton.Position = UDim2.new(1, -220, 0, 10)
            loadButton.BackgroundColor3 = THEME.GREEN
            loadButton.Text = "LOAD"
            loadButton.TextColor3 = THEME.WHITE
            loadButton.TextScaled = true
            loadButton.Font = Enum.Font.GothamBold
            loadButton.BorderSizePixel = 0
            loadButton.Parent = scriptFrame
            
            local copyButton = Instance.new("TextButton")
            copyButton.Size = UDim2.new(0, 100, 0, 30)
            copyButton.Position = UDim2.new(1, -110, 0, 10)
            copyButton.BackgroundColor3 = THEME.BLUE
            copyButton.Text = "COPY"
            copyButton.TextColor3 = THEME.WHITE
            copyButton.TextScaled = true
            copyButton.Font = Enum.Font.GothamBold
            copyButton.BorderSizePixel = 0
            copyButton.Parent = scriptFrame
            
            local deleteButton = Instance.new("TextButton")
            deleteButton.Size = UDim2.new(0, 30, 0, 30)
            deleteButton.Position = UDim2.new(1, -40, 0, 40)
            deleteButton.BackgroundColor3 = THEME.RED
            deleteButton.Text = "X"
            deleteButton.TextColor3 = THEME.WHITE
            deleteButton.TextScaled = true
            deleteButton.Font = Enum.Font.GothamBold
            deleteButton.BorderSizePixel = 0
            deleteButton.Parent = scriptFrame
            
            loadButton.MouseButton1Click:Connect(function()
                loadExportedScript(scriptData)
                loadGui:Destroy()
            end)
            
            copyButton.MouseButton1Click:Connect(function()
                if setclipboard then
                    setclipboard(scriptData.code)
                    copyButton.Text = "OK"
                    copyButton.BackgroundColor3 = THEME.GREEN
                    spawn(function()
                        wait(1)
                        copyButton.Text = "COPY"
                        copyButton.BackgroundColor3 = THEME.BLUE
                    end)
                else
                    print(scriptData.code)
                    copyButton.Text = "CONSOLE"
                    spawn(function()
                        wait(1)
                        copyButton.Text = "COPY"
                    end)
                end
            end)
            
            deleteButton.MouseButton1Click:Connect(function()
                table.remove(exportedScripts, i)
                saveDataPersistent() -- Save after deletion
                loadGui:Destroy()
                createLoadScriptsGui()
            end)
        end
    end
    
    listLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        scrollFrame.CanvasSize = UDim2.new(0, 0, 0, listLayout.AbsoluteContentSize.Y + 10)
    end)
    scrollFrame.CanvasSize = UDim2.new(0, 0, 0, listLayout.AbsoluteContentSize.Y + 10)
    
    closeBtn.MouseButton1Click:Connect(function()
        loadGui:Destroy()
    end)
    
    bg.MouseButton1Click:Connect(function()
        loadGui:Destroy()
    end)
end

-- Play single animation with relative positioning
local function playAnimation(rig)
    if (#savedPositions == 0 and not next(bodyPartPositions)) or not rig then 
        return 
    end
    
    local rootPart = rig:FindFirstChild("HumanoidRootPart")
    if not rootPart then
        print("Warning: No HumanoidRootPart found for animation")
        return
    end
    
    isPlaying = true
    clearKeyframePreview()
    
    for i = 1, #savedPositions do
        if not isPlaying then break end
        
        local savedData = savedPositions[i]
        local duration = savedData.duration or keyframeDuration
        
        local tweenInfo = TweenInfo.new(duration, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
        local tweens = {}
        
        for partName, data in pairs(savedData.parts) do
            local part = rig:FindFirstChild(partName)
            
            if part and part:IsA("BasePart") then
                local targetCFrame
                if part == rootPart then
                    targetCFrame = rootPart.CFrame * data.CFrame
                else
                    targetCFrame = rootPart.CFrame * data.CFrame
                end
                
                local tween = TweenService:Create(part, tweenInfo, {
                    CFrame = targetCFrame,
                    Size = data.Size
                })
                table.insert(tweens, tween)
                tween:Play()
            end
        end
        
        if #tweens > 0 then
            tweens[1].Completed:Wait()
        end
        
        if i < #savedPositions then
            wait(0.05)
        end
    end
    
    for partName, positions in pairs(bodyPartPositions) do
        if not isPlaying then break end
        
        local part = rig:FindFirstChild(partName)
        
        if part then
            for i, data in ipairs(positions) do
                if not isPlaying then break end
                
                local targetCFrame
                if part == rootPart then
                    targetCFrame = rootPart.CFrame * data.CFrame
                else
                    targetCFrame = rootPart.CFrame * data.CFrame
                end
                
                local tweenInfo = TweenInfo.new(data.duration, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
                local tween = TweenService:Create(part, tweenInfo, {
                    CFrame = targetCFrame,
                    Size = data.Size
                })
                
                tween:Play()
                tween.Completed:Wait()
                
                if i < #positions then
                    wait(0.05)
                end
            end
        end
    end
    
    isPlaying = false
end

-- Play loop animation
local function playAnimationLoop(rig)
    if (#savedPositions == 0 and not next(bodyPartPositions)) or not rig then 
        return 
    end
    
    local rootPart = rig:FindFirstChild("HumanoidRootPart")
    if not rootPart then
        print("Warning: No HumanoidRootPart found for loop animation")
        return
    end
    
    isPlaying = true
    isLooping = true
    clearKeyframePreview()
    
    local function playOnce()
        local startIndex = math.max(1, math.min(startFromKeyframe, #savedPositions))
        
        for i = startIndex, #savedPositions do
            if not isLooping or not isPlaying then break end
            
            local savedData = savedPositions[i]
            local duration = savedData.duration or keyframeDuration
            
            local tweenInfo = TweenInfo.new(duration, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
            local tweens = {}
            
            for partName, data in pairs(savedData.parts) do
                local part = rig:FindFirstChild(partName)
                
                if part and part:IsA("BasePart") then
                    local targetCFrame
                    if part == rootPart then
                        targetCFrame = rootPart.CFrame * data.CFrame
                    else
                        targetCFrame = rootPart.CFrame * data.CFrame
                    end
                    
                    local tween = TweenService:Create(part, tweenInfo, {
                        CFrame = targetCFrame,
                        Size = data.Size
                    })
                    table.insert(tweens, tween)
                    tween:Play()
                end
            end
            
            if #tweens > 0 then
                tweens[1].Completed:Wait()
            end
            
            if i < #savedPositions then
                wait(0.1)
            end
        end
        
        for partName, positions in pairs(bodyPartPositions) do
            if not isLooping or not isPlaying then break end
            
            local part = rig:FindFirstChild(partName)
            
            if part then
                for i, data in ipairs(positions) do
                    if not isLooping or not isPlaying then break end
                    
                    local targetCFrame
                    if part == rootPart then
                        targetCFrame = rootPart.CFrame * data.CFrame
                    else
                        targetCFrame = rootPart.CFrame * data.CFrame
                    end
                    
                    local tweenInfo = TweenInfo.new(data.duration, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
                    local tween = TweenService:Create(part, tweenInfo, {
                        CFrame = targetCFrame,
                        Size = data.Size
                    })
                    
                    tween:Play()
                    tween.Completed:Wait()
                    
                    if i < #positions then
                        wait(0.1)
                    end
                end
            end
        end
        
        wait(0.5)
    end
    
    loopConnection = spawn(function()
        while isLooping and isPlaying do
            playOnce()
            if isLooping and isPlaying then
                wait(0.2)
            end
        end
    end)
end

-- Show preview
local function showKeyframePreview(rig)
    clearKeyframePreview()
    
    if not rig or (#savedPositions == 0 and not next(bodyPartPositions)) then 
        return 
    end
    
    local colors = {THEME.ORANGE, THEME.GRAY, THEME.LIGHT_GRAY}
    
    for i, savedData in ipairs(savedPositions) do
        local color = colors[((i-1) % #colors) + 1]
        
        for partName, data in pairs(savedData.parts) do
            local originalPart = rig:FindFirstChild(partName)
            
            if originalPart and originalPart:IsA("BasePart") then
                local ghostPart = originalPart:Clone()
                ghostPart.Name = "GhostPart_Full_" .. tostring(i) .. "_" .. partName
                ghostPart.Parent = workspace
                ghostPart.CFrame = data.CFrame
                ghostPart.Size = data.Size
                ghostPart.Transparency = 0.9
                ghostPart.CanCollide = false
                ghostPart.Anchored = true
                
                for _, child in pairs(ghostPart:GetChildren()) do
                    if not child:IsA("SpecialMesh") and not child:IsA("Decal") then
                        child:Destroy()
                    end
                end
                
                local previewHighlight = Instance.new("Highlight")
                previewHighlight.Parent = ghostPart
                previewHighlight.FillColor = color
                previewHighlight.OutlineColor = THEME.WHITE
                previewHighlight.FillTransparency = 0.7
                previewHighlight.OutlineTransparency = 0.2
                
                table.insert(previewConnections, {
                    part = ghostPart,
                    highlight = previewHighlight
                })
            end
        end
    end
end

-- Reset to initial position
local function resetToInitial(rig)
    if not rig or #savedPositions == 0 then 
        return 
    end
    
    local initialData = savedPositions[1]
    if not initialData then 
        return 
    end
    
    for partName, data in pairs(initialData.parts) do
        local part = rig:FindFirstChild(partName)
        if part and part:IsA("BasePart") then
            part.CFrame = data.CFrame
            part.Size = data.Size
        end
    end
end

-- Create main GUI
local function createMainGui()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "MoonAnimatorX"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = playerGui
    
    local mainFrame = Instance.new("Frame")
    mainFrame.Size = UDim2.new(0, 950, 0, 220)
    mainFrame.Position = UDim2.new(0.5, -475, 0, 10)
    mainFrame.BackgroundColor3 = THEME.BLACK
    mainFrame.BorderSizePixel = 2
    mainFrame.BorderColor3 = THEME.ORANGE
    mainFrame.Active = true
    mainFrame.Draggable = true
    mainFrame.Parent = screenGui
    
    local titleBar = Instance.new("Frame")
    titleBar.Size = UDim2.new(1, 0, 0, 30)
    titleBar.BackgroundColor3 = THEME.ORANGE
    titleBar.BorderSizePixel = 0
    titleBar.Parent = mainFrame
    
    local titleLabel = Instance.new("TextLabel")
    titleLabel.Size = UDim2.new(1, -35, 1, 0)
    titleLabel.Position = UDim2.new(0, 5, 0, 0)
    titleLabel.BackgroundTransparency = 1
    titleLabel.Text = "Moon Animator X - Enhanced with Persistent Save"
    titleLabel.TextColor3 = THEME.WHITE
    titleLabel.TextScaled = true
    titleLabel.Font = Enum.Font.GothamBold
    titleLabel.TextXAlignment = Enum.TextXAlignment.Left
    titleLabel.Parent = titleBar
    
    local closeButton = Instance.new("TextButton")
    closeButton.Size = UDim2.new(0, 30, 0, 30)
    closeButton.Position = UDim2.new(1, -30, 0, 0)
    closeButton.BackgroundColor3 = THEME.RED
    closeButton.Text = "X"
    closeButton.TextColor3 = THEME.WHITE
    closeButton.TextScaled = true
    closeButton.Font = Enum.Font.GothamBold
    closeButton.BorderSizePixel = 0
    closeButton.Parent = titleBar
    
    local mainButtonsFrame = Instance.new("Frame")
    mainButtonsFrame.Size = UDim2.new(1, -10, 0, 50)
    mainButtonsFrame.Position = UDim2.new(0, 5, 0, 35)
    mainButtonsFrame.BackgroundColor3 = THEME.DARK_GRAY
    mainButtonsFrame.BorderSizePixel = 1
    mainButtonsFrame.BorderColor3 = THEME.GRAY
    mainButtonsFrame.Parent = mainFrame
    
    -- Create buttons with improved layout
    local playButton = Instance.new("TextButton")
    playButton.Size = UDim2.new(0, 70, 0, 35)
    playButton.Position = UDim2.new(0, 5, 0, 7)
    playButton.BackgroundColor3 = THEME.GREEN
    playButton.Text = "Play"
    playButton.TextColor3 = THEME.WHITE
    playButton.TextScaled = true
    playButton.Font = Enum.Font.GothamBold
    playButton.BorderSizePixel = 0
    playButton.Parent = mainButtonsFrame
    
    local loopButton = Instance.new("TextButton")
    loopButton.Size = UDim2.new(0, 70, 0, 35)
    loopButton.Position = UDim2.new(0, 80, 0, 7)
    loopButton.BackgroundColor3 = THEME.BLUE
    loopButton.Text = "Loop"
    loopButton.TextColor3 = THEME.WHITE
    loopButton.TextScaled = true
    loopButton.Font = Enum.Font.GothamBold
    loopButton.BorderSizePixel = 0
    loopButton.Parent = mainButtonsFrame
    
    local stopButton = Instance.new("TextButton")
    stopButton.Size = UDim2.new(0, 70, 0, 35)
    stopButton.Position = UDim2.new(0, 155, 0, 7)
    stopButton.BackgroundColor3 = THEME.RED
    stopButton.Text = "Stop"
    stopButton.TextColor3 = THEME.WHITE
    stopButton.TextScaled = true
    stopButton.Font = Enum.Font.GothamBold
    stopButton.BorderSizePixel = 0
    stopButton.Parent = mainButtonsFrame
    
    local resetButton = Instance.new("TextButton")
    resetButton.Size = UDim2.new(0, 70, 0, 35)
    resetButton.Position = UDim2.new(0, 230, 0, 7)
    resetButton.BackgroundColor3 = THEME.GRAY
    resetButton.Text = "Reset"
    resetButton.TextColor3 = THEME.WHITE
    resetButton.TextScaled = true
    resetButton.Font = Enum.Font.GothamBold
    resetButton.BorderSizePixel = 0
    resetButton.Parent = mainButtonsFrame
    
    local previewButton = Instance.new("TextButton")
    previewButton.Size = UDim2.new(0, 80, 0, 35)
    previewButton.Position = UDim2.new(0, 305, 0, 7)
    previewButton.BackgroundColor3 = THEME.PURPLE
    previewButton.Text = "Preview"
    previewButton.TextColor3 = THEME.WHITE
    previewButton.TextScaled = true
    previewButton.Font = Enum.Font.GothamBold
    previewButton.BorderSizePixel = 0
    previewButton.Parent = mainButtonsFrame
    
    local savePositionButton = Instance.new("TextButton")
    savePositionButton.Size = UDim2.new(0, 80, 0, 35)
    savePositionButton.Position = UDim2.new(0, 390, 0, 7)
    savePositionButton.BackgroundColor3 = THEME.ORANGE
    savePositionButton.Text = "Save"
    savePositionButton.TextColor3 = THEME.WHITE
    savePositionButton.TextScaled = true
    savePositionButton.Font = Enum.Font.GothamBold
    savePositionButton.BorderSizePixel = 0
    savePositionButton.Parent = mainButtonsFrame
    
    local exportButton = Instance.new("TextButton")
    exportButton.Size = UDim2.new(0, 80, 0, 35)
    exportButton.Position = UDim2.new(0, 475, 0, 7)
    exportButton.BackgroundColor3 = THEME.GREEN
    exportButton.Text = "Export"
    exportButton.TextColor3 = THEME.WHITE
    exportButton.TextScaled = true
    exportButton.Font = Enum.Font.GothamBold
    exportButton.BorderSizePixel = 0
    exportButton.Parent = mainButtonsFrame
    
    local loadButton = Instance.new("TextButton")
    loadButton.Size = UDim2.new(0, 80, 0, 35)
    loadButton.Position = UDim2.new(0, 560, 0, 7)
    loadButton.BackgroundColor3 = THEME.BLUE
    loadButton.Text = "Load"
    loadButton.TextColor3 = THEME.WHITE
    loadButton.TextScaled = true
    loadButton.Font = Enum.Font.GothamBold
    loadButton.BorderSizePixel = 0
    loadButton.Parent = mainButtonsFrame
    
    -- Duration and Start inputs
    local durationLabel = Instance.new("TextLabel")
    durationLabel.Size = UDim2.new(0, 60, 0, 15)
    durationLabel.Position = UDim2.new(0, 650, 0, 5)
    durationLabel.BackgroundTransparency = 1
    durationLabel.Text = "Duration:"
    durationLabel.TextColor3 = THEME.WHITE
    durationLabel.TextScaled = true
    durationLabel.Font = Enum.Font.Gotham
    durationLabel.Parent = mainButtonsFrame
    
    local durationInput = Instance.new("TextBox")
    durationInput.Size = UDim2.new(0, 50, 0, 25)
    durationInput.Position = UDim2.new(0, 650, 0, 20)
    durationInput.BackgroundColor3 = THEME.WHITE
    durationInput.Text = tostring(keyframeDuration)
    durationInput.TextColor3 = THEME.BLACK
    durationInput.TextScaled = true
    durationInput.Font = Enum.Font.Gotham
    durationInput.BorderSizePixel = 1
    durationInput.BorderColor3 = THEME.GRAY
    durationInput.Parent = mainButtonsFrame
    
    local startFromLabel = Instance.new("TextLabel")
    startFromLabel.Size = UDim2.new(0, 60, 0, 15)
    startFromLabel.Position = UDim2.new(0, 710, 0, 5)
    startFromLabel.BackgroundTransparency = 1
    startFromLabel.Text = "Start From:"
    startFromLabel.TextColor3 = THEME.WHITE
    startFromLabel.TextScaled = true
    startFromLabel.Font = Enum.Font.Gotham
    local startFromInput = Instance.new("TextBox")
    startFromInput.Size = UDim2.new(0, 50, 0, 25)
    startFromInput.Position = UDim2.new(0, 710, 0, 20)
    startFromInput.BackgroundColor3 = THEME.WHITE
    startFromInput.Text = "1"
    startFromInput.TextColor3 = THEME.BLACK
    startFromInput.TextScaled = true
    startFromInput.Font = Enum.Font.Gotham
    startFromInput.BorderSizePixel = 1
    startFromInput.BorderColor3 = THEME.GRAY
    startFromInput.Parent = mainButtonsFrame
    
    local bottomFrame = Instance.new("Frame")
    bottomFrame.Size = UDim2.new(1, -10, 1, -95)
    bottomFrame.Position = UDim2.new(0, 5, 0, 90)
    bottomFrame.BackgroundTransparency = 1
    bottomFrame.Parent = mainFrame
    
    local dangerFrame = Instance.new("Frame")
    dangerFrame.Size = UDim2.new(0.25, -5, 1, 0)
    dangerFrame.Position = UDim2.new(0, 0, 0, 0)
    dangerFrame.BackgroundColor3 = THEME.DARK_GRAY
    dangerFrame.BorderSizePixel = 1
    dangerFrame.BorderColor3 = THEME.GRAY
    dangerFrame.Parent = bottomFrame
    
    local dangerTitle = Instance.new("TextLabel")
    dangerTitle.Size = UDim2.new(1, 0, 0, 25)
    dangerTitle.BackgroundColor3 = THEME.RED
    dangerTitle.Text = "KEYFRAME CONTROLS"
    dangerTitle.TextColor3 = THEME.WHITE
    dangerTitle.TextScaled = true
    dangerTitle.Font = Enum.Font.GothamBold
    dangerTitle.BorderSizePixel = 0
    dangerTitle.Parent = dangerFrame
    
    local clearAllButton = Instance.new("TextButton")
    clearAllButton.Size = UDim2.new(1, -10, 0, 30)
    clearAllButton.Position = UDim2.new(0, 5, 0, 30)
    clearAllButton.BackgroundColor3 = THEME.RED
    clearAllButton.Text = "Clear All"
    clearAllButton.TextColor3 = THEME.WHITE
    clearAllButton.TextScaled = true
    clearAllButton.Font = Enum.Font.GothamBold
    clearAllButton.BorderSizePixel = 0
    clearAllButton.Parent = dangerFrame
    
    local deleteSelectedButton = Instance.new("TextButton")
    deleteSelectedButton.Size = UDim2.new(1, -10, 0, 30)
    deleteSelectedButton.Position = UDim2.new(0, 5, 0, 65)
    deleteSelectedButton.BackgroundColor3 = THEME.ORANGE
    deleteSelectedButton.Text = "Delete Selected"
    deleteSelectedButton.TextColor3 = THEME.WHITE
    deleteSelectedButton.TextScaled = true
    deleteSelectedButton.Font = Enum.Font.GothamBold
    deleteSelectedButton.BorderSizePixel = 0
    deleteSelectedButton.Parent = dangerFrame
    
    local infoFrame = Instance.new("Frame")
    infoFrame.Size = UDim2.new(0.75, -5, 1, 0)
    infoFrame.Position = UDim2.new(0.25, 5, 0, 0)
    infoFrame.BackgroundColor3 = THEME.DARK_GRAY
    infoFrame.BorderSizePixel = 1
    infoFrame.BorderColor3 = THEME.GRAY
    infoFrame.Parent = bottomFrame
    
    local infoTitle = Instance.new("TextLabel")
    infoTitle.Size = UDim2.new(1, 0, 0, 25)
    infoTitle.BackgroundColor3 = THEME.BLUE
    infoTitle.Text = "ANIMATION INFO"
    infoTitle.TextColor3 = THEME.WHITE
    infoTitle.TextScaled = true
    infoTitle.Font = Enum.Font.GothamBold
    infoTitle.BorderSizePixel = 0
    infoTitle.Parent = infoFrame
    
    local rigInfoLabel = Instance.new("TextLabel")
    rigInfoLabel.Size = UDim2.new(1, -10, 0, 15)
    rigInfoLabel.Position = UDim2.new(0, 5, 0, 30)
    rigInfoLabel.BackgroundTransparency = 1
    rigInfoLabel.Text = "No rig selected - Click on a character"
    rigInfoLabel.TextColor3 = THEME.WHITE
    rigInfoLabel.TextScaled = true
    rigInfoLabel.Font = Enum.Font.Gotham
    rigInfoLabel.TextXAlignment = Enum.TextXAlignment.Left
    rigInfoLabel.Parent = infoFrame
    
    local partInfoLabel = Instance.new("TextLabel")
    partInfoLabel.Size = UDim2.new(1, -10, 0, 15)
    partInfoLabel.Position = UDim2.new(0, 5, 0, 50)
    partInfoLabel.BackgroundTransparency = 1
    partInfoLabel.Text = "No part selected - Right click on body part"
    partInfoLabel.TextColor3 = THEME.LIGHT_GRAY
    partInfoLabel.TextScaled = true
    partInfoLabel.Font = Enum.Font.Gotham
    partInfoLabel.TextXAlignment = Enum.TextXAlignment.Left
    partInfoLabel.Parent = infoFrame
    
    local bodyPartsFrame = Instance.new("Frame")
    bodyPartsFrame.Size = UDim2.new(1, -10, 0, 40)
    bodyPartsFrame.Position = UDim2.new(0, 5, 1, -45)
    bodyPartsFrame.BackgroundColor3 = THEME.GRAY
    bodyPartsFrame.BorderSizePixel = 1
    bodyPartsFrame.BorderColor3 = THEME.ORANGE
    bodyPartsFrame.Parent = infoFrame
    
    local bodyPartsTitle = Instance.new("TextLabel")
    bodyPartsTitle.Size = UDim2.new(0, 100, 1, 0)
    bodyPartsTitle.BackgroundColor3 = THEME.ORANGE
    bodyPartsTitle.Text = "KEYFRAMES"
    bodyPartsTitle.TextColor3 = THEME.WHITE
    bodyPartsTitle.TextScaled = true
    bodyPartsTitle.Font = Enum.Font.GothamBold
    bodyPartsTitle.BorderSizePixel = 0
    bodyPartsTitle.Parent = bodyPartsFrame
    
    local bodyPartsList = Instance.new("ScrollingFrame")
    bodyPartsList.Size = UDim2.new(1, -100, 1, 0)
    bodyPartsList.Position = UDim2.new(0, 100, 0, 0)
    bodyPartsList.BackgroundTransparency = 1
    bodyPartsList.ScrollBarThickness = 8
    bodyPartsList.ScrollBarImageColor3 = THEME.ORANGE
    bodyPartsList.ScrollingDirection = Enum.ScrollingDirection.X
    bodyPartsList.CanvasSize = UDim2.new(0, 0, 0, 0)
    bodyPartsList.BorderSizePixel = 0
    bodyPartsList.ElasticBehavior = Enum.ElasticBehavior.Never
    bodyPartsList.Parent = bodyPartsFrame
    
    local bodyPartsLayout = Instance.new("UIListLayout")
    bodyPartsLayout.FillDirection = Enum.FillDirection.Horizontal
    bodyPartsLayout.Padding = UDim.new(0, 2)
    bodyPartsLayout.Parent = bodyPartsList
    
    return screenGui, rigInfoLabel, partInfoLabel, playButton, loopButton, stopButton, resetButton, previewButton, savePositionButton, exportButton, loadButton, clearAllButton, deleteSelectedButton, closeButton, mainFrame, durationInput, startFromInput, bodyPartsList, bodyPartsLayout
end

-- Main setup function
local function setupSystem()
    -- Load persistent data first
    loadDataPersistent()
    
    local gui, rigInfoLabel, partInfoLabel, playButton, loopButton, stopButton, resetButton, previewButton, savePositionButton, exportButton, loadButton, clearAllButton, deleteSelectedButton, closeButton, mainFrame, durationInput, startFromInput, bodyPartsList, bodyPartsLayout = createMainGui()
    
    -- Duration input
    durationInput.FocusLost:Connect(function()
        local newDuration = tonumber(durationInput.Text)
        if newDuration and newDuration > 0 then
            keyframeDuration = newDuration
            saveDataPersistent() -- Auto-save settings
        else
            durationInput.Text = tostring(keyframeDuration)
        end
    end)
    
    -- Start from input
    startFromInput.FocusLost:Connect(function()
        local newStart = tonumber(startFromInput.Text)
        if newStart and newStart > 0 then
            startFromKeyframe = math.ceil(newStart)
        else
            startFromInput.Text = tostring(startFromKeyframe)
        end
    end)
    
    -- Mouse selection
    mouse.Button1Down:Connect(function()
        local target = mouse.Target
        if target and target:IsA("BasePart") then
            local rig = findRigFromPart(target)
            
            if rig and detectRigType(rig) then
                selectedRig = rig
                selectedPart = nil
                editingKeyframe = nil
                rigInfoLabel.Text = "Selected: " .. rig.Name .. " (" .. detectRigType(rig) .. ")"
                partInfoLabel.Text = "No part selected - Right click on body part"
                removeHighlight()
                
                if rigCheckConnection then
                    rigCheckConnection:Disconnect()
                end
                
                rigCheckConnection = spawn(function()
                    while selectedRig do
                        wait(1)
                        if not checkRigExists(selectedRig, rigInfoLabel, bodyPartsList, bodyPartsLayout) then
                            break
                        end
                    end
                end)
            end
        end
    end)
    
    -- Right click part selection
    UserInputService.InputBegan:Connect(function(input, gameProcessed)
        if gameProcessed then return end
        
        if input.UserInputType == Enum.UserInputType.MouseButton2 then
            local target = mouse.Target
            if target and target:IsA("BasePart") and selectedRig then
                local rig = findRigFromPart(target)
                
                if rig == selectedRig then
                    selectedPart = target
                    partInfoLabel.Text = "Selected part: " .. target.Name
                    createHighlight(target)
                end
            end
        end
    end)
    
    local previewActive = false
    
    -- Save button
    savePositionButton.MouseButton1Click:Connect(function()
        if selectedRig and checkRigExists(selectedRig, rigInfoLabel, bodyPartsList, bodyPartsLayout) then
            local index, targetName, wasUpdated = savePosition(selectedRig, selectedPart, true)
            
            if wasUpdated then
                rigInfoLabel.Text = "Updated " .. targetName .. " keyframe #" .. tostring(index)
            else
                rigInfoLabel.Text = "Saved " .. targetName .. " keyframe #" .. tostring(index)
            end
            
            updateBodyPartsList(bodyPartsList, bodyPartsLayout)
            
            savePositionButton.BackgroundColor3 = THEME.WHITE
            savePositionButton.TextColor3 = THEME.BLACK
            spawn(function()
                wait(0.3)
                savePositionButton.BackgroundColor3 = THEME.ORANGE
                savePositionButton.TextColor3 = THEME.WHITE
            end)
        else
            rigInfoLabel.Text = "Select a rig first!"
        end
    end)
    
    -- Export button with selection GUI
    exportButton.MouseButton1Click:Connect(function()
        if #savedPositions > 0 or next(bodyPartPositions) then
            createExportSelectionGui(function(mode, scriptType, infiniteLoop)
                local loopText = infiniteLoop and "Infinite" or "Once"
                local scriptName = string.format("Animation_%s_%s_%s_%d", mode, scriptType, loopText, os.time())
                local code = generateExportScript(mode, scriptType, infiniteLoop)
                
                saveExportedScript(scriptName, code, mode, scriptType, infiniteLoop)
                
                if setclipboard then
                    setclipboard(code)
                    rigInfoLabel.Text = "Script exported and copied to clipboard!"
                else
                    print("-- EXPORTED ANIMATION SCRIPT --")
                    print(code)
                    rigInfoLabel.Text = "Script exported and printed to console!"
                end
                
                exportButton.BackgroundColor3 = THEME.WHITE
                exportButton.TextColor3 = THEME.BLACK
                spawn(function()
                    wait(0.3)
                    exportButton.BackgroundColor3 = THEME.GREEN
                    exportButton.TextColor3 = THEME.WHITE
                    wait(2)
                    if selectedRig then
                        rigInfoLabel.Text = "Selected: " .. selectedRig.Name
                    else
                        rigInfoLabel.Text = "No rig selected - Click on a character"
                    end
                end)
            end)
        else
            rigInfoLabel.Text = "No animation data to export!"
            exportButton.BackgroundColor3 = THEME.GRAY
            spawn(function()
                wait(0.5)
                exportButton.BackgroundColor3 = THEME.GREEN
            end)
        end
    end)
    
    -- Load button
    loadButton.MouseButton1Click:Connect(function()
        createLoadScriptsGui()
    end)
    
    -- Preview button
    previewButton.MouseButton1Click:Connect(function()
        if selectedRig and checkRigExists(selectedRig, rigInfoLabel, bodyPartsList, bodyPartsLayout) and (#savedPositions > 0 or next(bodyPartPositions)) then
            if previewActive then
                clearKeyframePreview()
                previewButton.Text = "Preview"
                previewButton.BackgroundColor3 = THEME.PURPLE
                previewActive = false
            else
                showKeyframePreview(selectedRig)
                previewButton.Text = "Hide"
                previewButton.BackgroundColor3 = THEME.GRAY
                previewActive = true
            end
        else
            rigInfoLabel.Text = "Select rig and save positions!"
        end
    end)
    
    -- Play button
    playButton.MouseButton1Click:Connect(function()
        if selectedRig and checkRigExists(selectedRig, rigInfoLabel, bodyPartsList, bodyPartsLayout) and (#savedPositions > 0 or next(bodyPartPositions)) and not isPlaying then
            playButton.Text = "Playing"
            playButton.BackgroundColor3 = THEME.GRAY
            rigInfoLabel.Text = "Playing animation once..."
            
            spawn(function()
                playAnimation(selectedRig)
                playButton.Text = "Play"
                playButton.BackgroundColor3 = THEME.GREEN
                if selectedRig then
                    rigInfoLabel.Text = "Animation completed"
                end
            end)
        end
    end)
    
    -- Loop button
    loopButton.MouseButton1Click:Connect(function()
        if selectedRig and checkRigExists(selectedRig, rigInfoLabel, bodyPartsList, bodyPartsLayout) and (#savedPositions > 0 or next(bodyPartPositions)) then
            if not isLooping then
                loopButton.Text = "Looping"
                loopButton.BackgroundColor3 = THEME.GRAY
                rigInfoLabel.Text = "Auto-loop animation started!"
                
                spawn(function()
                    playAnimationLoop(selectedRig)
                end)
            else
                isLooping = false
                isPlaying = false
                if loopConnection then
                    loopConnection:Disconnect()
                    loopConnection = nil
                end
                loopButton.Text = "Loop"
                loopButton.BackgroundColor3 = THEME.BLUE
                rigInfoLabel.Text = "Animation loop stopped"
            end
        else
            rigInfoLabel.Text = "Select rig and save keyframes first!"
        end
    end)
    
    -- Stop button
    stopButton.MouseButton1Click:Connect(function()
        isPlaying = false
        isLooping = false
        if loopConnection then
            loopConnection:Disconnect()
            loopConnection = nil
        end
        playButton.Text = "Play"
        playButton.BackgroundColor3 = THEME.GREEN
        loopButton.Text = "Loop"
        loopButton.BackgroundColor3 = THEME.BLUE
        rigInfoLabel.Text = "All animations stopped"
        
        stopButton.BackgroundColor3 = THEME.WHITE
        stopButton.TextColor3 = THEME.BLACK
        spawn(function()
            wait(0.3)
            stopButton.BackgroundColor3 = THEME.RED
            stopButton.TextColor3 = THEME.WHITE
        end)
    end)
    
    -- Reset button
    resetButton.MouseButton1Click:Connect(function()
        if selectedRig and checkRigExists(selectedRig, rigInfoLabel, bodyPartsList, bodyPartsLayout) and (#savedPositions > 0 or next(bodyPartPositions)) then
            resetToInitial(selectedRig)
            resetButton.BackgroundColor3 = THEME.WHITE
            resetButton.TextColor3 = THEME.BLACK
            spawn(function()
                wait(0.3)
                resetButton.BackgroundColor3 = THEME.GRAY
                resetButton.TextColor3 = THEME.WHITE
            end)
        end
    end)
    
    -- Clear All button
    clearAllButton.MouseButton1Click:Connect(function()
        savedPositions = {}
        bodyPartPositions = {}
        currentKeyframe = 0
        editingKeyframe = nil
        startFromKeyframe = 1
        startFromInput.Text = "1"
        
        isLooping = false
        isPlaying = false
        if loopConnection then
            loopConnection:Disconnect()
            loopConnection = nil
        end
        playButton.Text = "Play"
        playButton.BackgroundColor3 = THEME.GREEN
        loopButton.Text = "Loop"
        loopButton.BackgroundColor3 = THEME.BLUE
        
        updateBodyPartsList(bodyPartsList, bodyPartsLayout)
        clearKeyframePreview()
        previewButton.Text = "Preview"
        previewButton.BackgroundColor3 = THEME.PURPLE
        previewActive = false
        
        saveDataPersistent() -- Auto-save after clearing
        rigInfoLabel.Text = "All keyframes cleared!"
    end)
    
    -- Delete Selected button
    deleteSelectedButton.MouseButton1Click:Connect(function()
        if editingKeyframe then
            local deleted = deleteSpecificKeyframe()
            if deleted then
                updateBodyPartsList(bodyPartsList, bodyPartsLayout)
                clearKeyframePreview()
                rigInfoLabel.Text = "Selected keyframe deleted!"
            else
                rigInfoLabel.Text = "Failed to delete keyframe!"
            end
        else
            rigInfoLabel.Text = "Select a keyframe first!"
        end
    end)
    
    -- Close button
    closeButton.MouseButton1Click:Connect(function()
        isLooping = false
        isPlaying = false
        if loopConnection then
            loopConnection:Disconnect()
            loopConnection = nil
        end
        
        removeHighlight()
        clearKeyframePreview()
        if rigCheckConnection then
            rigCheckConnection:Disconnect()
        end
        
        -- Final save before closing
        saveDataPersistent()
        gui:Destroy()
    end)
    
    -- Auto-save every 30 seconds
    spawn(function()
        while gui.Parent do
            wait(30)
            saveDataPersistent()
        end
    end)
    
    updateBodyPartsList(bodyPartsList, bodyPartsLayout)
    
    print("Moon Animator X Enhanced loaded successfully!")
    print("New features:")
    print("- Both RIG and PLAYER modes use velocity checking (not TweenService)")
    print("- RIG Static: Low velocity/Anchored detection")
    print("- RIG Walking: High velocity/Moving detection")
    print("- PLAYER Static: Low velocity detection")
    print("- PLAYER Walking: High velocity detection")
    print("- Persistent data saves automatically")
    print("- All texts in English")
    print("- Auto-save every 30 seconds and on actions")
    
    if #exportedScripts > 0 then
        print("Loaded " .. #exportedScripts .. " saved scripts")
    end
    if #savedPositions > 0 or next(bodyPartPositions) then
        print("Loaded previous animation data")
    end
end

setupSystem()
