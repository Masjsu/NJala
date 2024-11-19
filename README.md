getgenv().extort = {
    ['Silent'] = {
        Normal = {
            SilentAimEnabled = true,
            HitPart = "HumanoidRootPart",
            Prediction = 0.1257724,
            AirPrediction = 0.129,
        },
        FOV = {
            SilentAimFOVSize = 150,
            SilentAimShowFOV = false,
        },
        Checks = {
            AntiGroundShots = false,
        },
        Resolver = {
            Enabled = false,
        }
    },
    ['CamLock'] = {
        Normal = {
            Enabled = false,
            Prediction = 0.168,
            Smoothness = 1,
            AimPart = "UpperTorso",
            ShakeValue = 0.86,
            AutoPred = true,
        },
        Check = {
            KOCheck = true,
            WallCheck = true,
        },
        FOVSettings = {
            FOVSize = 100,
            ShowFOV = false,
        },
        ToggleMethod = {
            CamlockTool = false,
            CamlockButton = true,
        },
        EasingStyle = Enum.EasingStyle.Quad,
    }
}




local SilentAimFOVCircle = Drawing.new("Circle")
SilentAimFOVCircle.Color = Color3.new(255, 0, 0)
SilentAimFOVCircle.Visible = extort.Silent.FOV.SilentAimShowFOV
SilentAimFOVCircle.Filled = false
SilentAimFOVCircle.Radius = extort.Silent.FOV.SilentAimFOVSize
SilentAimFOVCircle.Transparency = 100
SilentAimFOVCircle.Thickness = 2.0


local function updateFOVCirclePosition()
    local centerScreenPosition = Vector2.new(workspace.CurrentCamera.ViewportSize.X / 2, workspace.CurrentCamera.ViewportSize.Y / 2)
    SilentAimFOVCircle.Position = centerScreenPosition
end


local function getClosestPlayerToCenter()
    local centerScreenPosition = Vector2.new(workspace.CurrentCamera.ViewportSize.X / 2, workspace.CurrentCamera.ViewportSize.Y / 2)
    local closestPlayer
    local closestDistance = math.huge
    local localPlayer = game.Players.LocalPlayer
    local camera = workspace.CurrentCamera

    for _, player in ipairs(game.Players:GetPlayers()) do
        if player ~= localPlayer and player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
            local playerRootPart = player.Character.HumanoidRootPart
            local screenPosition, onScreen = camera:WorldToViewportPoint(playerRootPart.Position)

            if onScreen then
                local KOd = player.Character:FindFirstChild("BodyEffects") and player.Character.BodyEffects["K.O"].Value
                local Grabbed = player.Character:FindFirstChild("GRABBING_CONSTRAINT") ~= nil

                if not KOd and not Grabbed then
                    local ray = Ray.new(camera.CFrame.Position, playerRootPart.Position - camera.CFrame.Position)
                    local part, position = workspace:FindPartOnRay(ray, localPlayer.Character, false, true)

                    if part and part:IsDescendantOf(player.Character) then
                        local distance = (centerScreenPosition - Vector2.new(screenPosition.X, screenPosition.Y)).Magnitude

                        if distance < closestDistance and distance <= SilentAimFOVCircle.Radius then
                            closestPlayer = player
                            closestDistance = distance
                        end
                    end
                end
            end
        end
    end

    return closestPlayer
end


local function getPrediction(target)
    if target.Character.Humanoid.FloorMaterial == Enum.Material.Air then
        return extort.Silent.Normal.AirPrediction
    else
        return extort.Silent.Normal.Prediction
    end
end


local Smoothing = 0.025
local Process = 0.1
local Measurement = 0.1
local Estimate = 1
local Gain = 0

local Data = {
    Position = Vector3.new(0, 0, 0),
    Velocity = Vector3.new(0, 0, 0),
    Acceleration = Vector3.new(0, 0, 0)
}

