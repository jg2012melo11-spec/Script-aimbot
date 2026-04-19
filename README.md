local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

local aimbotActive = false
local aimbotFOV = 150
local aimbotSmoothness = 0.3
local currentTarget = nil

local espActive = false
local espLinesActive = false

local mainUIVisible = true
local loginUIVisible = true

local loginScreen = nil
local mainUI = nil
local fovCircle = nil
local espObjects = {}
local espLines = {}

local dragging = false
local dragStart = nil
local startPos = nil

local function playClickSound()
    local sound = Instance.new("Sound")
    sound.SoundId = "rbxasset://sounds/ui_button.mp3"
    sound.Volume = 0.3
    sound.Parent = player.Character or workspace
    sound:Play()
    game:Debris:AddItem(sound, 1)
end

local function getHeadPosition(targetPlayer)
    if not targetPlayer or not targetPlayer.Character then
        return nil
    end
    
    local character = targetPlayer.Character
    local head = character:FindFirstChild("Head")
    local humanoid = character:FindFirstChild("Humanoid")
    
    if not head or not humanoid or humanoid.Health <= 0 then
        return nil
    end
    
    return head.Position
end

local function worldToScreen(worldPosition)
    if not worldPosition then
        return nil
    end
    
    local vector, onScreen = camera:WorldToScreenPoint(worldPosition)
    if onScreen then
        return Vector2.new(vector.X, vector.Y)
    end
    return nil
end

local function isInFOV(screenPosition, centerScreen)
    local distance = (screenPosition - centerScreen).Magnitude
    return distance <= aimbotFOV
end

local function findBestTarget()
    if not aimbotActive then
        return nil
    end
    
    local centerScreen = Vector2.new(camera.ViewportSize.X / 2, camera.ViewportSize.Y / 2)
    local bestTarget = nil
    local bestDistance = aimbotFOV + 1
    
    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player and otherPlayer.Character then
            local headPos = getHeadPosition(otherPlayer)
            if headPos then
                local screenPos = worldToScreen(headPos)
                if screenPos and isInFOV(screenPos, centerScreen) then
                    local distanceToCenter = (screenPos - centerScreen).Magnitude
                    if distanceToCenter < bestDistance then
                        bestDistance = distanceToCenter
                        bestTarget = otherPlayer
                    end
                end
            end
        end
    end
    
    return bestTarget
end

local function updateAimbot()
    if not aimbotActive then
        if camera.CameraType == Enum.CameraType.Scriptable then
            camera.CameraType = Enum.CameraType.Custom
        end
        currentTarget = nil
        return
    end
    
    local bestTarget = findBestTarget()
    
    if bestTarget then
        currentTarget = bestTarget
        local headPos = getHeadPosition(currentTarget)
        
        if headPos then
            camera.CameraType = Enum.CameraType.Scriptable
            
            local cameraPos = camera.CFrame.Position
            local targetCFrame = CFrame.lookAt(cameraPos, headPos)
            
            if aimbotSmoothness > 0 then
                local alpha = math.max(0, math.min(1, 1 - aimbotSmoothness))
                camera.CFrame = camera.CFrame:Lerp(targetCFrame, alpha)
            else
                camera.CFrame = targetCFrame
            end
        end
    else
        if currentTarget then
            currentTarget = nil
            camera.CameraType = Enum.CameraType.Custom
        end
    end
end

local function createESPBox(targetPlayer)
    if not espActive or not targetPlayer.Character then
        return nil
    end
    
    if espObjects[targetPlayer] then
        return espObjects[targetPlayer]
    end
    
    local espGui = Instance.new("BillboardGui")
    espGui.Name = "ESPBox"
    espGui.AlwaysOnTop = true
    espGui.Size = UDim2.new(0, 60, 0, 80)
    espGui.StudsOffset = Vector3.new(0, 2.5, 0)
    espGui.Adornee = targetPlayer.Character
    espGui.Parent = targetPlayer.Character
    
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, 0, 1, 0)
    frame.BackgroundTransparency = 0.7
    frame.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
    frame.BorderSizePixel = 2
    frame.BorderColor3 = Color3.fromRGB(255, 0, 0)
    frame.Parent = espGui
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 5)
    corner.Parent = frame
    
    local nameLabel = Instance.new("TextLabel")
    nameLabel.Size = UDim2.new(1, 0, 0, 20)
    nameLabel.Position = UDim2.new(0, 0, 1, 0)
    nameLabel.BackgroundTransparency = 1
    nameLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    nameLabel.TextStrokeTransparency = 0.3
    nameLabel.Font = Enum.Font.Gotham
    nameLabel.TextSize = 12
    nameLabel.Text = targetPlayer.Name
    nameLabel.Parent = espGui
    
    espObjects[targetPlayer] = espGui
    return espGui
