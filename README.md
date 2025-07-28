-- Create ScreenGui  
local screenGui = Instance.new("ScreenGui")  
screenGui.Parent = game.Players.LocalPlayer:WaitForChild("PlayerGui")  
screenGui.ResetOnSpawn = false  
  
-- Create Frame  
local frame = Instance.new("Frame")  
frame.Parent = screenGui  
frame.BackgroundColor3 = Color3.fromRGB(255, 255, 255)  
frame.Size = UDim2.new(0, 220, 0, 200)  
frame.Position = UDim2.new(0.5, -110, 0.5, -100)  
frame.Active = true  
frame.Draggable = true  
  
-- Create Buttons  
local recordButton = Instance.new("TextButton")  
recordButton.Parent = frame  
recordButton.BackgroundColor3 = Color3.fromRGB(255, 0, 0)  
recordButton.Size = UDim2.new(0, 60, 0, 30)  
recordButton.Position = UDim2.new(0, 10, 0, 20)  
recordButton.Text = "Record"  
recordButton.TextScaled = true  
  
local stopRecordButton = Instance.new("TextButton")  
stopRecordButton.Parent = frame  
stopRecordButton.BackgroundColor3 = Color3.fromRGB(255, 165, 0)  
stopRecordButton.Size = UDim2.new(0, 60, 0, 30)  
stopRecordButton.Position = UDim2.new(0, 80, 0, 20)  
stopRecordButton.Text = "Stop Record"  
stopRecordButton.TextScaled = true  
  
local stopReplayButton = Instance.new("TextButton")  
stopReplayButton.Parent = frame  
stopReplayButton.BackgroundColor3 = Color3.fromRGB(255, 100, 100)  
stopReplayButton.Size = UDim2.new(0, 60, 0, 30)  
stopReplayButton.Position = UDim2.new(0, 150, 0, 20)  
stopReplayButton.Text = "Stop Replay"  
stopReplayButton.TextScaled = true  
  
local destroyButton = Instance.new("TextButton")  
destroyButton.Parent = frame  
destroyButton.BackgroundColor3 = Color3.fromRGB(255, 255, 255)  
destroyButton.Size = UDim2.new(0, 90, 0, 30)  
destroyButton.Position = UDim2.new(0, 10, 0, 60)  
destroyButton.Text = "Destroy"  
destroyButton.TextScaled = true  
  
local deleteButton = Instance.new("TextButton")  
deleteButton.Parent = frame  
deleteButton.BackgroundColor3 = Color3.fromRGB(200, 100, 100)  
deleteButton.Size = UDim2.new(0, 90, 0, 30)  
deleteButton.Position = UDim2.new(0, 120, 0, 60)  
deleteButton.Text = "Delete"  
deleteButton.TextScaled = true  
  
-- Status Indicator  
local statusLabel = Instance.new("TextLabel")  
statusLabel.Parent = frame  
statusLabel.BackgroundColor3 = Color3.fromRGB(255, 255, 255)  
statusLabel.Size = UDim2.new(0, 220, 0, 30)  
statusLabel.Position = UDim2.new(0, 0, 0, -30)  
statusLabel.Text = "Status: Idle"  
statusLabel.TextColor3 = Color3.fromRGB(0, 0, 0)  
statusLabel.TextScaled = true  
  
-- Create Scrollable List  
local scrollFrame = Instance.new("ScrollingFrame")  
scrollFrame.Parent = frame  
scrollFrame.Size = UDim2.new(0, 200, 0, 70)  
scrollFrame.Position = UDim2.new(0, 10, 0, 100)  
scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0) -- Dynamically adjusted  
scrollFrame.ScrollBarThickness = 8  
  
local uiListLayout = Instance.new("UIListLayout")  
uiListLayout.Parent = scrollFrame  
uiListLayout.Padding = UDim.new(0, 5)  
  