local function CalculateKalman(deltaTime, CurrentPosition, PreviousPosition, PreviousVelocity, PreviousAcceleration)
    local CurrentVelocity = (CurrentPosition - PreviousPosition) / deltaTime
    local PredictedPosition = PreviousPosition + Data.Velocity * deltaTime
    local PredictedVelocity = PreviousVelocity

    Gain = Estimate / (Estimate + Measurement)

    local CorrectedPosition = PredictedPosition + Gain * (CurrentPosition - PredictedPosition)
    local CorrectedVelocity = PredictedVelocity + Gain * (CurrentVelocity - PredictedVelocity)
    Estimate = (1 - Gain) * Estimate + Process

    local Alpha = Smoothing / (Smoothing + deltaTime)
    local SmoothedVelocity = Data.Velocity + Alpha * (CorrectedVelocity - Data.Velocity)
    local SmoothedAcceleration = Alpha * Data.Acceleration + (1 - Alpha) * (SmoothedVelocity - Data.Velocity) / deltaTime

    return SmoothedVelocity, SmoothedAcceleration
end

local function BuildVelocity(deltaTime, Target)
    if Target and Target.Character then
        local CurrentPosition = Target.Character.HumanoidRootPart.Position
        local CurrentVelocity, CurrentAcceleration = CalculateKalman(
            deltaTime, CurrentPosition, Data.Position, Data.Velocity, Data.Acceleration
        )

        Data.Position = CurrentPosition
        Data.Velocity = CurrentVelocity
        Data.Acceleration = CurrentAcceleration

        return CurrentVelocity
    end
end

local SilentAimTarget = nil

local function isInAir(character)
    return character.Humanoid.FloorMaterial == Enum.Material.Air
end



local function adjustVerticalVelocity(targetBone, character)
    if extort.Silent.Checks.AntiGroundShots then
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if humanoid and humanoid:GetState() == Enum.HumanoidStateType.Freefall then
            pcall(function()
                
                targetBone.Velocity = Vector3.new(targetBone.Velocity.X, 0.5, targetBone.Velocity.Z)
                targetBone.AssemblyLinearVelocity = Vector3.new(targetBone.Velocity.X, 0.5, targetBone.Velocity.Z)
            end)
        end
    end
end


game:GetService("RunService").RenderStepped:Connect(function(deltaTime)
    SilentAimTarget = getClosestPlayerToCenter()
    if extort.Silent.Resolver.Enabled then
        local Velocity = BuildVelocity(deltaTime, SilentAimTarget)
    end
end)

game:GetService("RunService").RenderStepped:Connect(function()
    updateFOVCirclePosition()
end)

local mt = getrawmetatable(game)
local old = mt.__namecall
setreadonly(mt, false)
mt.__namecall = newcclosure(function(...)
    local args = {...}
    if extort.Silent.Normal.SilentAimEnabled and SilentAimTarget ~= nil and SilentAimTarget.Character and getnamecallmethod() == "FireServer" then
        if args[2] == "UpdateMousePos" or args[2] == "MOUSE" or args[2] == "UpdateMousePosI2" or args[2] == "MousePosUpdate" then
            local targetBone = SilentAimTarget.Character[extort.Silent.Normal.HitPart]

            adjustVerticalVelocity(targetBone, SilentAimTarget.Character)

            local velocityToUse = extort.Silent.Resolver.Enabled and Data.Velocity or targetBone.Velocity
            args[3] = targetBone.Position + (velocityToUse * getPrediction(SilentAimTarget))

            return old(unpack(args))
        end
    end
    return old(...)
end)
setreadonly(mt, true)


local camlockConfig = extort['CamLock']
getgenv().Prediction = camlockConfig.Normal.Prediction
getgenv().Smoothness = camlockConfig.Normal.Smoothness
getgenv().AimPart = camlockConfig.Normal.AimPart
getgenv().ShakeValue = camlockConfig.Normal.ShakeValue
getgenv().AutoPred = camlockConfig.Normal.AutoPred
getgenv().CamlockTool = camlockConfig.ToggleMethod.CamlockTool
getgenv().FOVSize = camlockConfig.FOVSettings.FOVSize
getgenv().CamlockButton = camlockConfig.ToggleMethod.CamlockButton
getgenv().ShowFOV = camlockConfig.FOVSettings.ShowFOV
getgenv().EasingStyle = camlockConfig.EasingStyle

local player = game.Players.LocalPlayer


local Players = game:GetService("Players")
local RS = game:GetService("RunService")
local WS = game:GetService("Workspace")
local TS = game:GetService("TweenService") 
local SG = game:GetService("StarterGui")


local LP = Players.LocalPlayer
local Camera = WS.CurrentCamera
local ScreenCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2) 

local AimlockState = true
local Locked = false
local Victim = nil


