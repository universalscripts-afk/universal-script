-- Universal Shooter Framework (Auto-Shoot, Wall Check, Smooth Interpolation & Status Tags)
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local CoreGui = game:GetService("CoreGui")
local VirtualInputManager = game:GetService("VirtualInputManager")

local LocalPlayer = Players.LocalPlayer
local Camera = Workspace.CurrentCamera

-- Configuration & Default Keybinds
local Config = {
    AimbotEnabledState = false,
    AimbotFOV = 150,
    ShowFOVCircle = false,
    AimbotKey = Enum.KeyCode.E,
    SmoothAimbot = false,
    Smoothness = 5, -- Higher value = smoother / slower interpolation
    WallCheck = false,
    AutoShoot = false,
    
    -- Visuals
    BoxESP = false,
    Tracers = false,
    NameESP = false,
    DistanceESP = false,
    HealthBar = false,
    
    -- Movement
    SpeedHack = false,
    WalkSpeed = 32,
    JumpHack = false,
    JumpPower = 100,
    Noclip = false,
    
    -- Menu Keybind
    MenuKey = Enum.KeyCode.RightControl
}

-- Cleanup old elements
if CoreGui:FindFirstChild("UniversalShooterMenu") then CoreGui.UniversalShooterMenu:Destroy() end
if CoreGui:FindFirstChild("FOVGui") then CoreGui.FOVGui:Destroy() end

---------------------------------------------------------
-- FOV CIRCLE DRAWING (CENTERED)
---------------------------------------------------------
local FOVGui = Instance.new("ScreenGui", CoreGui)
FOVGui.Name = "FOVGui"

local FOVCircle = Instance.new("Frame", FOVGui)
FOVCircle.AnchorPoint = Vector2.new(0.5, 0.5)
FOVCircle.Position = UDim2.new(0.5, 0, 0.5, 0)
FOVCircle.BackgroundTransparency = 1
FOVCircle.Size = UDim2.new(0, Config.AimbotFOV * 2, 0, Config.AimbotFOV * 2)
FOVCircle.Visible = false

local FOVCorner = Instance.new("UICorner", FOVCircle)
FOVCorner.CornerRadius = UDim.new(1, 0)

local FOVStroke = Instance.new("UIStroke", FOVCircle)
FOVStroke.Color = Color3.fromRGB(0, 255, 200)
FOVStroke.Thickness = 1.5

---------------------------------------------------------
-- TARGETING & WALL CHECK SYSTEM
---------------------------------------------------------
local currentLockedTarget = nil
local isShooting = false

local function isVisible(targetPart)
    if not Config.WallCheck then return true end
    local origin = Camera.CFrame.Position
    local direction = (targetPart.Position - origin)
    
    local raycastParams = RaycastParams.new()
    raycastParams.FilterType = Enum.RaycastFilterType.Blacklist
    
    local ignoreList = {LocalPlayer.Character}
    if targetPart.Parent then table.insert(ignoreList, targetPart.Parent) end
    raycastParams.FilterDescendantsInstances = ignoreList
    
    local result = Workspace:Raycast(origin, direction, raycastParams)
    return result == nil
end

local function getClosestPlayerToCenter()
    local closestPlayer = nil
    local shortestDistance = Config.AimbotFOV
    local screenCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)

    for _, player in pairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("Head") then
            local humanoid = player.Character:FindFirstChildOfClass("Humanoid")
            if humanoid and humanoid.Health > 0 then
                local head = player.Character.Head
                local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)

                if onScreen and isVisible(head) then
                    local distance = (Vector2.new(screenPos.X, screenPos.Y) - screenCenter).Magnitude
                    if distance < shortestDistance then
                        closestPlayer = head
                        shortestDistance = distance
                    end
                end
            end
        end
    end
    return closestPlayer
end

---------------------------------------------------------
-- ESP & TRACERS MANAGEMENT
---------------------------------------------------------
local ESPData = {}