-- Variables for recording and platforms  
local recording = false  
local replaying = false  
local player = game.Players.LocalPlayer  
local character = player.Character or player.CharacterAdded:Wait()  
local humanoid = character:WaitForChild("Humanoid") -- Get the Humanoid  
local platforms = {}  
local yellowPlatforms = {}  
local platformData = {}  
local platformCounter = 0  
local lastPosition = nil  
local replayThread  
local yellowToRedMapping = {} -- Mapping of yellow platforms to red platforms  
  
-- Add these variables at the top with other declarations  
local currentReplayThread = nil  
local shouldStopReplay = false  
local currentPlatformIndex = 0  
local totalPlatformsToPlay = 0  
  
-- Pathfinding service  
local PathfindingService = game:GetService("PathfindingService")  
local HttpService = game:GetService("HttpService")  
local RunService = game:GetService("RunService")  
  
-- Helper Functions  
local function isCharacterMoving()  
    local currentPosition = character.PrimaryPart.Position  
    if lastPosition then  
        local distance = (currentPosition - lastPosition).magnitude  
        lastPosition = currentPosition  
        return distance > 0.05  
    end  
    lastPosition = currentPosition  
    return false  
end  
  
local function cleanupPlatform(platform)  
    for i, p in ipairs(platforms) do  
        if p == platform then  
            table.remove(platforms, i)  
            platformData[platform] = nil  
            break  
        end  
    end  
end  
  