end

local function updateESPDistance(targetPlayer)
    if not espActive or not espObjects[targetPlayer] then
        return
    end
    
    local espGui = espObjects[targetPlayer]
    local nameLabel = espGui:FindFirstChild("TextLabel")
    
    if nameLabel and targetPlayer.Character then
        local headPos = getHeadPosition(targetPlayer)
        if headPos and camera.CFrame then
            local distance = (headPos - camera.CFrame.Position).Magnitude
            nameLabel.Text = string.format("%s | %.0fm", targetPlayer.Name, distance)
        end
    end
end

local function removeESP(targetPlayer)
    if espObjects[targetPlayer] then
        espObjects[targetPlayer]:Destroy()
        espObjects[targetPlayer] = nil
    end
end

local function createESPLine(targetPlayer)
    if not espLinesActive or not targetPlayer.Character then
        return nil
    end
    
    if espLines[targetPlayer] then
        return espLines[targetPlayer]
    end
    
    local linePart = Instance.new("Part")
    linePart.Name = "ESPLine"
    linePart.Size = Vector3.new(0.1, 0.1, 0.1)
    linePart.Anchored = true
    linePart.CanCollide = false
    linePart.Transparency = 0.4
    linePart.BrickColor = BrickColor.new("Bright red")
    linePart.Material = Enum.Material.Neon
    linePart.Parent = workspace
    
    espLines[targetPlayer] = linePart
    return linePart
end

local function updateESPLine(targetPlayer)
    if not espLinesActive or not espLines[targetPlayer] then
        return
    end
    
    local linePart = espLines[targetPlayer]
    local headPos = getHeadPosition(targetPlayer)
    
    if headPos and camera.CFrame then
        local startPos = camera.CFrame.Position
        local distance = (headPos - startPos).Magnitude
        
        linePart.Size = Vector3.new(0.05, 0.05, distance)
        linePart.CFrame = CFrame.lookAt(startPos, headPos) * CFrame.new(0, 0, -distance/2)
        linePart.Transparency = 0.3
    else
        linePart.Transparency = 1
    end
end

local function removeESPLine(targetPlayer)
    if espLines[targetPlayer] then
        espLines[targetPlayer]:Destroy()
        espLines[targetPlayer] = nil
    end
end

local function updateESP()
    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player then
            if espActive and otherPlayer.Character then
                createESPBox(otherPlayer)
                updateESPDistance(otherPlayer)
                
                if espLinesActive then
                    createESPLine(otherPlayer)
                    updateESPLine(otherPlayer)
                else
                    removeESPLine(otherPlayer)
                end
            else
                removeESP(otherPlayer)
                removeESPLine(otherPlayer)
            end
        end
    end
end

local function createFOVCircle()
    local fovGui = Instance.new("ScreenGui")
    fovGui.Name = "FOVCircleGUI"
    fovGui.ResetOnSpawn = false
    fovGui.IgnoreGuiInset = true
    fovGui.Parent = player:WaitForChild("PlayerGui")
    
    local circleFrame = Instance.new("Frame")
    circleFrame.Name = "FOVCircle"
    circleFrame.AnchorPoint = Vector2.new(0.5, 0.5)
    circleFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
    circleFrame.Size = UDim2.new(0, aimbotFOV * 2, 0, aimbotFOV * 2)
    circleFrame.BackgroundTransparency = 1
    circleFrame.BorderSizePixel = 0
    circleFrame.Parent = fovGui
    
    local stroke = Instance.new("UIStroke")
    stroke.Thickness = 3
    stroke.Color = Color3.fromRGB(0, 255, 255)
    stroke.Transparency = 0.2
    stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    stroke.Parent = circleFrame
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(1, 0)
    corner.Parent = circleFrame
    
    local tweenInfo = TweenInfo.new(1, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true)
    local pulseTween = TweenService:Create(stroke, tweenInfo, {Transparency = 0.1})
    pulseTween:Play()
    
    return fovGui
