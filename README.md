-- ============================================================================
-- JK ADMIN AIM — SISTEMA ADMINISTRATIVO INTEGRADO (SERVIDOR & CLIENTE)
-- ============================================================================

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")

local player = Players.LocalPlayer
local camera = Workspace.CurrentCamera

-- ============================================================================
-- 1. CONFIGURAÇÃO E WHITELIST (Compartilhado)
-- ============================================================================
local Config = {}
Config.WhitelistedUserIds = {
    12345678, -- Substitua pelo UserId real do seu amigo (dono/desenvolvedor)
}

function Config.IsAuthorized(userId: number): boolean
    for _, id in ipairs(Config.WhitelistedUserIds) do
        if id == userId then return true end
    end
    return false
end

-- ============================================================================
-- 2. LÓGICA DO SERVIDOR (Executado apenas no Server)
-- ============================================================================
if RunService:IsServer() then
    local JKAdminStorage = Instance.new("Folder")
    JKAdminStorage.Name = "JKAdminStorage"
    JKAdminStorage.Parent = ReplicatedStorage

    local RemoteEvents = Instance.new("Folder")
    RemoteEvents.Name = "RemoteEvents"
    RemoteEvents.Parent = JKAdminStorage

    local RequestToggleFeature = Instance.new("RemoteFunction")
    RequestToggleFeature.Name = "RequestToggleFeature"
    RequestToggleFeature.Parent = RemoteEvents

    local activeStates = {}

    Players.PlayerRemoving:Connect(function(plr)
        activeStates[plr] = nil
    end)

    RequestToggleFeature.OnServerInvoke = function(plr, featureName, state)
        if not Config.IsAuthorized(plr.UserId) then
            warn("[JK ADMIN] Tentativa de acesso não autorizado bloqueada: " .. plr.Name)
            return false
        end

        if featureName == "Fly" or featureName == "NoClip" then
            if not activeStates[plr] then activeStates[plr] = {} end
            activeStates[plr][featureName] = state
            
            local character = plr.Character
            if character then
                local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
                
                if featureName == "Fly" and humanoidRootPart then
                    local bv = humanoidRootPart:FindFirstChild("JKAdminBodyVelocity")
                    local bg = humanoidRootPart:FindFirstChild("JKAdminBodyGyro")
                    
                    if state then
                        if not bv then
                            bv = Instance.new("BodyVelocity")
                            bv.Name = "JKAdminBodyVelocity"
                            bv.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
                            bv.Velocity = Vector3.zero
                            bv.Parent = humanoidRootPart
                            
                            bg = Instance.new("BodyGyro")
                            bg.Name = "JKAdminBodyGyro"
                            bg.MaxTorque = Vector3.new(math.huge, math.huge, math.huge)
                            bg.CFrame = humanoidRootPart.CFrame
                            bg.Parent = humanoidRootPart
                        end
                    else
                        if bv then bv:Destroy() end
                        if bg then bg:Destroy() end
                    end
                elseif featureName == "NoClip" then
                    for _, part in ipairs(character:GetDescendants()) do
                        if part:IsA("BasePart") then
                            part.CanCollide = not state
                        end
                    end
                end
            end
            return true
        end
        return false
    end

    task.spawn(function()
        while true do
            task.wait(0.5)
            for plr, states in pairs(activeStates) do
                local character = plr.Character
                if character and states["NoClip"] then
                    for _, part in ipairs(character:GetDescendants()) do
                        if part:IsA("BasePart") then
                            part.CanCollide = false
                        end
                    end
                end
            end
        end
    end)