local function setupPlayerESP(player)
    if player == LocalPlayer then return end

    local function onCharacterAdded(character)
        if ESPData[player] then
            if ESPData[player].Highlight then ESPData[player].Highlight:Destroy() end
            if ESPData[player].Beam then ESPData[player].Beam:Destroy() end
            if ESPData[player].Attachment0 then ESPData[player].Attachment0:Destroy() end
            if ESPData[player].Attachment1 then ESPData[player].Attachment1:Destroy() end
            if ESPData[player].Billboard then ESPData[player].Billboard:Destroy() end
        end

        local rootPart = character:WaitForChild("HumanoidRootPart", 5)
        if not rootPart then return end

        local highlight = Instance.new("Highlight")
        highlight.Name = "ESPHighlight"
        highlight.FillTransparency = 0.5
        highlight.FillColor = Color3.fromRGB(255, 50, 50)
        highlight.OutlineColor = Color3.fromRGB(255, 255, 255)
        highlight.OutlineTransparency = 0
        highlight.Adornee = character
        highlight.Enabled = Config.BoxESP
        highlight.Parent = character

        local att0 = Instance.new("Attachment", Workspace.Terrain)
        local att1 = Instance.new("Attachment", rootPart)

        local beam = Instance.new("Beam")
        beam.Name = "ESPTracer"
        beam.Attachment0 = att0
        beam.Attachment1 = att1
        beam.Color = ColorSequence.new(Color3.fromRGB(0, 255, 200))
        beam.Width0 = 0.1
        beam.Width1 = 0.1
        beam.FaceCamera = true
        beam.Enabled = Config.Tracers
        beam.Parent = rootPart

        local billboard = Instance.new("BillboardGui")
        billboard.Name = "ESPInfo"
        billboard.Adornee = character:WaitForChild("Head", 5)
        billboard.Size = UDim2.new(0, 200, 0, 70)
        billboard.StudsOffset = Vector3.new(0, 2.5, 0)
        billboard.AlwaysOnTop = true
        billboard.Enabled = Config.NameESP or Config.DistanceESP or Config.HealthBar
        billboard.Parent = character

        local label = Instance.new("TextLabel", billboard)
        label.Size = UDim2.new(1, 0, 0, 25)
        label.Position = UDim2.new(0, 0, 0, 0)
        label.BackgroundTransparency = 1
        label.TextColor3 = Color3.fromRGB(255, 255, 255)
        label.TextStrokeTransparency = 0
        label.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
        label.Font = Enum.Font.SourceSansBold
        label.TextSize = 14

        local healthBarBg = Instance.new("Frame", billboard)
        healthBarBg.Name = "HealthBarBg"
        healthBarBg.Size = UDim2.new(0, 80, 0, 8)
        healthBarBg.Position = UDim2.new(0.5, -40, 0, 28)
        healthBarBg.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
        healthBarBg.BorderSizePixel = 0
        healthBarBg.Visible = Config.HealthBar

        local healthBarFill = Instance.new("Frame", healthBarBg)
        healthBarFill.Name = "HealthBarFill"
        healthBarFill.Size = UDim2.new(1, 0, 1, 0)
        healthBarFill.BackgroundColor3 = Color3.fromRGB(0, 255, 100)
        healthBarFill.BorderSizePixel = 0

        local healthText = Instance.new("TextLabel", billboard)
        healthText.Name = "HealthText"
        healthText.Size = UDim2.new(1, 0, 0, 20)
        healthText.Position = UDim2.new(0, 0, 0, 38)
        healthText.BackgroundTransparency = 1
        healthText.TextColor3 = Color3.fromRGB(0, 255, 100)
        healthText.TextStrokeTransparency = 0
        healthText.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
        healthText.Font = Enum.Font.SourceSansBold
        healthText.TextSize = 12
        healthText.Visible = Config.HealthBar

        ESPData[player] = {
            Character = character,
            Highlight = highlight,
            Beam = beam,
            Attachment0 = att0,
            Attachment1 = att1,
            Billboard = billboard,
            Label = label,
            HealthBarBg = healthBarBg,
            HealthBarFill = healthBarFill,
            HealthText = healthText
        }
    end

    if player.Character then onCharacterAdded(player.Character) end
    player.CharacterAdded:Connect(onCharacterAdded)