local function addPlatformToScrollFrame(platformName)  
    local button = Instance.new("TextButton")  
    button.Parent = scrollFrame  
    button.Size = UDim2.new(1, -10, 0, 25)  
    button.Text = platformName  
    button.TextScaled = true  
    button.BackgroundColor3 = Color3.fromRGB(150, 150, 255)  
  
    local playButton = Instance.new("TextButton")  
    playButton.Parent = button  
    playButton.Size = UDim2.new(0, 50, 1, 0)  
    playButton.Position = UDim2.new(1, -40, 0, 0)  
    playButton.Text = "Play"  
    playButton.TextScaled = true  
    playButton.BackgroundColor3 = Color3.fromRGB(0, 255, 0)  
  
    playButton.MouseButton1Click:Connect(function()  
        if replaying then return end  
  
        -- Stop any existing replay first  
        if currentReplayThread then  
            shouldStopReplay = true  
            task.wait() -- Allow cancellation to propagate  
        end  
  
        -- Reset control flags  
        replaying = true  
        shouldStopReplay = false  
  
        -- Extract platform number safely  
        local platformNumber = tonumber(button.Text:match("%d+"))  
        if not platformNumber or platformNumber < 1 or platformNumber > #platforms then  
            statusLabel.Text = "Status: Invalid platform index"  
            statusLabel.TextColor3 = Color3.fromRGB(255, 0, 0)  
            return  
        end  
  
        local platform = platforms[platformNumber]  
  
        -- Update initial status  
        statusLabel.Text = string.format("Starting playback from Platform %d", platformNumber)  
        statusLabel.TextColor3 = Color3.fromRGB(0, 0, 255)  
  
        -- Walk to the platform before replaying (Pathfinding Integrated)  
        local function walkToPlatform(destination)  
            local humanoid = character:WaitForChild("Humanoid")  
            local rootPart = character:WaitForChild("HumanoidRootPart")  
  
            -- Ensure the Humanoid is alive and can move  
            if not humanoid or humanoid.Health <= 0 then  
                return  
            end  
  
            local path = PathfindingService:CreatePath()  
            path:ComputeAsync(rootPart.Position, destination)  
  
            if path.Status == Enum.PathStatus.Success then  
                local waypoints = path:GetWaypoints()  
                for _, waypoint in ipairs(waypoints) do  
                    if shouldStopReplay then break end  
                    humanoid:MoveTo(waypoint.Position)  
                    if waypoint.Action == Enum.PathWaypointAction.Jump then  
                        humanoid.Jump = true  
                    end  
                    humanoid.MoveToFinished:Wait()  
                end  
            else  
                -- Fallback: Directly move to the destination if pathfinding fails  
                humanoid:MoveTo(destination)  
                humanoid.MoveToFinished:Wait()  
            end  
        end  
  
        -- Replay logic for sequential platforms (Improved)  
        local function replayPlatforms(startIndex)  
            totalPlatformsToPlay = #platforms -- Total platforms  
            currentPlatformIndex = startIndex -- Starting point  
  
            for i = startIndex, #platforms do  
                if shouldStopReplay then break end  
                  
                -- Update status with absolute position  
                statusLabel.Text = string.format("Playing from Platform %d/%d", currentPlatformIndex, totalPlatformsToPlay)  
                statusLabel.TextColor3 = Color3.fromRGB(0, 0, 255)  
                  
                local platform = platforms[i]  
                -- Walk to platform using pathfinding  
                walkToPlatform(platform.Position + Vector3.new(0, 3, 0))  
                local movements = platformData[platform]  
                if movements then  
                    --print("Movement data type check:", typeof(movements[1].position), typeof(movements[1].orientation)) -- Debug print  
                    for j = 1, #movements - 1 do  
                        if shouldStopReplay then break end  
  
                        local startMovement = movements[j]  
                        local endMovement = movements[j + 1]  
                        endMovement.isJumping = startMovement.isJumping  
  
                        local startTime = tick()  
  
                        -- Calculate duration based on distance and a speed factor  
                        local distance = (endMovement.position - startMovement.position).magnitude  
                        local speedFactor = 0.01  -- Adjust this for desired replay speed (higher = faster)  
                        local duration = distance * speedFactor  
  
                        -- Minimum duration to prevent division by zero or extremely fast replays  
                        duration = math.max(duration, 0.01)  
  
                        local endTime = startTime + duration  
  
                        while tick() < endTime do  
                            if shouldStopReplay then break end  
                            local alpha = (tick() - startTime) / duration  
                            alpha = math.min(alpha, 1)  
  
                            local interpolatedPosition = startMovement.position:Lerp(endMovement.position, alpha)  
                              
                            -- Fixed orientation interpolation  
                            local interpolatedOrientation = CFrame.fromEulerAnglesXYZ(  
                                math.rad(startMovement.orientation.X),   
                                math.rad(startMovement.orientation.Y),   
                                math.rad(startMovement.orientation.Z)  
                            ):Lerp(  
                                CFrame.fromEulerAnglesXYZ(  
                                    math.rad(endMovement.orientation.X),  
                                    math.rad(endMovement.orientation.Y),  
                                    math.rad(endMovement.orientation.Z)  
                                ),   
                                alpha  
                            )  
  
                            character:SetPrimaryPartCFrame(  
                                CFrame.new(interpolatedPosition) * interpolatedOrientation  
                            )  
  
                            if endMovement.isJumping then  
                                humanoid.Jump = true  
                            end  
  
                            RunService.Heartbeat:Wait()  
                        end  
                    end  
                end  
                  
                currentPlatformIndex += 1 -- Move to next platform index  
            end  
            -- Final status update  
            if not shouldStopReplay then  
                statusLabel.Text = "Status: Completed all platforms"  
                statusLabel.TextColor3 = Color3.fromRGB(0, 255, 0)  
                task.wait(1)  
            end  
              
            statusLabel.Text = "Status: Idle"  
            statusLabel.TextColor3 = Color3.fromRGB(0, 0, 0)  
            replaying = false  
        end  
  
        -- Start new replay in a tracked thread  
        currentReplayThread = task.spawn(function()  
            replayPlatforms(platformNumber)  
            currentReplayThread = nil  
        end)  
    end)  
end  
  
