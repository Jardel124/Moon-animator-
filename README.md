
local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local TweenService = game:GetService("TweenService")

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

-- Theme colors
local THEME = {
    ORANGE = Color3.fromRGB(255, 140, 0),
    BLACK = Color3.fromRGB(0, 0, 0),
    WHITE = Color3.fromRGB(255, 255, 255),
    GRAY = Color3.fromRGB(128, 128, 128),
    DARK_GRAY = Color3.fromRGB(64, 64, 64),
    LIGHT_GRAY = Color3.fromRGB(192, 192, 192)
}

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

-- Export clean animation script with relative positioning
local function exportAnimationScript()
    if #savedPositions == 0 and not next(bodyPartPositions) then
        return "-- No animation data to export"
    end
    
    local code = [[-- Exported animation from Moon Animator X (Using Relative Positioning)
-- This is a LocalScript if you want it to run on client
-- This is a Script if you want it to run on server
local TweenService = game:GetService('TweenService')
local rig = script.Parent -- Change this to your rig path

local function playAnimation()
    local rootPart = rig:FindFirstChild("HumanoidRootPart")
    if not rootPart then
        warn("No HumanoidRootPart found for animation")
        return
    end
    
]]
    
    -- Add keyframes with relative positioning
    for i, savedData in ipairs(savedPositions) do
        code = code .. "    -- Keyframe " .. i .. "\n"
        code = code .. "    local tweenInfo" .. i .. " = TweenInfo.new(" .. (savedData.duration or keyframeDuration) .. ", Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)\n"
        code = code .. "    local tweens" .. i .. " = {}\n"
        
        local partIndex = 1
        for partName, data in pairs(savedData.parts) do
            -- Convert relative CFrame back to code format
            local x, y, z = data.CFrame.Position.X, data.CFrame.Position.Y, data.CFrame.Position.Z
            local rx, ry, rz = data.CFrame:toEulerAnglesXYZ()
            
            code = code .. "    local part" .. partIndex .. " = rig:FindFirstChild(\"" .. partName .. "\")\n"
            code = code .. "    if part" .. partIndex .. " then\n"
            
            if partName == "HumanoidRootPart" then
                code = code .. "        local targetCFrame" .. partIndex .. " = rootPart.CFrame * CFrame.new(" .. string.format("%.3f, %.3f, %.3f", x, y, z) .. ") * CFrame.Angles(" .. string.format("%.3f, %.3f, %.3f", rx, ry, rz) .. ")\n"
            else
                code = code .. "        local targetCFrame" .. partIndex .. " = rootPart.CFrame * CFrame.new(" .. string.format("%.3f, %.3f, %.3f", x, y, z) .. ") * CFrame.Angles(" .. string.format("%.3f, %.3f, %.3f", rx, ry, rz) .. ")\n"
            end
            
            code = code .. "        local tween" .. partIndex .. " = TweenService:Create(part" .. partIndex .. ", tweenInfo" .. i .. ", {\n"
            code = code .. "            CFrame = targetCFrame" .. partIndex .. ",\n"
            code = code .. string.format("            Size = Vector3.new(%.3f, %.3f, %.3f)\n", data.Size.X, data.Size.Y, data.Size.Z)
            code = code .. "        })\n"
            code = code .. "        table.insert(tweens" .. i .. ", tween" .. partIndex .. ")\n"
            code = code .. "    end\n"
            
            partIndex = partIndex + 1
        end
        
        code = code .. "    \n"
        code = code .. "    for _, tween in pairs(tweens" .. i .. ") do\n"
        code = code .. "        tween:Play()\n"
        code = code .. "    end\n"
        code = code .. "    if #tweens" .. i .. " > 0 then\n"
        code = code .. "        tweens" .. i .. "[1].Completed:Wait()\n"
        code = code .. "    end\n\n"
    end
    
    -- Add individual parts with relative positioning
    for partName, positions in pairs(bodyPartPositions) do
        code = code .. "    -- " .. partName .. " animation (relative)\n"
        code = code .. "    local " .. partName:gsub(" ", ""):gsub("[^%w]", "") .. "Part = rig:FindFirstChild(\"" .. partName .. "\")\n"
        code = code .. "    if " .. partName:gsub(" ", ""):gsub("[^%w]", "") .. "Part then\n"
        
        for j, data in ipairs(positions) do
            local x, y, z = data.CFrame.Position.X, data.CFrame.Position.Y, data.CFrame.Position.Z
            local rx, ry, rz = data.CFrame:toEulerAnglesXYZ()
            
            code = code .. "        local partTweenInfo = TweenInfo.new(" .. data.duration .. ", Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)\n"
            
            if partName == "HumanoidRootPart" then
                code = code .. "        local targetCFrame = rootPart.CFrame * CFrame.new(" .. string.format("%.3f, %.3f, %.3f", x, y, z) .. ") * CFrame.Angles(" .. string.format("%.3f, %.3f, %.3f", rx, ry, rz) .. ")\n"
            else
                code = code .. "        local targetCFrame = rootPart.CFrame * CFrame.new(" .. string.format("%.3f, %.3f, %.3f", x, y, z) .. ") * CFrame.Angles(" .. string.format("%.3f, %.3f, %.3f", rx, ry, rz) .. ")\n"
            end
            
            code = code .. "        local partTween = TweenService:Create(" .. partName:gsub(" ", ""):gsub("[^%w]", "") .. "Part, partTweenInfo, {\n"
            code = code .. "            CFrame = targetCFrame,\n"
            code = code .. string.format("            Size = Vector3.new(%.3f, %.3f, %.3f)\n", data.Size.X, data.Size.Y, data.Size.Z)
            code = code .. "        })\n"
            code = code .. "        partTween:Play()\n"
            code = code .. "        partTween.Completed:Wait()\n"
            
            if j < #positions then
                code = code .. "        wait(0.1)\n"
            end
        end
        
        code = code .. "    end\n"
    end
    
    code = code .. [[end

-- Call this to play animation once
playAnimation()

-- For infinite loop, uncomment below:
-- while true do
--     playAnimation()
--     wait(0.5)
-- end
]]
    
    return code