end

local function removePlayerESP(player)
    if ESPData[player] then
        if ESPData[player].Highlight then ESPData[player].Highlight:Destroy() end
        if ESPData[player].Beam then ESPData[player].Beam:Destroy() end
        if ESPData[player].Attachment0 then ESPData[player].Attachment0:Destroy() end
        if ESPData[player].Attachment1 then ESPData[player].Attachment1:Destroy() end
        if ESPData[player].Billboard then ESPData[player].Billboard:Destroy() end
        ESPData[player] = nil
    end
end

for _, p in pairs(Players:GetPlayers()) do setupPlayerESP(p) end
Players.PlayerAdded:Connect(setupPlayerESP)
Players.PlayerRemoving:Connect(removePlayerESP)

---------------------------------------------------------
-- MOVEMENT AUTO-LOCK HOOK
---------------------------------------------------------
local function hookCharacter(character)
    if not character then return end
    local humanoid = character:WaitForChild("Humanoid", 5)
    if not humanoid then return end

    humanoid:GetPropertyChangedSignal("WalkSpeed"):Connect(function()
        if Config.SpeedHack then
            humanoid.WalkSpeed = Config.WalkSpeed
        end
    end)
end

if LocalPlayer.Character then hookCharacter(LocalPlayer.Character) end
LocalPlayer.CharacterAdded:Connect(hookCharacter)