end

local function updateFOVCircleSize()
    if fovCircle then
        local circleFrame = fovCircle:FindFirstChild("FOVCircle")
        if circleFrame then
            circleFrame.Size = UDim2.new(0, aimbotFOV * 2, 0, aimbotFOV * 2)
        end
    end
end

local function removeFOVCircle()
    if fovCircle then
        fovCircle:Destroy()
        fovCircle = nil
    end
end

local function createMainUI()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "MainUI"
    screenGui.ResetOnSpawn = false
    screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
    screenGui.Parent = player:WaitForChild("PlayerGui")
    
    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 380, 0, 480)
    mainFrame.Position = UDim2.new(0.5, -190, 0.5, -240)
    mainFrame.BackgroundColor3 = Color3.fromRGB(15, 15, 25)
    mainFrame.BackgroundTransparency = 0.1
    mainFrame.BorderSizePixel = 0
    mainFrame.Parent = screenGui
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 12)
    corner.Parent = mainFrame
    
    local stroke = Instance.new("UIStroke")
    stroke.Thickness = 1
    stroke.Color = Color3.fromRGB(0, 255, 255)
    stroke.Transparency = 0.5
    stroke.Parent = mainFrame
    
    local titleBar = Instance.new("Frame")
    titleBar.Name = "TitleBar"
    titleBar.Size = UDim2.new(1, 0, 0, 40)
    titleBar.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
    titleBar.BackgroundTransparency = 0.3
    titleBar.BorderSizePixel = 0
    titleBar.Parent = mainFrame
    
    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 12)
    titleCorner.Parent = titleBar
    
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -80, 1, 0)
    title.Position = UDim2.new(0, 15, 0, 0)
    title.BackgroundTransparency = 1
    title.Text = "⚡ CYBER AIMBOT v1.0"
    title.TextColor3 = Color3.fromRGB(0, 255, 255)
    title.TextSize = 15
    title.Font = Enum.Font.GothamBold
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = titleBar
    
    local minButton = Instance.new("TextButton")
    minButton.Size = UDim2.new(0, 35, 1, 0)
    minButton.Position = UDim2.new(1, -70, 0, 0)
    minButton.BackgroundTransparency = 1
    minButton.Text = "─"
    minButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    minButton.TextSize = 20
    minButton.Font = Enum.Font.GothamBold
    minButton.Parent = titleBar
    
    local closeButton = Instance.new("TextButton")
    closeButton.Size = UDim2.new(0, 35, 1, 0)
    closeButton.Position = UDim2.new(1, -35, 0, 0)
    closeButton.BackgroundTransparency = 1
    closeButton.Text = "✕"
    closeButton.TextColor3 = Color3.fromRGB(255, 80, 80)
    closeButton.TextSize = 16
    closeButton.Font = Enum.Font.GothamBold
    closeButton.Parent = titleBar
    
    local scrollFrame = Instance.new("ScrollingFrame")
    scrollFrame.Size = UDim2.new(1, -20, 1, -60)
    scrollFrame.Position = UDim2.new(0, 10, 0, 50)
    scrollFrame.BackgroundTransparency = 1
    scrollFrame.BorderSizePixel = 0
    scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 0)
    scrollFrame.ScrollBarThickness = 4
    scrollFrame.ScrollBarImageColor3 = Color3.fromRGB(0, 255, 255)
    scrollFrame.Parent = mainFrame
    
    local layout = Instance.new("UIListLayout")
    layout.Padding = UDim.new(0, 12)
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Parent = scrollFrame
    
    local aimbotSection = Instance.new("Frame")
    aimbotSection.Size = UDim2.new(1, 0, 0, 110)
    aimbotSection.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
    aimbotSection.BackgroundTransparency = 0.5
    aimbotSection.BorderSizePixel = 0
    aimbotSection.Parent = scrollFrame
    
    local sectionCorner = Instance.new("UICorner")
    sectionCorner.CornerRadius = UDim.new(0, 8)
    sectionCorner.Parent = aimbotSection
    
    local sectionTitle = Instance.new("TextLabel")
    sectionTitle.Size = UDim2.new(1, -20, 0, 30)
    sectionTitle.Position = UDim2.new(0, 10, 0, 5)
    sectionTitle.BackgroundTransparency = 1
    sectionTitle.Text = "🎯 AIMBOT"
    sectionTitle.TextColor3 = Color3.fromRGB(0, 255, 255)
    sectionTitle.TextSize = 14
    sectionTitle.Font = Enum.Font.GothamBold
    sectionTitle.TextXAlignment = Enum.TextXAlignment.Left
    sectionTitle.Parent = aimbotSection
    
    local aimbotToggle = Instance.new("TextButton")
    aimbotToggle.Size = UDim2.new(0, 120, 0, 40)
    aimbotToggle.Position = UDim2.new(0, 10, 0, 40)
    aimbotToggle.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    aimbotToggle.Text = "🔴 DESLIGADO"
    aimbotToggle.TextColor3 = Color3.fromRGB(255, 255, 255)
    aimbotToggle.TextSize = 14
    aimbotToggle.Font = Enum.Font.GothamSemibold
    aimbotToggle.Parent = aimbotSection
    
    local toggleCorner = Instance.new("UICorner")
    toggleCorner.CornerRadius = UDim.new(0, 6)
    toggleCorner.Parent = aimbotToggle
    
    local settingsSection = Instance.new("Frame")
    settingsSection.Size = UDim2.new(1, 0, 0, 140)
    settingsSection.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
    settingsSection.BackgroundTransparency = 0.5
    settingsSection.BorderSizePixel = 0
    settingsSection.Parent = scrollFrame
    
    local settingsCorner = Instance.new("UICorner")
    settingsCorner.CornerRadius = UDim.new(0, 8)
    settingsCorner.Parent = settingsSection
    
    local settingsTitle = Instance.new("TextLabel")
    settingsTitle.Size = UDim2.new(1, -20, 0, 30)
    settingsTitle.Position = UDim2.new(0, 10, 0, 5)
    settingsTitle.BackgroundTransparency = 1
    settingsTitle.Text = "⚙️ CONFIGURAÇÕES"
    settingsTitle.TextColor3 = Color3.fromRGB(0, 255, 255)
    settingsTitle.TextSize = 14
    settingsTitle.Font = Enum.Font.GothamBold
    settingsTitle.TextXAlignment = Enum.TextXAlignment.Left
    settingsTitle.Parent = settingsSection
    
    local fovLabel = Instance.new("TextLabel")
    fovLabel.Size = UDim2.new(1, -20, 0, 25)
    fovLabel.Position = UDim2.new(0, 10, 0, 40)
    fovLabel.BackgroundTransparency = 1
    fovLabel.Text = "Raio do Círculo (FOV): " .. aimbotFOV .. "px"
    fovLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    fovLabel.TextSize = 12
    fovLabel.TextXAlignment = Enum.TextXAlignment.Left
    fovLabel.Parent = settingsSection
    
    local fovSlider = Instance.new("TextBox")
    fovSlider.Size = UDim2.new(1, -20, 0, 30)
    fovSlider.Position = UDim2.new(0, 10, 0, 65)
    fovSlider.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
    fovSlider.Text = tostring(aimbotFOV)
    fovSlider.TextColor3 = Color3.fromRGB(0, 255, 255)
    fovSlider.PlaceholderText = "50 - 500"
    fovSlider.Parent = settingsSection
    
    local fovSliderCorner = Instance.new("UICorner")
    fovSliderCorner.CornerRadius = UDim.new(0, 4)
    fovSliderCorner.Parent = fovSlider
    
    local smoothLabel = Instance.new("TextLabel")
    smoothLabel.Size = UDim2.new(1, -20, 0, 25)
    smoothLabel.Position = UDim2.new(0, 10, 0, 105)
    smoothLabel.BackgroundTransparency = 1
    smoothLabel.Text = "Suavidade da Mira: " .. string.format("%.1f", aimbotSmoothness)
    smoothLabel.TextColor3 = Color3.fromRGB(200, 200, 200)
    smoothLabel.TextSize = 12
    smoothLabel.TextXAlignment = Enum.TextXAlignment.Left
    smoothLabel.Parent = settingsSection
    
    local smoothSlider = Instance.new("TextBox")
    smoothSlider.Size = UDim2.new(1, -20, 0, 30)
    smoothSlider.Position = UDim2.new(0, 10, 0, 130)
    smoothSlider.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
    smoothSlider.Text = string.format("%.1f", aimbotSmoothness)
    smoothSlider.TextColor3 = Color3.fromRGB(0, 255, 255)
    smoothSlider.PlaceholderText = "0 - 1"
    smoothSlider.Parent = settingsSection
    
    local smoothSliderCorner = Instance.new("UICorner")
    smoothSliderCorner.CornerRadius = UDim.new(0, 4)
    smoothSliderCorner.Parent = smoothSlider
    
    local espSection = Instance.new("Frame")
    espSection.Size = UDim2.new(1, 0, 0, 100)
    espSection.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
    espSection.BackgroundTransparency = 0.5
    espSection.BorderSizePixel = 0
    espSection.Parent = scrollFrame
    
    local espCorner = Instance.new("UICorner")
    espCorner.CornerRadius = UDim.new(0, 8)
    espCorner.Parent = espSection
    
    local espTitle = Instance.new("TextLabel")
    espTitle.Size = UDim2.new(1, -20, 0, 30)
    espTitle.Position = UDim2.new(0, 10, 0, 5)
    espTitle.BackgroundTransparency = 1
    espTitle.Text = "👁️ ESP"
    espTitle.TextColor3 = Color3.fromRGB(0, 255, 255)
    espTitle.TextSize = 14
    espTitle.Font = Enum.Font.GothamBold
    espTitle.TextXAlignment = Enum.TextXAlignment.Left
    espTitle.Parent = espSection
    
    local espToggle = Instance.new("TextButton")
    espToggle.Size = UDim2.new(0, 110, 0, 40)
    espToggle.Position = UDim2.new(0, 10, 0, 40)
    espToggle.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    espToggle.Text = "🔴 ESP OFF"
    espToggle.TextColor3 = Color3.fromRGB(255, 255, 255)
    espToggle.TextSize = 13
    espToggle.Font = Enum.Font.GothamSemibold
    espToggle.Parent = espSection
    
    local espToggleCorner = Instance.new("UICorner")
    espToggleCorner.CornerRadius = UDim.new(0, 6)
    espToggleCorner.Parent = espToggle
    
    local linesToggle = Instance.new("TextButton")
    linesToggle.Size = UDim2.new(0, 110, 0, 40)
    linesToggle.Position = UDim2.new(0, 130, 0, 40)
    linesToggle.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
    linesToggle.Text = "🔴 LINES OFF"
    linesToggle.TextColor3 = Color3.fromRGB(255, 255, 255)
    linesToggle.TextSize = 13
    linesToggle.Font = Enum.Font.GothamSemibold
    linesToggle.Parent = espSection
    
    local linesToggleCorner = Instance.new("UICorner")
    linesToggleCorner.CornerRadius = UDim.new(0, 6)
    linesToggleCorner.Parent = linesToggle
    
    local infoSection = Instance.new("Frame")
    infoSection.Size = UDim2.new(1, 0, 0, 60)
    infoSection.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
    infoSection.BackgroundTransparency = 0.5
    infoSection.BorderSizePixel = 0
    infoSection.Parent = scrollFrame
    
    local infoCorner = Instance.new("UICorner")
    infoCorner.CornerRadius = UDim.new(0, 8)
    infoCorner.Parent = infoSection
    
    local infoText = Instance.new("TextLabel")
    infoText.Size = UDim2.new(1, -20, 1, 0)
    infoText.Position = UDim2.new(0, 10, 0, 0)
    infoText.BackgroundTransparency = 1
    infoText.Text = "Keybind: INSERT para abrir/fechar\nAlvo: Nenhum"
    infoText.TextColor3 = Color3.fromRGB(150, 150, 150)
    infoText.TextSize = 11
    infoText.TextXAlignment = Enum.TextXAlignment.Left
    infoText.TextYAlignment = Enum.TextYAlignment.Center
    infoText.Parent = infoSection
    
    local function updateCanvas()
        local height = 0
        for _, child in ipairs(scrollFrame:GetChildren()) do
            if child:IsA("Frame") then
                height = height + child.AbsoluteSize.Y + 12
            end
        end
        scrollFrame.CanvasSize = UDim2.new(0, 0, 0, height + 20)
    end
    
    layout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(updateCanvas)
    task.wait(0.1)
    updateCanvas()
    
    local dragging = false
    local dragStartPos = nil
    local frameStartPos = nil
    
    titleBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStartPos = input.Position
            frameStartPos = mainFrame.Position
        end
    end)
    
    UserInputService.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local delta = input.Position - dragStartPos
            mainFrame.Position = UDim2.new(frameStartPos.X.Scale, frameStartPos.X.Offset + delta.X, frameStartPos.Y.Scale, frameStartPos.Y.Offset + delta.Y)
        end
    end)
    
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end)
    
    aimbotToggle.MouseButton1Click:Connect(function()
        playClickSound()
        aimbotActive = not aimbotActive
        
        if aimbotActive then
            aimbotToggle.Text = "🟢 LIGADO"
            aimbotToggle.BackgroundColor3 = Color3.fromRGB(0, 100, 0)
            fovCircle = createFOVCircle()
        else
            aimbotToggle.Text = "🔴 DESLIGADO"
            aimbotToggle.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
            removeFOVCircle()
            camera.CameraType = Enum.CameraType.Custom
        end
    end)
    
    fovSlider.FocusLost:Connect(function()
        local newValue = tonumber(fovSlider.Text)
        if newValue then
            aimbotFOV = math.clamp(newValue, 50, 500)
            fovSlider.Text = tostring(aimbotFOV)
            fovLabel.Text = "Raio do Círculo (FOV): " .. aimbotFOV .. "px"
            updateFOVCircleSize()
        else
            fovSlider.Text = tostring(aimbotFOV)
        end
    end)
    
    smoothSlider.FocusLost:Connect(function()
        local newValue = tonumber(smoothSlider.Text)
        if newValue then
            aimbotSmoothness = math.clamp(newValue, 0, 1)
            smoothSlider.Text = string.format("%.1f", aimbotSmoothness)
            smoothLabel.Text = "Suavidade da Mira: " .. string.format("%.1f", aimbotSmoothness)
        else
            smoothSlider.Text = string.format("%.1f", aimbotSmoothness)
        end
    end)
    
    espToggle.MouseButton1Click:Connect(function()
        playClickSound()
        espActive = not espActive
        
        if espActive then
            espToggle.Text = "🟢 ESP ON"
            espToggle.BackgroundColor3 = Color3.fromRGB(0, 100, 0)
        else
            espToggle.Text = "🔴 ESP OFF"
            espToggle.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
            for p, _ in pairs(espObjects) do
                removeESP(p)
            end
            for p, _ in pairs(espLines) do
                removeESPLine(p)
            end
        end
    end)
    
    linesToggle.MouseButton1Click:Connect(function()
        playClickSound()
        espLinesActive = not espLinesActive
        
        if espLinesActive then
            linesToggle.Text = "🟢 LINES ON"
            linesToggle.BackgroundColor3 = Color3.fromRGB(0, 100, 0)
        else
            linesToggle.Text = "🔴 LINES OFF"
            linesToggle.BackgroundColor3 = Color3.fromRGB(50, 50, 60)
            for p, _ in pairs(espLines) do
                removeESPLine(p)
            end
        end
    end)
    
    minButton.MouseButton1Click:Connect(function()
        playClickSound()
        mainUIVisible = not mainUIVisible
        mainFrame.Visible = mainUIVisible
    end)
    
    closeButton.MouseButton1Click:Connect(function()
        playClickSound()
        mainFrame.Visible = false
        mainUIVisible = false
    end)
    
    RunService.RenderStepped:Connect(function()
        if mainFrame.Visible and infoText then
            infoText.Text = "Keybind: INSERT para abrir/fechar\nAlvo: " .. (currentTarget and currentTarget.Name or "Nenhum")
        end
    end)
    
    return screenGui