end

-- Save position with relative positioning
local function savePosition(rig, specificPart, updateExisting)
    if not rig or not rig:FindFirstChild("Humanoid") then 
        return 
    end
    
    -- Get root part for relative positioning
    local rootPart = rig:FindFirstChild("HumanoidRootPart")
    if not rootPart then
        print("Warning: No HumanoidRootPart found for relative positioning")
        return
    end
    
    if specificPart then
        if not bodyPartPositions[specificPart.Name] then
            bodyPartPositions[specificPart.Name] = {}
        end
        
        -- Calculate relative CFrame to root
        local relativeCFrame = rootPart.CFrame:Inverse() * specificPart.CFrame
        
        local newData = {
            CFrame = relativeCFrame, -- Store relative CFrame
            Size = specificPart.Size,
            duration = keyframeDuration,
            timestamp = tick(),
            position = specificPart.Position -- Keep for display
        }
        
        if updateExisting and editingKeyframe and editingKeyframe.type == "part" and editingKeyframe.partName == specificPart.Name then
            bodyPartPositions[specificPart.Name][editingKeyframe.index] = newData
            local index = editingKeyframe.index
            editingKeyframe = nil
            return index, specificPart.Name, true
        else
            table.insert(bodyPartPositions[specificPart.Name], newData)
            return #bodyPartPositions[specificPart.Name], specificPart.Name, false
        end
    else
        local savedData = {
            rigName = rig.Name,
            duration = keyframeDuration,
            parts = {},
            timestamp = tick(),
            rootCFrame = rootPart.CFrame -- Store root position for reference
        }
        
        -- Save all parts relative to root
        for _, part in pairs(rig:GetChildren()) do
            if part:IsA("BasePart") and part ~= rootPart then
                local relativeCFrame = rootPart.CFrame:Inverse() * part.CFrame
                savedData.parts[part.Name] = {
                    CFrame = relativeCFrame, -- Relative to root
                    Size = part.Size,
                    position = part.Position -- Keep for display
                }
            elseif part == rootPart then
                -- Root part uses its own CFrame as reference
                savedData.parts[part.Name] = {
                    CFrame = CFrame.new(0, 0, 0), -- Identity for root
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
            return index, "Full Body", true
        else
            table.insert(savedPositions, savedData)
            currentKeyframe = #savedPositions
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
        
        savedPositions = {}
        bodyPartPositions = {}
        currentKeyframe = 0
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
    return true
end

-- Reset to initial
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
    
    for partName, positions in pairs(bodyPartPositions) do
        local originalPart = rig:FindFirstChild(partName)
        
        if originalPart then
            for i, data in ipairs(positions) do
                local ghostPart = originalPart:Clone()
                ghostPart.Name = "GhostPart_Part_" .. partName .. "_" .. tostring(i)
                ghostPart.Parent = workspace
                ghostPart.CFrame = data.CFrame
                ghostPart.Size = data.Size
                ghostPart.Transparency = 0.8
                ghostPart.CanCollide = false
                ghostPart.Anchored = true
                
                for _, child in pairs(ghostPart:GetChildren()) do
                    if not child:IsA("SpecialMesh") and not child:IsA("Decal") then
                        child:Destroy()
                    end
                end
                
                local previewHighlight = Instance.new("Highlight")
                previewHighlight.Parent = ghostPart
                previewHighlight.FillColor = THEME.ORANGE
                previewHighlight.OutlineColor = THEME.WHITE
                previewHighlight.FillTransparency = 0.6
                previewHighlight.OutlineTransparency = 0.2
                
                table.insert(previewConnections, {
                    part = ghostPart,
                    highlight = previewHighlight
                })
            end
        end
    end
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
                    -- Keep root at current position, just apply saved rotation/offset
                    targetCFrame = rootPart.CFrame * data.CFrame
                else
                    -- Apply relative CFrame to current root position
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

-- Play loop animation with relative positioning
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

-- Create GUI
local function createMainGui()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "MoonAnimatorX"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = playerGui
    
    local mainFrame = Instance.new("Frame")
    mainFrame.Size = UDim2.new(0, 900, 0, 200)
    mainFrame.Position = UDim2.new(0.5, -450, 0, 10)
    mainFrame.BackgroundColor3 = THEME.BLACK
    mainFrame.BorderSizePixel = 2
    mainFrame.BorderColor3 = THEME.ORANGE
    mainFrame.Active = true
    mainFrame.Draggable = true
    mainFrame.Parent = screenGui
    
    local titleBar = Instance.new("Frame")
    titleBar.Size = UDim2.new(1, 0, 0, 25)
    titleBar.BackgroundColor3 = THEME.ORANGE
    titleBar.BorderSizePixel = 0
    titleBar.Parent = mainFrame
    
    local titleLabel = Instance.new("TextLabel")
    titleLabel.Size = UDim2.new(1, -30, 1, 0)
    titleLabel.Position = UDim2.new(0, 5, 0, 0)
    titleLabel.BackgroundTransparency = 1
    titleLabel.Text = "Moon Animator X - Auto Loop"
    titleLabel.TextColor3 = THEME.WHITE
    titleLabel.TextScaled = true
    titleLabel.Font = Enum.Font.GothamBold
    titleLabel.TextXAlignment = Enum.TextXAlignment.Left
    titleLabel.Parent = titleBar
    
    local closeButton = Instance.new("TextButton")
    closeButton.Size = UDim2.new(0, 25, 0, 25)
    closeButton.Position = UDim2.new(1, -25, 0, 0)
    closeButton.BackgroundColor3 = THEME.GRAY
    closeButton.Text = "X"
    closeButton.TextColor3 = THEME.WHITE
    closeButton.TextScaled = true
    closeButton.Font = Enum.Font.GothamBold
    closeButton.BorderSizePixel = 0
    closeButton.Parent = titleBar
    
    local mainButtonsFrame = Instance.new("Frame")
    mainButtonsFrame.Size = UDim2.new(1, -10, 0, 40)
    mainButtonsFrame.Position = UDim2.new(0, 5, 0, 30)
    mainButtonsFrame.BackgroundColor3 = THEME.DARK_GRAY
    mainButtonsFrame.BorderSizePixel = 1
    mainButtonsFrame.BorderColor3 = THEME.GRAY
    mainButtonsFrame.Parent = mainFrame
    
    -- Create buttons
    local playButton = Instance.new("TextButton")
    playButton.Size = UDim2.new(0, 60, 0, 30)
    playButton.Position = UDim2.new(0, 5, 0, 5)
    playButton.BackgroundColor3 = THEME.ORANGE
    playButton.Text = "Play"
    playButton.TextColor3 = THEME.WHITE
    playButton.TextScaled = true
    playButton.Font = Enum.Font.GothamBold
    playButton.BorderSizePixel = 0
    playButton.Parent = mainButtonsFrame
    
    local loopButton = Instance.new("TextButton")
    loopButton.Size = UDim2.new(0, 60, 0, 30)
    loopButton.Position = UDim2.new(0, 70, 0, 5)
    loopButton.BackgroundColor3 = THEME.ORANGE
    loopButton.Text = "Loop"
    loopButton.TextColor3 = THEME.WHITE
    loopButton.TextScaled = true
    loopButton.Font = Enum.Font.GothamBold
    loopButton.BorderSizePixel = 0
    loopButton.Parent = mainButtonsFrame
    
    local stopButton = Instance.new("TextButton")
    stopButton.Size = UDim2.new(0, 60, 0, 30)
    stopButton.Position = UDim2.new(0, 135, 0, 5)
    stopButton.BackgroundColor3 = THEME.ORANGE
    stopButton.Text = "Stop"
    stopButton.TextColor3 = THEME.WHITE
    stopButton.TextScaled = true
    stopButton.Font = Enum.Font.GothamBold
    stopButton.BorderSizePixel = 0
    stopButton.Parent = mainButtonsFrame
    
    local resetButton = Instance.new("TextButton")
    resetButton.Size = UDim2.new(0, 60, 0, 30)
    resetButton.Position = UDim2.new(0, 200, 0, 5)
    resetButton.BackgroundColor3 = THEME.ORANGE
    resetButton.Text = "Reset"
    resetButton.TextColor3 = THEME.WHITE
    resetButton.TextScaled = true
    resetButton.Font = Enum.Font.GothamBold
    resetButton.BorderSizePixel = 0
    resetButton.Parent = mainButtonsFrame
    
    local previewButton = Instance.new("TextButton")
    previewButton.Size = UDim2.new(0, 70, 0, 30)
    previewButton.Position = UDim2.new(0, 265, 0, 5)
    previewButton.BackgroundColor3 = THEME.ORANGE
    previewButton.Text = "Preview"
    previewButton.TextColor3 = THEME.WHITE
    previewButton.TextScaled = true
    previewButton.Font = Enum.Font.GothamBold
    previewButton.BorderSizePixel = 0
    previewButton.Parent = mainButtonsFrame
    
    local savePositionButton = Instance.new("TextButton")
    savePositionButton.Size = UDim2.new(0, 80, 0, 30)
    savePositionButton.Position = UDim2.new(0, 340, 0, 5)
    savePositionButton.BackgroundColor3 = THEME.ORANGE
    savePositionButton.Text = "Save"
    savePositionButton.TextColor3 = THEME.WHITE
    savePositionButton.TextScaled = true
    savePositionButton.Font = Enum.Font.GothamBold
    savePositionButton.BorderSizePixel = 0
    savePositionButton.Parent = mainButtonsFrame
    
    local exportButton = Instance.new("TextButton")
    exportButton.Size = UDim2.new(0, 70, 0, 30)
    exportButton.Position = UDim2.new(0, 425, 0, 5)
    exportButton.BackgroundColor3 = THEME.ORANGE
    exportButton.Text = "Export"
    exportButton.TextColor3 = THEME.WHITE
    exportButton.TextScaled = true
    exportButton.Font = Enum.Font.GothamBold
    exportButton.BorderSizePixel = 0
    exportButton.Parent = mainButtonsFrame
    
    local durationLabel = Instance.new("TextLabel")
    durationLabel.Size = UDim2.new(0, 60, 0, 15)
    durationLabel.Position = UDim2.new(0, 500, 0, 5)
    durationLabel.BackgroundTransparency = 1
    durationLabel.Text = "Duration:"
    durationLabel.TextColor3 = THEME.WHITE
    durationLabel.TextScaled = true
    durationLabel.Font = Enum.Font.Gotham
    durationLabel.Parent = mainButtonsFrame
    
    local durationInput = Instance.new("TextBox")
    durationInput.Size = UDim2.new(0, 50, 0, 20)
    durationInput.Position = UDim2.new(0, 500, 0, 15)
    durationInput.BackgroundColor3 = THEME.WHITE
    durationInput.Text = "0.5"
    durationInput.TextColor3 = THEME.BLACK
    durationInput.TextScaled = true
    durationInput.Font = Enum.Font.Gotham
    durationInput.BorderSizePixel = 1
    durationInput.BorderColor3 = THEME.GRAY
    durationInput.Parent = mainButtonsFrame
    
    local startFromLabel = Instance.new("TextLabel")
    startFromLabel.Size = UDim2.new(0, 60, 0, 15)
    startFromLabel.Position = UDim2.new(0, 560, 0, 5)
    startFromLabel.BackgroundTransparency = 1
    startFromLabel.Text = "Start From:"
    startFromLabel.TextColor3 = THEME.WHITE
    startFromLabel.TextScaled = true
    startFromLabel.Font = Enum.Font.Gotham
    startFromLabel.Parent = mainButtonsFrame
    
    local startFromInput = Instance.new("TextBox")
    startFromInput.Size = UDim2.new(0, 50, 0, 20)
    startFromInput.Position = UDim2.new(0, 560, 0, 15)
    startFromInput.BackgroundColor3 = THEME.WHITE
    startFromInput.Text = "1"
    startFromInput.TextColor3 = THEME.BLACK
    startFromInput.TextScaled = true
    startFromInput.Font = Enum.Font.Gotham
    startFromInput.BorderSizePixel = 1
    startFromInput.BorderColor3 = THEME.GRAY
    startFromInput.Parent = mainButtonsFrame
    
    local bottomFrame = Instance.new("Frame")
    bottomFrame.Size = UDim2.new(1, -10, 1, -80)
    bottomFrame.Position = UDim2.new(0, 5, 0, 75)
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
    dangerTitle.Size = UDim2.new(1, 0, 0, 20)
    dangerTitle.BackgroundColor3 = THEME.GRAY
    dangerTitle.Text = "KEYFRAME CONTROLS"
    dangerTitle.TextColor3 = THEME.WHITE
    dangerTitle.TextScaled = true
    dangerTitle.Font = Enum.Font.GothamBold
    dangerTitle.BorderSizePixel = 0
    dangerTitle.Parent = dangerFrame
    
    local clearAllButton = Instance.new("TextButton")
    clearAllButton.Size = UDim2.new(1, -10, 0, 25)
    clearAllButton.Position = UDim2.new(0, 5, 0, 25)
    clearAllButton.BackgroundColor3 = THEME.GRAY
    clearAllButton.Text = "Clear All"
    clearAllButton.TextColor3 = THEME.WHITE
    clearAllButton.TextScaled = true
    clearAllButton.Font = Enum.Font.GothamBold
    clearAllButton.BorderSizePixel = 0
    clearAllButton.Parent = dangerFrame
    
    local deleteSelectedButton = Instance.new("TextButton")
    deleteSelectedButton.Size = UDim2.new(1, -10, 0, 25)
    deleteSelectedButton.Position = UDim2.new(0, 5, 0, 55)
    deleteSelectedButton.BackgroundColor3 = THEME.GRAY
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
    infoTitle.Size = UDim2.new(1, 0, 0, 20)
    infoTitle.BackgroundColor3 = THEME.GRAY
    infoTitle.Text = "ANIMATION INFO"
    infoTitle.TextColor3 = THEME.WHITE
    infoTitle.TextScaled = true
    infoTitle.Font = Enum.Font.GothamBold
    infoTitle.BorderSizePixel = 0
    infoTitle.Parent = infoFrame
    
    local rigInfoLabel = Instance.new("TextLabel")
    rigInfoLabel.Size = UDim2.new(1, -10, 0, 15)
    rigInfoLabel.Position = UDim2.new(0, 5, 0, 25)
    rigInfoLabel.BackgroundTransparency = 1
    rigInfoLabel.Text = "No rig selected - Click on a character"
    rigInfoLabel.TextColor3 = THEME.WHITE
    rigInfoLabel.TextScaled = true
    rigInfoLabel.Font = Enum.Font.Gotham
    rigInfoLabel.TextXAlignment = Enum.TextXAlignment.Left
    rigInfoLabel.Parent = infoFrame
    
    local partInfoLabel = Instance.new("TextLabel")
    partInfoLabel.Size = UDim2.new(1, -10, 0, 15)
    partInfoLabel.Position = UDim2.new(0, 5, 0, 45)
    partInfoLabel.BackgroundTransparency = 1
    partInfoLabel.Text = "No part selected - Right click on body part"
    partInfoLabel.TextColor3 = THEME.LIGHT_GRAY
    partInfoLabel.TextScaled = true
    partInfoLabel.Font = Enum.Font.Gotham
    partInfoLabel.TextXAlignment = Enum.TextXAlignment.Left
    partInfoLabel.Parent = infoFrame
    
    local bodyPartsFrame = Instance.new("Frame")
    bodyPartsFrame.Size = UDim2.new(1, -10, 0, 35)
    bodyPartsFrame.Position = UDim2.new(0, 5, 1, -40)
    bodyPartsFrame.BackgroundColor3 = THEME.GRAY
    bodyPartsFrame.BorderSizePixel = 1
    bodyPartsFrame.BorderColor3 = THEME.ORANGE
    bodyPartsFrame.Parent = infoFrame
    
    local bodyPartsTitle = Instance.new("TextLabel")
    bodyPartsTitle.Size = UDim2.new(0, 80, 1, 0)
    bodyPartsTitle.BackgroundColor3 = THEME.ORANGE
    bodyPartsTitle.Text = "KEYFRAMES"
    bodyPartsTitle.TextColor3 = THEME.WHITE
    bodyPartsTitle.TextScaled = true
    bodyPartsTitle.Font = Enum.Font.GothamBold
    bodyPartsTitle.BorderSizePixel = 0
    bodyPartsTitle.Parent = bodyPartsFrame
    
    local bodyPartsList = Instance.new("ScrollingFrame")
    bodyPartsList.Size = UDim2.new(1, -80, 1, 0)
    bodyPartsList.Position = UDim2.new(0, 80, 0, 0)
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
    
    return screenGui, rigInfoLabel, partInfoLabel, playButton, loopButton, stopButton, resetButton, previewButton, savePositionButton, exportButton, clearAllButton, deleteSelectedButton, closeButton, mainFrame, durationInput, startFromInput, bodyPartsList, bodyPartsLayout
end

-- Main setup function
local function setupSystem()
    local gui, rigInfoLabel, partInfoLabel, playButton, loopButton, stopButton, resetButton, previewButton, savePositionButton, exportButton, clearAllButton, deleteSelectedButton, closeButton, mainFrame, durationInput, startFromInput, bodyPartsList, bodyPartsLayout = createMainGui()
    
    -- Duration input
    durationInput.FocusLost:Connect(function()
        local newDuration = tonumber(durationInput.Text)
        if newDuration and newDuration > 0 then
            keyframeDuration = newDuration
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
    
    -- Export button (COPIA PARA CLIPBOARD)
    exportButton.MouseButton1Click:Connect(function()
        if #savedPositions > 0 or next(bodyPartPositions) then
            local code = exportAnimationScript()
            
            if setclipboard then
                setclipboard(code)
                rigInfoLabel.Text = "Animation script copied to clipboard!"
            else
                print("-- EXPORTED ANIMATION SCRIPT --")
                print(code)
                rigInfoLabel.Text = "Animation script printed to console!"
            end
            
            exportButton.BackgroundColor3 = THEME.WHITE
            exportButton.TextColor3 = THEME.BLACK
            spawn(function()
                wait(0.3)
                exportButton.BackgroundColor3 = THEME.ORANGE
                exportButton.TextColor3 = THEME.WHITE
                wait(2)
                if selectedRig then
                    rigInfoLabel.Text = "Selected: " .. selectedRig.Name
                else
                    rigInfoLabel.Text = "No rig selected - Click on a character"
                end
            end)
        else
            rigInfoLabel.Text = "No animation data to export!"
            exportButton.BackgroundColor3 = THEME.GRAY
            spawn(function()
                wait(0.5)
                exportButton.BackgroundColor3 = THEME.ORANGE
            end)
        end
    end)
    
    -- Preview button
    previewButton.MouseButton1Click:Connect(function()
        if selectedRig and checkRigExists(selectedRig, rigInfoLabel, bodyPartsList, bodyPartsLayout) and (#savedPositions > 0 or next(bodyPartPositions)) then
            if previewActive then
                clearKeyframePreview()
                previewButton.Text = "Preview"
                previewButton.BackgroundColor3 = THEME.ORANGE
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
                playButton.BackgroundColor3 = THEME.ORANGE
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
                loopButton.BackgroundColor3 = THEME.ORANGE
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
        playButton.BackgroundColor3 = THEME.ORANGE
        loopButton.Text = "Loop"
        loopButton.BackgroundColor3 = THEME.ORANGE
        rigInfoLabel.Text = "All animations stopped"
        
        stopButton.BackgroundColor3 = THEME.WHITE
        stopButton.TextColor3 = THEME.BLACK
        spawn(function()
            wait(0.3)
            stopButton.BackgroundColor3 = THEME.ORANGE
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
                resetButton.BackgroundColor3 = THEME.ORANGE
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
        playButton.BackgroundColor3 = THEME.ORANGE
        loopButton.Text = "Loop"
        loopButton.BackgroundColor3 = THEME.ORANGE
        
        updateBodyPartsList(bodyPartsList, bodyPartsLayout)
        clearKeyframePreview()
        previewButton.Text = "Preview"
        previewButton.BackgroundColor3 = THEME.ORANGE
        previewActive = false
        
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
        gui:Destroy()
    end)
    
    updateBodyPartsList(bodyPartsList, bodyPartsLayout)
    
    print("Moon Animator X loaded successfully!")
end

setupSystem()