---------------------------------------------------------
-- MAIN RENDER LOOP (AIMBOT, SMOOTHING & AUTO-SHOOT)
---------------------------------------------------------
RunService.RenderStepped:Connect(function()
    FOVCircle.Size = UDim2.new(0, Config.AimbotFOV * 2, 0, Config.AimbotFOV * 2)
    FOVCircle.Visible = Config.ShowFOVCircle

    currentLockedTarget = nil
    if Config.AimbotEnabledState then
        local targetHead = getClosestPlayerToCenter()
        if targetHead then
            currentLockedTarget = targetHead.Parent
            local targetCFrame = CFrame.new(Camera.CFrame.Position, targetHead.Position)
            
            if Config.SmoothAimbot then
                Camera.CFrame = Camera.CFrame:Lerp(targetCFrame, 1 / math.max(Config.Smoothness, 1))
            else
                Camera.CFrame = targetCFrame
            end

            -- Auto-Shoot Handling
            if Config.AutoShoot and not isShooting then
                isShooting = true
                task.spawn(function()
                    VirtualInputManager:SendMouseButtonEvent(0, 0, 0, true, game, 0)
                    task.wait(0.05)
                    VirtualInputManager:SendMouseButtonEvent(0, 0, 0, false, game, 0)
                    task.wait(0.05)
                    isShooting = false
                end)
            end
        end
    end

    -- Movement Modifications & Force Push Bypass
    if LocalPlayer.Character then
        local humanoid = LocalPlayer.Character:FindFirstChildOfClass("Humanoid")
        local rootPart = LocalPlayer.Character:FindFirstChild("HumanoidRootPart")

        if humanoid then
            if Config.SpeedHack then 
                humanoid.WalkSpeed = Config.WalkSpeed 
                if rootPart and humanoid.MoveDirection.Magnitude > 0 then
                    rootPart.CFrame = rootPart.CFrame + (humanoid.MoveDirection * (Config.WalkSpeed / 16) * 0.12)
                end
            end
            if Config.JumpHack then
                humanoid.UseJumpPower = true
                humanoid.JumpPower = Config.JumpPower
            end
        end

        if Config.Noclip then
            for _, part in pairs(LocalPlayer.Character:GetDescendants()) do
                if part:IsA("BasePart") and part.CanCollide then
                    part.CanCollide = false
                end
            end
        end
    end

    -- ESP Visuals & Dynamic Color Updates
    local localRoot = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")

    for player, esp in pairs(ESPData) do
        local char = player.Character
        local humanoid = char and char:FindFirstChildOfClass("Humanoid")
        local rootPart = char and char:FindFirstChild("HumanoidRootPart")

        if char and humanoid and rootPart and humanoid.Health > 0 then
            if char == currentLockedTarget then
                esp.Highlight.FillColor = Color3.fromRGB(0, 255, 100)
                esp.Highlight.Enabled = true
            else
                esp.Highlight.FillColor = Color3.fromRGB(255, 50, 50)
                esp.Highlight.Enabled = Config.BoxESP
            end

            if Config.Tracers and localRoot then
                esp.Attachment0.WorldPosition = localRoot.Position - Vector3.new(0, 2, 0)
                esp.Beam.Enabled = true
            else
                esp.Beam.Enabled = false
            end

            if Config.NameESP or Config.DistanceESP then
                esp.Billboard.Enabled = true
                local text = ""
                if Config.NameESP then text = player.Name end
                if Config.DistanceESP then
                    local dist = math.floor((rootPart.Position - Camera.CFrame.Position).Magnitude)
                    text = text .. (text ~= "" and " [" or "[") .. dist .. "m]"
                end
                esp.Label.Text = text
                esp.Label.Visible = true
            else
                esp.Label.Visible = false
            end

            esp.HealthBarBg.Visible = Config.HealthBar
            esp.HealthText.Visible = Config.HealthBar
            if Config.HealthBar then
                local healthPercent = math.clamp(humanoid.Health / humanoid.MaxHealth, 0, 1)
                esp.HealthBarFill.Size = UDim2.new(healthPercent, 0, 1, 0)
                esp.HealthText.Text = "HP: " .. math.floor(humanoid.Health) .. "/" .. math.floor(humanoid.MaxHealth)
            end

            if not Config.BoxESP and not Config.Tracers and not Config.NameESP and not Config.DistanceESP and not Config.HealthBar and char ~= currentLockedTarget then
                esp.Billboard.Enabled = false
            end
        else
            esp.Highlight.Enabled = false
            esp.Beam.Enabled = false
            esp.Billboard.Enabled = false
        end
    end
end)

---------------------------------------------------------
-- UI MENU BUILDER
---------------------------------------------------------
local ScreenGui = Instance.new("ScreenGui", CoreGui)
ScreenGui.Name = "UniversalShooterMenu"

local MainFrame = Instance.new("Frame", ScreenGui)
MainFrame.Size = UDim2.new(0, 440, 0, 310)
MainFrame.Position = UDim2.new(0.5, -220, 0.5, -155)
MainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 22)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true

Instance.new("UICorner", MainFrame).CornerRadius = UDim.new(0, 6)

local TitleBar = Instance.new("Frame", MainFrame)
TitleBar.Size = UDim2.new(1, 0, 0, 30)
TitleBar.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
TitleBar.BorderSizePixel = 0

local TitleText = Instance.new("TextLabel", TitleBar)
TitleText.Size = UDim2.new(1, -10, 1, 0)
TitleText.Position = UDim2.new(0, 10, 0, 0)
TitleText.BackgroundTransparency = 1
TitleText.Text = "Universal Shooter Framework"
TitleText.TextColor3 = Color3.fromRGB(255, 255, 255)
TitleText.TextSize = 13
TitleText.Font = Enum.Font.SourceSansBold
TitleText.TextXAlignment = Enum.TextXAlignment.Left

local Sidebar = Instance.new("Frame", MainFrame)
Sidebar.Size = UDim2.new(0, 110, 1, -30)
Sidebar.Position = UDim2.new(0, 0, 0, 30)
Sidebar.BackgroundColor3 = Color3.fromRGB(15, 15, 18)
Sidebar.BorderSizePixel = 0