-- ============================================================================
-- 3. LÓGICA DO CLIENTE (Executado apenas no Client)
-- ============================================================================
elseif RunService:IsClient() then
    if not Config.IsAuthorized(player.UserId) then return end

    local JKAdminStorage = ReplicatedStorage:WaitForChild("JKAdminStorage", 5)
    if not JKAdminStorage then return end
    local RemoteEvents = JKAdminStorage:WaitForChild("RemoteEvents")
    local RequestToggleFeature = RemoteEvents:WaitForChild("RequestToggleFeature") :: RemoteFunction

    local aimLockEnabled = false
    local espEnabled = false
    local flyEnabled = false

    local fovRadius = 100
    local maxDistance = 500
    local aimSmoothness = 5

    -- Interface Gráfica
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "JKAdminUI"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = player:WaitForChild("PlayerGui")

    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 520, 0, 320)
    mainFrame.Position = UDim2.new(0.5, -260, 0.5, -160)
    mainFrame.BackgroundColor3 = Color3.fromRGB(18, 18, 24)
    mainFrame.BorderSizePixel = 0
    mainFrame.Active = true
    mainFrame.Draggable = true
    mainFrame.Parent = screenGui

    local uiCorner = Instance.new("UICorner")
    uiCorner.CornerRadius = UDim.new(0, 12)
    uiCorner.Parent = mainFrame

    local uiStroke = Instance.new("UIStroke")
    uiStroke.Color = Color3.fromRGB(138, 43, 226)
    uiStroke.Thickness = 2
    uiStroke.Parent = mainFrame

    local topBar = Instance.new("Frame")
    topBar.Size = UDim2.new(1, 0, 0, 40)
    topBar.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
    topBar.BorderSizePixel = 0
    topBar.Parent = mainFrame

    local topCorner = Instance.new("UICorner")
    topCorner.CornerRadius = UDim.new(0, 12)
    topCorner.Parent = topBar

    local titleLabel = Instance.new("TextLabel")
    titleLabel.Size = UDim2.new(1, -100, 1, 0)
    titleLabel.Position = UDim2.new(0, 15, 0, 0)
    titleLabel.BackgroundTransparency = 1
    titleLabel.Font = Enum.Font.GothamBold
    titleLabel.Text = "JK ADMIN AIM — TEST PANEL"
    titleLabel.TextColor3 = Color3.fromRGB(200, 150, 255)
    titleLabel.TextSize = 14
    titleLabel.TextXAlignment = Enum.TextXAlignment.Left
    titleLabel.Parent = topBar

    local minButton = Instance.new("TextButton")
    minButton.Size = UDim2.new(0, 30, 0, 30)
    minButton.Position = UDim2.new(1, -40, 0.5, -15)
    minButton.BackgroundColor3 = Color3.fromRGB(45, 35, 60)
    minButton.Font = Enum.Font.GothamBold
    minButton.Text = "-"
    minButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    minButton.TextSize = 16
    minButton.Parent = topBar

    local minCorner = Instance.new("UICorner")
    minCorner.CornerRadius = UDim.new(0, 6)
    minCorner.Parent = minButton

    local minimizedButton = Instance.new("TextButton")
    minimizedButton.Size = UDim2.new(0, 50, 0, 50)
    minimizedButton.Position = UDim2.new(0, 20, 0.8, -60)
    minimizedButton.BackgroundColor3 = Color3.fromRGB(18, 18, 24)
    minimizedButton.Font = Enum.Font.GothamBold
    minimizedButton.Text = "JK"
    minimizedButton.TextColor3 = Color3.fromRGB(138, 43, 226)
    minimizedButton.TextSize = 16
    minimizedButton.Visible = false
    minimizedButton.Parent = screenGui

    local minIconCorner = Instance.new("UICorner")
    minIconCorner.CornerRadius = UDim.new(1, 0)
    minIconCorner.Parent = minimizedButton

    local minIconStroke = Instance.new("UIStroke")
    minIconStroke.Color = Color3.fromRGB(138, 43, 226)
    minIconStroke.Thickness = 2
    minIconStroke.Parent = minimizedButton

    minButton.MouseButton1Click:Connect(function()
        mainFrame.Visible = false
        minimizedButton.Visible = true
    end)

    minimizedButton.MouseButton1Click:Connect(function()
        mainFrame.Visible = true
        minimizedButton.Visible = false
    end)

    local contentContainer = Instance.new("ScrollingFrame")
    contentContainer.Size = UDim2.new(1, -20, 1, -60)
    contentContainer.Position = UDim2.new(0, 10, 0, 50)
    contentContainer.BackgroundTransparency = 1
    contentContainer.CanvasSize = UDim2.new(0, 0, 0, 250)
    contentContainer.ScrollBarThickness = 4
    contentContainer.Parent = mainFrame

    local uiListLayout = Instance.new("UIListLayout")
    uiListLayout.Padding = UDim.new(0, 10)
    uiListLayout.SortOrder = Enum.SortOrder.LayoutOrder
    uiListLayout.Parent = contentContainer

    local function createToggleOption(name: string, callback: (boolean) -> ())
        local optionFrame = Instance.new("Frame")
        optionFrame.Size = UDim2.new(1, -10, 0, 45)
        optionFrame.BackgroundColor3 = Color3.fromRGB(28, 28, 38)
        optionFrame.BorderSizePixel = 0
        optionFrame.Parent = contentContainer

        local corner = Instance.new("UICorner")
        corner.CornerRadius = UDim.new(0, 8)
        corner.Parent = optionFrame

        local label = Instance.new("TextLabel")
        label.Size = UDim2.new(0.7, 0, 1, 0)
        label.Position = UDim2.new(0, 15, 0, 0)
        label.BackgroundTransparency = 1
        label.Font = Enum.Font.GothamMedium
        label.Text = name
        label.TextColor3 = Color3.fromRGB(220, 220, 230)
        label.TextSize = 14
        label.TextXAlignment = Enum.TextXAlignment.Left
        label.Parent = optionFrame

        local toggleBtn = Instance.new("TextButton")
        toggleBtn.Size = UDim2.new(0, 70, 0, 26)
        toggleBtn.Position = UDim2.new(1, -85, 0.5, -13)
        toggleBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 75)
        toggleBtn.Font = Enum.Font.GothamBold
        toggleBtn.Text = "OFF"
        toggleBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
        toggleBtn.TextSize = 12
        toggleBtn.Parent = optionFrame

        local btnCorner = Instance.new("UICorner")
        btnCorner.CornerRadius = UDim.new(0, 6)
        btnCorner.Parent = toggleBtn

        local state = false
        toggleBtn.MouseButton1Click:Connect(function()
            state = not state
            if state then
                toggleBtn.Text = "ON"
                toggleBtn.BackgroundColor3 = Color3.fromRGB(138, 43, 226)
                toggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
            else
                toggleBtn.Text = "OFF"
                toggleBtn.BackgroundColor3 = Color3.fromRGB(60, 60, 75)
                toggleBtn.TextColor3 = Color3.fromRGB(200, 200, 200)
            end
            callback(state)
        end)
    end

    createToggleOption("Aim Lock (Head / FOV)", function(state)
        aimLockEnabled = state
    end)

    createToggleOption("ESP (Boxes & Health)", function(state)
        espEnabled = state
    end)

    createToggleOption("Fly & NoClip (Admin Test)", function(state)
        flyEnabled = state
        RequestToggleFeature:InvokeServer("Fly", state)
        RequestToggleFeature:InvokeServer("NoClip", state)
    end)

    local fovCircle = Drawing.new("Circle")
    fovCircle.Visible = false
    fovCircle.Radius = fovRadius
    fovCircle.Color = Color3.fromRGB(138, 43, 226)
    fovCircle.Thickness = 1.5
    fovCircle.Filled = false
    fovCircle.Transparency = 0.8

    RunService.RenderStepped:Connect(function()
        if aimLockEnabled then
            fovCircle.Visible = true
            fovCircle.Position = UserInputService:GetMouseLocation()
        else
            fovCircle.Visible = false
        end

        if aimLockEnabled then
            local mousePos = UserInputService:GetMouseLocation()
            local closestTarget = nil
            local shortestDist = fovRadius

            for _, otherPlayer in ipairs(Players:GetPlayers()) do
                if otherPlayer ~= player and otherPlayer.Character then
                    local char = otherPlayer.Character
                    local humanoid = char:FindFirstChildOfClass("Humanoid")
                    local head = char:FindFirstChild("Head")

                    if humanoid and humanoid.Health > 0 and head then
                        local isAlly = (player.Team and otherPlayer.Team == player.Team)
                        if not isAlly then
                            local screenPos, onScreen = camera:WorldToViewportPoint(head.Position)
                            if onScreen then
                                local screenDist = (Vector2.new(screenPos.X, screenPos.Y) - mousePos).Magnitude
                                local worldDist = (head.Position - camera.CFrame.Position).Magnitude

                                if screenDist < shortestDist and worldDist <= maxDistance then
                                    local rayParams = RaycastParams.new()
                                    rayParams.FilterType = Enum.RaycastFilterType.Exclude
                                    rayParams.FilterDescendantsInstances = {player.Character, char}
                                    local rayResult = Workspace:Raycast(camera.CFrame.Position, head.Position - camera.CFrame.Position, rayParams)

                                    if not rayResult then
                                        closestTarget = head
                                        shortestDist = screenDist
                                    end
                                end
                            end
                        end
                    end
                end
            end

            if closestTarget then
                local targetCFrame = CFrame.new(camera.CFrame.Position, closestTarget.Position)
                camera.CFrame = camera.CFrame:Lerp(targetCFrame, aimSmoothness * RunService.RenderStepped:Wait())
            end
        end
    end)

    local espCache = {}
    local function removeEsp(plr)
        if espCache[plr] then
            for _, obj in pairs(espCache[plr]) do
                if typeof(obj) == "Instance" then obj:Destroy()
                elseif typeof(obj) == "table" and obj.Remove then obj:Remove() end
            end
            espCache[plr] = nil
        end
    end

    Players.PlayerRemoving:Connect(removeEsp)

    RunService.RenderStepped:Connect(function()
        if not espEnabled then
            for plr, _ in pairs(espCache) do removeEsp(plr) end
            return
        end

        for _, otherPlayer in ipairs(Players:GetPlayers()) do
            if otherPlayer ~= player then
                local char = otherPlayer.Character
                local rootPart = char and char:FindFirstChild("HumanoidRootPart")
                local humanoid = char and char:FindFirstChildOfClass("Humanoid")

                if char and rootPart and humanoid and humanoid.Health > 0 then
                    if not espCache[otherPlayer] then
                        espCache[otherPlayer] = {
                            box = Drawing.new("Square"),
                            healthBar = Drawing.new("Line")
                        }
                        espCache[otherPlayer].box.Thickness = 1.5
                        espCache[otherPlayer].box.Color = Color3.fromRGB(138, 43, 226)
                        espCache[otherPlayer].box.Filled = false
                    end

                    local box = espCache[otherPlayer].box
                    local healthBar = espCache[otherPlayer].healthBar

                    local vector, onScreen = camera:WorldToViewportPoint(rootPart.Position)
                    if onScreen then
                        local size = Vector2.new(2000 / vector.Z, 3000 / vector.Z)
                        box.Size = size
                        box.Position = Vector2.new(vector.X - size.X / 2, vector.Y - size.Y / 2)
                        box.Visible = true

                        healthBar.From = Vector2.new(box.Position.X - 6, box.Position.Y + box.Size.Y)
                        healthBar.To = Vector2.new(healthBar.From.X, box.Position.Y + (box.Size.Y * (1 - (humanoid.Health / humanoid.MaxHealth))))
                        healthBar.Color = Color3.fromRGB(0, 255, 100)
                        healthBar.Thickness = 2
                        healthBar.Visible = true
                    else
                        box.Visible = false
                        healthBar.Visible = false
                    end
                else
                    removeEsp(otherPlayer)
                end
            end
        end
    end)

    RunService.Stepped:Connect(function()
        if flyEnabled and player.Character then
            local rootPart = player.Character:FindFirstChild("HumanoidRootPart")
            if rootPart then
                local bv = rootPart:FindFirstChild("JKAdminBodyVelocity")
                local bg = rootPart:FindFirstChild("JKAdminBodyGyro")
                
                if bv and bg then
                    bg.CFrame = camera.CFrame
                    local moveDir = Vector3.zero
                    
                    if UserInputService:IsKeyDown(Enum.KeyCode.W) then moveDir += camera.CFrame.LookVector end
                    if UserInputService:IsKeyDown(Enum.KeyCode.S) then moveDir -= camera.CFrame.LookVector end
                    if UserInputService:IsKeyDown(Enum.KeyCode.D) then moveDir += camera.CFrame.RightVector end
                    if UserInputService:IsKeyDown(Enum.KeyCode.A) then moveDir -= camera.CFrame.RightVector end
                    
                    bv.Velocity = moveDir * 50
                end
            end
        end
    end)
end