function Notify(tx)
    SG:SetCore("SendNotification", {
        Title = "Extort.lua",
        Text = tx,
        Duration = 5
    })
end


if getgenv().Loaded == true then
    Notify("Loaded")
    return
end

getgenv().Loaded = true


if camlockConfig.Normal.Enabled then
    
    local fov = Drawing.new("Circle")
    fov.Filled = false
    fov.Transparency = 1
    fov.Thickness = 1
    fov.Color = Color3.fromRGB(255, 255, 0)
    fov.NumSides = 1000

    
    local CamlockButtonGui
    local DraggableFrame

    if getgenv().CamlockButton then
        local screenGui = Instance.new("ScreenGui")
        CamlockButtonGui = Instance.new("TextButton")
        DraggableFrame = Instance.new("Frame")

        screenGui.Name = "CamlockGui"
        screenGui.Parent = LP:WaitForChild("PlayerGui")
        screenGui.ResetOnSpawn = false
        
        DraggableFrame.Size = UDim2.new(0, 120, 0, 80) 
        DraggableFrame.Position = UDim2.new(0, 0, 0, 0) 
        DraggableFrame.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
        DraggableFrame.BackgroundTransparency = 0.5
        DraggableFrame.Parent = screenGui

        
        local UICornerFrame = Instance.new("UICorner")
        UICornerFrame.CornerRadius = UDim.new(0.2, 0) 
        UICornerFrame.Parent = DraggableFrame

        
        local dragging = false
        local dragInput, dragStart, startPos

        local function update(input)
            local delta = input.Position - dragStart
            DraggableFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
        end

        DraggableFrame.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                dragging = true
                dragStart = input.Position
                startPos = DraggableFrame.Position
                input.Changed:Connect(function()
                    if input.UserInputState == Enum.UserInputState.End then
                        dragging = false
                    end
                end)
            end
        end)

        DraggableFrame.InputChanged:Connect(function(input)
            if (input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch) and dragging then
                update(input)
            end
        end)

        
        CamlockButtonGui.Size = UDim2.new(0, 100, 0, 50) 
        CamlockButtonGui.Position = UDim2.new(0, 10, 0, 15) 
        CamlockButtonGui.Text = "Extort CAM"
        CamlockButtonGui.TextSize = 12 
        CamlockButtonGui.TextColor3 = Color3.fromRGB(250, 0, 0)
        CamlockButtonGui.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
        CamlockButtonGui.BackgroundTransparency = 0.9
        CamlockButtonGui.TextStrokeTransparency = 0
        CamlockButtonGui.Parent = DraggableFrame

        
        local UICornerButton = Instance.new("UICorner")
        UICornerButton.CornerRadius = UDim.new(0.2, 0) 
        UICornerButton.Parent = CamlockButtonGui

        CamlockButtonGui.MouseButton1Click:Connect(function()
            if not Locked then
                Victim = getClosest()
                if Victim then
                    Locked = true
                    Notify("Locked onto: " .. tostring(Victim.Character.Humanoid.DisplayName))
                else
                    Notify("No target found in FOV!")
                end
            elseif Locked then
                Locked = false
                Victim = nil
                Notify("Aimlock disabled")
            end
        end)
    end

    
    local CamlockToolInstance
    if getgenv().CamlockTool then
        CamlockToolInstance = Instance.new("Tool")
        CamlockToolInstance.RequiresHandle = false
        CamlockToolInstance.Name = "CamlockTool"
        CamlockToolInstance.Parent = player.Backpack
    end

local player = game.Players.LocalPlayer

local function connectCharacterAdded()
    player.CharacterAdded:Connect(onCharacterAdded)
end

connectCharacterAdded()