end

local function createLoginScreen()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "LoginScreen"
    screenGui.ResetOnSpawn = false
    screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
    screenGui.Parent = player:WaitForChild("PlayerGui")
    
    local background = Instance.new("Frame")
    background.Size = UDim2.new(1, 0, 1, 0)
    background.BackgroundColor3 = Color3.fromRGB(0, 0, 0)
    background.BackgroundTransparency = 0.3
    background.Parent = screenGui
    
    local container = Instance.new("Frame")
    container.Size = UDim2.new(0, 400, 0, 350)
    container.Position = UDim2.new(0.5, -200, 0.5, -175)
    container.BackgroundColor3 = Color3.fromRGB(15, 15, 25)
    container.BackgroundTransparency = 0.1
    container.BorderSizePixel = 0
    container.Parent = screenGui
    
    local containerCorner = Instance.new("UICorner")
    containerCorner.CornerRadius = UDim.new(0, 16)
    containerCorner.Parent = container
    
    local containerStroke = Instance.new("UIStroke")
    containerStroke.Thickness = 1.5
    containerStroke.Color = Color3.fromRGB(0, 255, 255)
    containerStroke.Transparency = 0.6
    containerStroke.Parent = container
    
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, 0, 0, 60)
    title.Position = UDim2.new(0, 0, 0, 20)
    title.BackgroundTransparency = 1
    title.Text = "⚡ CYBER ACCESS ⚡"
    title.TextColor3 = Color3.fromRGB(0, 255, 255)
    title.TextSize = 24
    title.Font = Enum.Font.GothamBold
    title.Parent = container
    
    local subtitle = Instance.new("TextLabel")
    subtitle.Size = UDim2.new(1, 0, 0, 30)
    subtitle.Position = UDim2.new(0, 0, 0, 80)
    subtitle.BackgroundTransparency = 1
    subtitle.Text = "Insira a chave de acesso"
    subtitle.TextColor3 = Color3.fromRGB(150, 150, 150)
    subtitle.TextSize = 14
    subtitle.Font = Enum.Font.Gotham
    subtitle.Parent = container
    
    local passwordBox = Instance.new("TextBox")
    passwordBox.Size = UDim2.new(0, 250, 0, 45)
    passwordBox.Position = UDim2.new(0.5, -125, 0, 140)
    passwordBox.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
    passwordBox.BackgroundTransparency = 0.5
    passwordBox.PlaceholderText = "Digite a Key"
    passwordBox.Text = ""
    passwordBox.TextColor3 = Color3.fromRGB(0, 255, 255)
    passwordBox.TextSize = 16
    passwordBox.Font = Enum.Font.Gotham
    passwordBox.Parent = container
    
    local passCorner = Instance.new("UICorner")
    passCorner.CornerRadius = UDim.new(0, 8)
    passCorner.Parent = passwordBox
    
    local loginButton = Instance.new("TextButton")
    loginButton.Size = UDim2.new(0, 200, 0, 50)
    loginButton.Position = UDim2.new(0.5, -100, 0, 220)
    loginButton.BackgroundColor3 = Color3.fromRGB(0, 150, 200)
    loginButton.Text = "ACESSAR"
    loginButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    loginButton.TextSize = 16
    loginButton.Font = Enum.Font.GothamBold
    loginButton.Parent = container
    
    local loginCorner = Instance.new("UICorner")
    loginCorner.CornerRadius = UDim.new(0, 8)
    loginCorner.Parent = loginButton
    
    local errorLabel = Instance.new("TextLabel")
    errorLabel.Size = UDim2.new(1, -40, 0, 30)
    errorLabel.Position = UDim2.new(0, 20, 0, 285)
    errorLabel.BackgroundTransparency = 1
    errorLabel.Text = ""
    errorLabel.TextColor3 = Color3.fromRGB(255, 50, 50)
    errorLabel.TextSize = 12
    errorLabel.Font = Enum.Font.Gotham
    errorLabel.Parent = container
    
    loginButton.MouseButton1Click:Connect(function()
        playClickSound()
        if passwordBox.Text == "5609" then
            screenGui:Destroy()
            mainUI = createMainUI()
            
            RunService.RenderStepped:Connect(function()
                updateAimbot()
                if espActive or espLinesActive then
                    updateESP()
                end
            end)
        else
            errorLabel.Text = "❌ Key Inválida! Tente novamente."
            passwordBox.Text = ""
            task.wait(2)
            errorLabel.Text = ""
        end
    end)
    
    return screenGui
end

loginScreen = createLoginScreen()

UserInputService.InputBegan:Connect(function(input, gameProcessed)
    if gameProcessed then return end
    
    if input.KeyCode == Enum.KeyCode.Insert then
        if mainUI then
            local mainFrame = mainUI:FindFirstChild("MainFrame")
            if mainFrame then
                mainFrame.Visible = not mainFrame.Visible
            end
        end
    end
end)