local SidebarLayout = Instance.new("UIListLayout", Sidebar)
SidebarLayout.Padding = UDim.new(0, 4)

local ContentContainer = Instance.new("Frame", MainFrame)
ContentContainer.Size = UDim2.new(1, -120, 1, -35)
ContentContainer.Position = UDim2.new(0, 115, 0, 35)
ContentContainer.BackgroundTransparency = 1

-- Dragging System
local dragging, dragStart, startPos
TitleBar.InputBegan:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        dragging = true
        dragStart = input.Position
        startPos = MainFrame.Position
    end
end)
UserInputService.InputChanged:Connect(function(input)
    if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
        local delta = input.Position - dragStart
        MainFrame.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + delta.X, startPos.Y.Scale, startPos.Y.Offset + delta.Y)
    end
end)
UserInputService.InputEnded:Connect(function(input)
    if input.UserInputType == Enum.UserInputType.MouseButton1 then dragging = false end
end)

-- Global Key Listener
local aimbotKeyButtonRef = nil

UserInputService.InputBegan:Connect(function(input, gpe)
    if not gpe and input.UserInputType == Enum.UserInputType.Keyboard then
        if input.KeyCode == Config.MenuKey then
            MainFrame.Visible = not MainFrame.Visible
        elseif input.KeyCode == Config.AimbotKey then
            Config.AimbotEnabledState = not Config.AimbotEnabledState
            if aimbotKeyButtonRef then
                local statusText = Config.AimbotEnabledState and "ON" or "OFF"
                aimbotKeyButtonRef.Text = "Aimbot Toggle Keybind: [" .. tostring(Config.AimbotKey.Name) .. "] (" .. statusText .. ")"
                aimbotKeyButtonRef.TextColor3 = Config.AimbotEnabledState and Color3.fromRGB(0, 255, 120) or Color3.fromRGB(200, 200, 200)
            end
        end
    end
end)