player.CharacterRemoving:Connect(function()
    CamlockToolInstance.Parent = game.Players.LocalPlayer.Backpack
end)

    
    function update()
        if fov then
            fov.Radius = getgenv().FOVSize * 2
            fov.Visible = getgenv().ShowFOV
            fov.Position = ScreenCenter 
        end
    end

    function getClosest()
        local closestPlayer
        local shortestDistance = math.huge

        for _, v in pairs(game.Players:GetPlayers()) do
            if v ~= LP then 
                local character = v.Character
                if character and character:FindFirstChild("Humanoid") and character.Humanoid.Health > 0 then
                    local aimPart = character:FindFirstChild(getgenv().AimPart) or character:FindFirstChild("Head") or character.PrimaryPart
                    if aimPart then
                        local pos = Camera:WorldToViewportPoint(aimPart.Position)
                        local magnitude = (Vector2.new(pos.X, pos.Y) - ScreenCenter).magnitude

                        
                        if getgenv().ShowFOV then
                            if magnitude < fov.Radius then
                                if magnitude < shortestDistance then
                                    closestPlayer = v
                                    shortestDistance = magnitude
                                end
                            end
                        else
                            if magnitude < shortestDistance then
                                closestPlayer = v
                                shortestDistance = magnitude
                            end
                        end
                    end
                end
            end
        end
        return closestPlayer
    end

    
    if CamlockToolInstance then
        CamlockToolInstance.Activated:Connect(function()
            if AimlockState then
                if Locked then
                    Locked = false
                    Victim = nil
                    Notify("Unlocked!")
                else
                    Victim = getClosest()
                    if Victim then
                        Locked = true
                        Notify("Locked onto: " .. tostring(Victim.Character.Humanoid.DisplayName))
                    else
                        Notify("No target found in FOV!")
                    end
                end
            else
                Notify("Aimlock is not enabled!")
            end
        end)
    end
function isVictimKOd(victim)
    return victim.Character:FindFirstChild("BodyEffects") and victim.Character.BodyEffects["K.O"].Value
end
function isVictimHealthZero(victim)
    return victim.Character.Humanoid.Health <= 0
end

    
    RS.RenderStepped:Connect(function()
    update()
    if AimlockState and Locked then
        if Victim then
            local aimPart = Victim.Character:FindFirstChild(getgenv().AimPart) or Victim.Character:FindFirstChild("Head") or Victim.Character.PrimaryPart
            
            if aimPart then
                
                local isKO = camlockConfig.Check.KOCheck and Victim.Character:FindFirstChild("BodyEffects") and Victim.Character.BodyEffects["K.O"].Value
                local isHealthZero = camlockConfig.Check.KOCheck and isVictimHealthZero(Victim)
                
                if isKO or isHealthZero then
                    Locked = false
                    Victim = nil
                    Notify("Tapped that niggaÃ°Å¸Ëœâ€šÃ°Å¸ËœÂ­")
                else
                    local shakeOffset = Vector3.new(
                        math.random(-getgenv().ShakeValue, getgenv().ShakeValue),
                        math.random(-getgenv().ShakeValue, getgenv().ShakeValue),
                        math.random(-getgenv().ShakeValue, getgenv().ShakeValue)
                    ) * 0.1
                    local LookPosition = CFrame.new(Camera.CFrame.p, aimPart.Position + (Vector3.new(Victim.Character.HumanoidRootPart.Velocity.X, Victim.Character.HumanoidRootPart.AssemblyAngularVelocity.Y * 0.5, Victim.Character.HumanoidRootPart.Velocity.Z) * getgenv().Prediction)) + shakeOffset
                    Camera.CFrame = Camera.CFrame:Lerp(LookPosition, getgenv().Smoothness, getgenv().EasingStyle)
                end
            end
        end
    end
end)


    
    for _, con in next, getconnections(workspace.CurrentCamera.Changed) do
        task.wait()
        con:Disable()
    end
    for _, con in next, getconnections(workspace.CurrentCamera:GetPropertyChangedSignal("CFrame")) do
        task.wait()
        con:Disable()
    end

    
    while task.wait() do
        if getgenv().AutoPred then
            local pingvalue = game:GetService("Stats").Network.ServerStatsItem["Data Ping"]:GetValueString()
            local split = string.split(pingvalue, '(')
            local ping = tonumber(split[1])
            if ping < 200 then
                getgenv().Prediction = 0.1973432432343325
            elseif ping < 150 then
                getgenv().Prediction = 0.1922
            elseif ping < 90 then
                getgenv().Prediction = 0.16
            elseif ping < 80 then
                getgenv().Prediction = 0.169
            elseif ping < 70 then
                getgenv().Prediction = 0.1355
            elseif ping < 50 then
                getgenv().Prediction = 0.125
            elseif ping < 40 then
                getgenv().Prediction = 0.12
            elseif ping < 30 then
                getgenv().Prediction = 0.12
            end
        end
    end
end 