-- Modify the stopReplayButton click handler  
stopReplayButton.MouseButton1Click:Connect(function()  
    if replaying then  
        shouldStopReplay = true  
        replaying = false  
        statusLabel.Text = string.format("Stopped at Platform %d/%d", currentPlatformIndex-1, totalPlatformsToPlay)  
        statusLabel.TextColor3 = Color3.fromRGB(255, 165, 0)  
        task.wait(2)  
        statusLabel.Text = "Status: Idle"  
        -- Cancel any ongoing movement  
        if character:FindFirstChild("Humanoid") then  
            character.Humanoid:MoveTo(character.HumanoidRootPart.Position)  
        end  
    end  
end)  
  
-- Function to add text label to a platform  
local function addTextLabelToPlatform(platform, platformNumber)  
    local billboardGui = Instance.new("BillboardGui")  
    billboardGui.Size = UDim2.new(1, 0, 0.5, 0)  
    billboardGui.StudsOffset = Vector3.new(0, 3, 0)  
    billboardGui.AlwaysOnTop = true  
  
    local textLabel = Instance.new("TextLabel")  
    textLabel.Size = UDim2.new(1, 0, 1, 0)  
    textLabel.BackgroundTransparency = 1  
    textLabel.TextColor3 = Color3.fromRGB(255, 255, 255)  
    textLabel.TextScaled = true  
    textLabel.Text = tostring(platformNumber)  
    textLabel.Parent = billboardGui  
  
    billboardGui.Parent = platform  
end  
  
-- Function to serialize platform data to JSON  
local function serializePlatformData()  
    local data = {  
        redPlatforms = {},  
        yellowPlatforms = {},  
        mappings = {}  
    }  
  
    -- Save red platforms  
    for i, platform in ipairs(platforms) do  
        local movementsData = {}  
        for _, movement in ipairs(platformData[platform]) do  
            table.insert(movementsData, {  
                position = {X = movement.position.X, Y = movement.position.Y, Z = movement.position.Z},  
                orientation = {X = movement.orientation.X, Y = movement.orientation.Y, Z = movement.orientation.Z},  
                isJumping = movement.isJumping  
            })  
        end  
        table.insert(data.redPlatforms, {  
            position = {X = platform.Position.X, Y = platform.Position.Y, Z = platform.Position.Z},  
            movements = movementsData  
        })  
    end  
  
    -- Save yellow platforms and their mappings  
    for i, yellowPlatform in ipairs(yellowPlatforms) do  
        table.insert(data.yellowPlatforms, {  
            position = {X = yellowPlatform.Position.X, Y = yellowPlatform.Position.Y, Z = yellowPlatform.Position.Z}  
        })  
  
        -- Store mapping index (map yellow platform to the corresponding red platform index)  
        if yellowToRedMapping[yellowPlatform] then  
            local redIndex = table.find(platforms, yellowToRedMapping[yellowPlatform])  
            table.insert(data.mappings, redIndex)  
        end  
    end  
  
    local rawData = HttpService:JSONEncode(data)  
    return rawData  
end  
  