-- Tab System
local activeTab = nil
local function CreateTab(name)
    local TabBtn = Instance.new("TextButton", Sidebar)
    TabBtn.Size = UDim2.new(1, 0, 0, 28)
    TabBtn.BackgroundColor3 = Color3.fromRGB(25, 25, 28)
    TabBtn.BorderSizePixel = 0
    TabBtn.Text = name
    TabBtn.TextColor3 = Color3.fromRGB(150, 150, 150)
    TabBtn.Font = Enum.Font.SourceSans
    TabBtn.TextSize = 13

    local Page = Instance.new("ScrollingFrame", ContentContainer)
    Page.Size = UDim2.new(1, 0, 1, 0)
    Page.BackgroundTransparency = 1
    Page.BorderSizePixel = 0
    Page.ScrollBarThickness = 2
    Page.Visible = false

    local PageLayout = Instance.new("UIListLayout", Page)
    PageLayout.Padding = UDim.new(0, 5)

    TabBtn.MouseButton1Click:Connect(function()
        for _, p in pairs(ContentContainer:GetChildren()) do p.Visible = false end
        for _, b in pairs(Sidebar:GetChildren()) do
            if b:IsA("TextButton") then b.TextColor3 = Color3.fromRGB(150, 150, 150) end
        end
        Page.Visible = true
        TabBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    end)

    if not activeTab then
        activeTab = Page
        Page.Visible = true
        TabBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    end

    local Elements = {}

    function Elements:AddToggle(text, defaultState, callback)
        local btn = Instance.new("TextButton", Page)
        btn.Size = UDim2.new(1, -5, 0, 26)
        btn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
        btn.BorderSizePixel = 0
        btn.Text = text .. (defaultState and " [ ON ]" or " [ OFF ]")
        btn.TextColor3 = defaultState and Color3.fromRGB(0, 255, 120) or Color3.fromRGB(180, 180, 180)
        btn.Font = Enum.Font.SourceSans
        btn.TextSize = 13

        local state = defaultState
        btn.MouseButton1Click:Connect(function()
            state = not state
            btn.Text = text .. (state and " [ ON ]" or " [ OFF ]")
            btn.TextColor3 = state and Color3.fromRGB(0, 255, 120) or Color3.fromRGB(180, 180, 180)
            callback(state)
        end)
    end

    function Elements:AddAimbotKeybind(labelText, defaultKey, callback)
        local btn = Instance.new("TextButton", Page)
        btn.Size = UDim2.new(1, -5, 0, 26)
        btn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
        btn.BorderSizePixel = 0
        local statusText = Config.AimbotEnabledState and "ON" or "OFF"
        btn.Text = labelText .. ": [" .. tostring(defaultKey.Name) .. "] (" .. statusText .. ")"
        btn.TextColor3 = Config.AimbotEnabledState and Color3.fromRGB(0, 255, 120) or Color3.fromRGB(200, 200, 200)
        btn.Font = Enum.Font.SourceSans
        btn.TextSize = 13
        
        aimbotKeyButtonRef = btn

        local listening = false
        btn.MouseButton1Click:Connect(function()
            if listening then return end
            listening = true
            btn.Text = labelText .. ": [Press Any Key...]"
            btn.TextColor3 = Color3.fromRGB(0, 255, 120)

            local connection
            connection = UserInputService.InputBegan:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.Keyboard then
                    Config.AimbotKey = input.KeyCode
                    local curStatus = Config.AimbotEnabledState and "ON" or "OFF"
                    btn.Text = labelText .. ": [" .. tostring(input.KeyCode.Name) .. "] (" .. curStatus .. ")"
                    btn.TextColor3 = Config.AimbotEnabledState and Color3.fromRGB(0, 255, 120) or Color3.fromRGB(200, 200, 200)
                    callback(input.KeyCode)
                    connection:Disconnect()
                    listening = false
                end
            end)
        end)
    end

    function Elements:AddKeybind(settingName, labelText, defaultKey, callback)
        local btn = Instance.new("TextButton", Page)
        btn.Size = UDim2.new(1, -5, 0, 26)
        btn.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
        btn.BorderSizePixel = 0
        btn.Text = labelText .. ": [" .. tostring(defaultKey.Name) .. "]"
        btn.TextColor3 = Color3.fromRGB(200, 200, 200)
        btn.Font = Enum.Font.SourceSans
        btn.TextSize = 13

        local listening = false
        btn.MouseButton1Click:Connect(function()
            if listening then return end
            listening = true
            btn.Text = labelText .. ": [Press Any Key...]"
            btn.TextColor3 = Color3.fromRGB(0, 255, 120)

            local connection
            connection = UserInputService.InputBegan:Connect(function(input)
                if input.UserInputType == Enum.UserInputType.Keyboard then
                    Config[settingName] = input.KeyCode
                    btn.Text = labelText .. ": [" .. tostring(input.KeyCode.Name) .. "]"
                    btn.TextColor3 = Color3.fromRGB(200, 200, 200)
                    callback(input.KeyCode)
                    connection:Disconnect()
                    listening = false
                end
            end)
        end)
    end

    function Elements:AddSlider(text, min, max, default, callback)
        local Frame = Instance.new("Frame", Page)
        Frame.Size = UDim2.new(1, -5, 0, 42)
        Frame.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
        Frame.BorderSizePixel = 0

        local Label = Instance.new("TextLabel", Frame)
        Label.Size = UDim2.new(1, -10, 0, 18)
        Label.Position = UDim2.new(0, 5, 0, 2)
        Label.BackgroundTransparency = 1
        Label.Text = text .. ": " .. tostring(default)
        Label.TextColor3 = Color3.fromRGB(255, 255, 255)
        Label.TextSize = 12
        Label.Font = Enum.Font.SourceSans
        Label.TextXAlignment = Enum.TextXAlignment.Left

        local Track = Instance.new("Frame", Frame)
        Track.Size = UDim2.new(1, -10, 0, 10)
        Track.Position = UDim2.new(0, 5, 0, 24)
        Track.BackgroundColor3 = Color3.fromRGB(15, 15, 20)
        Track.BorderSizePixel = 0

        local Fill = Instance.new("Frame", Track)
        Fill.Size = UDim2.new((default - min) / (max - min), 0, 1, 0)
        Fill.BackgroundColor3 = Color3.fromRGB(0, 255, 120)
        Fill.BorderSizePixel = 0

        local isSliding = false

        local function update(input)
            local pos = math.clamp((input.Position.X - Track.AbsolutePosition.X) / Track.AbsoluteSize.X, 0, 1)
            local val = math.floor(min + ((max - min) * pos))
            Fill.Size = UDim2.new(pos, 0, 1, 0)
            Label.Text = text .. ": " .. tostring(val)
            callback(val)
        end

        Track.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 then
                isSliding = true
                update(input)
            end
        end)

        UserInputService.InputChanged:Connect(function(input)
            if isSliding and input.UserInputType == Enum.UserInputType.MouseMovement then
                update(input)
            end
        end)

        UserInputService.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 then
                isSliding = false
            end
        end)
    end

    return Elements
