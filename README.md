

local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")

local LocalPlayer = Players.LocalPlayer
local Character, Humanoid, RootPart = nil, nil, nil
local lastDashTime = 0
local dashDelay = 0.35
local isEnabled = true
local isCoolingDown = false
local cooldownDuration = 4


local function findNearestPlayer()
    local nearestDistance = 20
    local nearestPlayer = nil
    if not RootPart then return nil end
    for _, obj in pairs(workspace:GetDescendants()) do
        if obj:IsA("Model") and obj:FindFirstChild("HumanoidRootPart") and obj ~= Character then
            local success, distance = pcall(function()
                return (RootPart.Position - obj.HumanoidRootPart.Position).Magnitude
            end)
            if success and distance and distance < nearestDistance then
                nearestDistance = distance
                nearestPlayer = obj
            end
        end
    end
    return nearestPlayer
end


local function sendRemoteEvents()
    pcall(function()
        local dashData = {[1] = {Dash = Enum.KeyCode.W, Key = Enum.KeyCode.Q, Goal = "KeyPress"}}
        if Character and Character:FindFirstChild("Communicate") then
            Character.Communicate:FireServer(unpack(dashData))
        end
    end)
    
    local function findNilInstance(name, className)
        for _, instance in pairs(getnilinstances()) do
            if instance.ClassName == className and instance.Name == name then
                return instance
            end
        end
    end
    
    pcall(function()
        local deleteData = {[1] = {Goal = "delete bv", BV = findNilInstance("moveme", "BodyVelocity")}}
        if Character and Character:FindFirstChild("Communicate") then
            Character.Communicate:FireServer(unpack(deleteData))
        end
    end)
end


local function executeDash()
    if not Character or not Humanoid or not RootPart then return end
    local target = findNearestPlayer()
    if not target then return end
    local targetRoot = target:FindFirstChild("HumanoidRootPart")
    if not targetRoot then return end

    pcall(function()
        Humanoid.WalkSpeed = 0
        Humanoid.PlatformStand = true
        Humanoid:ChangeState(Enum.HumanoidStateType.Physics)
    end)

    pcall(sendRemoteEvents)

    -- Inclinando conforme o source original
    if RootPart then
        RootPart.CFrame = RootPart.CFrame * CFrame.Angles(math.rad(85), 0, 0)
    end

    local dashDuration = 0.4
    local startTime = tick()
    local dashConnection
    
    dashConnection = RunService.Heartbeat:Connect(function()
        if dashDuration <= tick() - startTime then
            dashConnection:Disconnect()
            return
        end
        local success, cframe = pcall(function()
            return CFrame.new((targetRoot.Position - targetRoot.CFrame.LookVector * 0.3)) * CFrame.Angles(math.rad(85), 0, 0)
        end)
        if success and cframe and RootPart then
            RootPart.CFrame = cframe
        end
    end)

    repeat task.wait() until dashDuration <= tick() - startTime
    
    pcall(function()
        Humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
        Humanoid.PlatformStand = false
        Humanoid.WalkSpeed = 16
    end)
end


local mainGui = Instance.new("ScreenGui", LocalPlayer.PlayerGui)
mainGui.Name = "GojoRGB_Hub"
mainGui.ResetOnSpawn = false

local frame = Instance.new("Frame", mainGui)
frame.Size = UDim2.new(0, 60, 0, 60)
frame.Position = UDim2.new(0.5, -30, 0.8, 0)
frame.BackgroundColor3 = Color3.fromRGB(15, 15, 15)
frame.Active = true
frame.Draggable = true
Instance.new("UICorner", frame)
local stroke = Instance.new("UIStroke", frame)
stroke.Thickness = 2

local btn = Instance.new("TextButton", frame)
btn.Size = UDim2.new(1, 0, 1, 0)
btn.BackgroundTransparency = 1
btn.Text = "On"
btn.Font = Enum.Font.GothamBold
btn.TextSize = 14

-- Notificação de Cooldown (Como no source original)
local cdLabel = Instance.new("TextLabel", mainGui)
cdLabel.Size = UDim2.new(0, 100, 0, 20)
cdLabel.BackgroundTransparency = 1
cdLabel.TextColor3 = Color3.new(1,1,1)
cdLabel.Font = Enum.Font.GothamBold
cdLabel.TextSize = 12
cdLabel.Visible = false

RunService.RenderStepped:Connect(function()
    local c = Color3.fromHSV(tick() % 5 / 5, 0.8, 1)
    stroke.Color = c
    btn.TextColor3 = c
    cdLabel.Position = UDim2.new(frame.Position.X.Scale, frame.Position.X.Offset - 20, frame.Position.Y.Scale, frame.Position.Y.Offset + 65)
end)

local function handleCooldown(duration)
    isCoolingDown = true
    cdLabel.Visible = true
    local start = tick()
    while tick() - start < duration do
        cdLabel.Text = string.format("CD: %.1fs", duration - (tick() - start))
        task.wait(0.1)
    end
    isCoolingDown = false
    cdLabel.Visible = false
end

btn.MouseButton1Click:Connect(function()
    isEnabled = not isEnabled
    btn.Text = isEnabled and "On" or "Off"
end)


local function showNotification()
    local notif = Instance.new("Frame", mainGui)
    notif.Size = UDim2.new(0, 250, 0, 50)
    notif.Position = UDim2.new(0.5, -125, 0.1, 0)
    notif.BackgroundColor3 = Color3.fromRGB(20, 20, 20)
    Instance.new("UICorner", notif)
    local l = Instance.new("TextLabel", notif)
    l.Size = UDim2.new(1,0,1,0)
    l.Text = "Made by SA_DOMINICK"
    l.TextColor3 = Color3.new(1,1,1)
    l.Font = Enum.Font.GothamBold
    l.BackgroundTransparency = 1
    task.delay(3, function() notif:Destroy() end)
end


local function onAnimationPlayed(track)
    local id = tostring(track.Animation.AnimationId)
    if string.find(id, "10503381238") or string.find(id, "13379003796") then
        if isEnabled and not isCoolingDown then
            task.delay(0.3, function()
                executeDash()
                handleCooldown(cooldownDuration)
            end)
        end
    end
end

local function setupCharacter(newChar)
    Character = newChar
    Humanoid = newChar:WaitForChild("Humanoid")
    RootPart = newChar:WaitForChild("HumanoidRootPart")
    Humanoid.AnimationPlayed:Connect(onAnimationPlayed)
end

if LocalPlayer.Character then setupCharacter(LocalPlayer.Character) end
LocalPlayer.CharacterAdded:Connect(setupCharacter)
showNotification()