-- Function to deserialize platform data from JSON  
local function deserializePlatformData(jsonData)  
    local success, data = pcall(function()  
        return HttpService:JSONDecode(jsonData)  
    end)  
  
    if not success then  
        warn("Failed to decode JSON data:", data)  
        statusLabel.Text = "Status: Invalid JSON data"  
        statusLabel.TextColor3 = Color3.fromRGB(255, 0, 0)  
        return  
    end  
  
    if not data or type(data) ~= "table" then  
        warn("Invalid data format after decoding JSON")  
        statusLabel.Text = "Status: Invalid data format"  
        statusLabel.TextColor3 = Color3.fromRGB(255, 0, 0)  
        return  
    end  
  
    -- Clear existing data  
    platforms = {}  
    yellowPlatforms = {}  
    yellowToRedMapping = {}  
    platformCounter = 0  
  
    -- Clear scroll frame and recreate UIListLayout  
    scrollFrame:ClearAllChildren()  
    local uiListLayout = Instance.new("UIListLayout")  
    uiListLayout.Parent = scrollFrame  
    uiListLayout.Padding = UDim.new(0, 5)  
  
    -- Load red platforms (updated movement processing)  
    if data.redPlatforms then  
        for _, platformInfo in ipairs(data.redPlatforms) do  
            local platform = Instance.new("Part")  
            platform.Size = Vector3.new(5, 1, 5)  
            platform.Position = Vector3.new(platformInfo.position.X, platformInfo.position.Y, platformInfo.position.Z)  
            platform.Anchored = true  
            platform.BrickColor = BrickColor.Red()  
            platform.CanCollide = false  
            platform.Parent = workspace  
  
            -- Convert serialized movement data back to proper Vector3 formats  
            local restoredMovements = {}  
            if platformInfo.movements then  
                for _, movement in ipairs(platformInfo.movements) do  
                    table.insert(restoredMovements, {  
                        position = Vector3.new(movement.position.X, movement.position.Y, movement.position.Z),  
                        orientation = Vector3.new(movement.orientation.X, movement.orientation.Y, movement.orientation.Z),  
                        isJumping = movement.isJumping  
                    })  
                end  
            end  
  
            platformData[platform] = restoredMovements  -- Store converted movements  
  
            addTextLabelToPlatform(platform, #platforms + 1)  
            table.insert(platforms, platform)  
  
            platformCounter += 1  
            addPlatformToScrollFrame("Platform " .. platformCounter)  
        end  
    end  
  
  
    -- Load yellow platforms and mappings  
    if data.yellowPlatforms then  
        for i, yellowInfo in ipairs(data.yellowPlatforms) do  
            local yellowPlatform = Instance.new("Part")  
            yellowPlatform.Size = Vector3.new(5, 1, 5)  
            yellowPlatform.Position = Vector3.new(yellowInfo.position.X, yellowInfo.position.Y, yellowInfo.position.Z)  
            yellowPlatform.Anchored = true  
            yellowPlatform.BrickColor = BrickColor.Yellow()  
            yellowPlatform.CanCollide = false  
            yellowPlatform.Parent = workspace  
  
            if data.mappings and data.mappings[i] then  
                addTextLabelToPlatform(yellowPlatform, data.mappings[i])  
                -- Rebuild yellowToRedMapping using loaded platforms  
                if platforms[data.mappings[i]] then  
                    yellowToRedMapping[yellowPlatform] = platforms[data.mappings[i]]  
                else  
                     warn("Mapping points to non-existent red platform index:", data.mappings[i])  
                end  
            end  
             table.insert(yellowPlatforms, yellowPlatform)  
        end  
    end  
  
  
    scrollFrame.CanvasSize = UDim2.new(0, 0, 0, #platforms * 30)  
    statusLabel.Text = "Status: Platform data loaded"  
    statusLabel.TextColor3 = Color3.fromRGB(0, 255, 0)  
end  
  
-- Handle character respawn or reset  
player.CharacterAdded:Connect(function(newCharacter)  
    character = newCharacter  
    humanoid = newCharacter:WaitForChild("Humanoid") -- Get Humanoid for new character  
    lastPosition = nil  
    statusLabel.Text = "Status: Idle"  
    statusLabel.TextColor3 = Color3.fromRGB(0, 0, 0)  
  
    if recording then  
        platformCounter = 0  
        platforms = {}  
        yellowPlatforms = {}  
        platformData = {}  
        yellowToRedMapping = {}  
        scrollFrame:ClearAllChildren()  
        local uiListLayout = Instance.new("UIListLayout")  
        uiListLayout.Parent = scrollFrame  
        uiListLayout.Padding = UDim.new(0, 5)  
        scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)  
        shouldStopReplay = true  
        replaying = false  
        if currentReplayThread then  
            task.cancel(currentReplayThread) -- Cancel the replay thread  
            currentReplayThread = nil  
        end  
    end  
end)  
  
-- Function to calculate path  
local function calculatePath(start, goal)  
    local path = PathfindingService:CreatePath()  
    path:ComputeAsync(start, goal)  
   