end

---------------------------------------------------------
-- MENU CONFIGURATION
---------------------------------------------------------
local CombatTab = CreateTab("Combat")
CombatTab:AddAimbotKeybind("Aimbot Toggle Keybind", Enum.KeyCode.E, function(key) Config.AimbotKey = key end)
CombatTab:AddToggle("Show FOV Circle", false, function(s) Config.ShowFOVCircle = s end)
CombatTab:AddSlider("FOV Radius", 30, 500, 150, function(value) Config.AimbotFOV = value end)
CombatTab:AddToggle("Smooth Smoothing", false, function(s) Config.SmoothAimbot = s end)
CombatTab:AddSlider("Smoothness Value", 1, 20, 5, function(v) Config.Smoothness = v end)
CombatTab:AddToggle("Wall Check", false, function(s) Config.WallCheck = s end)
CombatTab:AddToggle("Auto Shoot", false, function(s) Config.AutoShoot = s end)

local VisualsTab = CreateTab("Visuals")
VisualsTab:AddToggle("Chams / Box ESP", false, function(s) Config.BoxESP = s end)
VisualsTab:AddToggle("Tracers (3D Beams)", false, function(s) Config.Tracers = s end)
VisualsTab:AddToggle("Name ESP", false, function(s) Config.NameESP = s end)
VisualsTab:AddToggle("Distance ESP", false, function(s) Config.DistanceESP = s end)
VisualsTab:AddToggle("Health Bar & HP Text", false, function(s) Config.HealthBar = s end)

local MovementTab = CreateTab("Movement")
MovementTab:AddToggle("Enable Speed Hack", false, function(s)
    Config.SpeedHack = s
    if not s and LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
        LocalPlayer.Character:FindFirstChildOfClass("Humanoid").WalkSpeed = 16
    end
end)
MovementTab:AddSlider("WalkSpeed", 16, 200, 32, function(v) Config.WalkSpeed = v end)

MovementTab:AddToggle("Enable Jump Hack", false, function(s)
    Config.JumpHack = s
    if not s and LocalPlayer.Character and LocalPlayer.Character:FindFirstChildOfClass("Humanoid") then
        LocalPlayer.Character:FindFirstChildOfClass("Humanoid").JumpPower = 50
    end
end)
MovementTab:AddSlider("JumpPower", 50, 300, 100, function(v) Config.JumpPower = v end)

MovementTab:AddToggle("Noclip (Walk Through Walls)", false, function(s) Config.Noclip = s end)

local SettingsTab = CreateTab("Settings")
SettingsTab:AddKeybind("MenuKey", "Toggle Menu Key", Enum.KeyCode.RightControl, function(key) Config.MenuKey = key end)
