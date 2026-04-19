--[[
    Script: Advanced Aimbot & ESP System - VERSÃO CORRIGIDA
    Autor: Dark Aura BR 🇧🇷
    Descrição: Sistema completo de aimbot com suavização e ESP
]]

-- ============================================
-- SEÇÃO: VERIFICAÇÃO INICIAL
-- ============================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer

-- Variáveis de controle
local AimbotEnabled = false
local ESPEnabled = false
local ESPLinesEnabled = false
local FOVRadius = 120
local Smoothness = 0.3
local CurrentTarget = nil

-- Objetos da UI
local ScreenGui = nil
local MainFrame = nil
local UIConnections = {}

-- Objetos do ESP
local ESPObjects = {}

-- ============================================
-- SEÇÃO: CRIAÇÃO DA INTERFACE (SEM BIBLIOTECA EXTERNA)
-- ============================================

local function CreateCustomUI()
    -- Criar ScreenGui
    ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "DarkAuraGUI"
    ScreenGui.Parent = game:GetService("CoreGui")
    
    -- Criar botão flutuante para abrir o menu
    local ToggleButton = Instance.new("ImageButton")
    ToggleButton.Size = UDim2.new(0, 50, 0, 50)
    ToggleButton.Position = UDim2.new(0, 10, 0, 100)
    ToggleButton.BackgroundColor3 = Color3.fromRGB(30, 30, 30)
    ToggleButton.BackgroundTransparency = 0.2
    ToggleButton.BorderSizePixel = 0
    ToggleButton.Image = "rbxassetid://3926305904"
    ToggleButton.Parent = ScreenGui
    
    -- Adicionar corner ao botão
    local ToggleCorner = Instance.new("UICorner")
    ToggleCorner.CornerRadius = UDim.new(1, 0)
    ToggleCorner.Parent = ToggleButton
    
    -- Criar frame principal (menu)
    MainFrame = Instance.new("Frame")
    MainFrame.Size = UDim2.new(0, 350, 0, 500)
    MainFrame.Position = UDim2.new(0.5, -175, 0.5, -250)
    MainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 35)
    MainFrame.BackgroundTransparency = 0.05
    MainFrame.BorderSizePixel = 0
    MainFrame.Visible = false
    MainFrame.Parent = ScreenGui
    
    -- Adicionar fundo semi-transparente
    local MainCorner = Instance.new("UICorner")
    MainCorner.CornerRadius = UDim.new(0, 8)
    MainCorner.Parent = MainFrame
    
    -- Título da janela
    local TitleBar = Instance.new("Frame")
    TitleBar.Size = UDim2.new(1, 0, 0, 40)
    TitleBar.BackgroundColor3 = Color3.fromRGB(40, 40, 55)
    TitleBar.BorderSizePixel = 0
    TitleBar.Parent = MainFrame
    
    local TitleCorner = Instance.new("UICorner")
    TitleCorner.CornerRadius = UDim.new(0, 8)
    TitleCorner.Parent = TitleBar
    
    local TitleText = Instance.new("TextLabel")
    TitleText.Size = UDim2.new(1, -60, 1, 0)
    TitleText.Position = UDim2.new(0, 10, 0, 0)
    TitleText.BackgroundTransparency = 1
    TitleText.Text = "Dark Aura BR 🇧🇷 - Aimbot System"
    TitleText.TextColor3 = Color3.fromRGB(255, 255, 255)
    TitleText.TextSize = 16
    TitleText.TextXAlignment = Enum.TextXAlignment.Left
    TitleText.Font = Enum.Font.GothamBold
    TitleText.Parent = TitleBar
    
    -- Botão minimizar
    local MinimizeBtn = Instance.new("TextButton")
    MinimizeBtn.Size = UDim2.new(0, 30, 1, 0)
    MinimizeBtn.Position = UDim2.new(1, -60, 0, 0)
    MinimizeBtn.BackgroundTransparency = 1
    MinimizeBtn.Text = "─"
    MinimizeBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
    MinimizeBtn.TextSize = 20
    MinimizeBtn.Font = Enum.Font.GothamBold
    MinimizeBtn.Parent = TitleBar
    
    -- Botão fechar
    local CloseBtn = Instance.new("TextButton")
    CloseBtn.Size = UDim2.new(0, 30, 1, 0)
    CloseBtn.Position = UDim2.new(1, -30, 0, 0)
    CloseBtn.BackgroundTransparency = 1
    CloseBtn.Text = "✕"
    CloseBtn.TextColor3 = Color3.fromRGB(255, 100, 100)
    CloseBtn.TextSize = 18
    CloseBtn.Font = Enum.Font.GothamBold
    CloseBtn.Parent = TitleBar
    
    -- Criar ScrollingFrame para o conteúdo
    local ContentFrame = Instance.new("ScrollingFrame")
    ContentFrame.Size = UDim2.new(1, -20, 1, -50)
    ContentFrame.Position = UDim2.new(0, 10, 0, 50)
    ContentFrame.BackgroundTransparency = 1
    ContentFrame.BorderSizePixel = 0
    ContentFrame.ScrollBarThickness = 6
    ContentFrame.CanvasSize = UDim2.new(0, 0, 0, 400)
    ContentFrame.Parent = MainFrame
    
    local UIListLayout = Instance.new("UIListLayout")
    UIListLayout.Padding = UDim.new(0, 10)
    UIListLayout.SortOrder = Enum.SortOrder.LayoutOrder
    UIListLayout.Parent = ContentFrame
    
    -- Função para criar toggle
    local function CreateToggle(text, defaultValue, callback)
        local ToggleFrame = Instance.new("Frame")
        ToggleFrame.Size = UDim2.new(1, 0, 0, 40)
        ToggleFrame.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
        ToggleFrame.BackgroundTransparency = 0.3
        ToggleFrame.BorderSizePixel = 0
        ToggleFrame.Parent = ContentFrame
        
        local ToggleCorner = Instance.new("UICorner")
        ToggleCorner.CornerRadius = UDim.new(0, 5)
        ToggleCorner.Parent = ToggleFrame
        
        local Label = Instance.new("TextLabel")
        Label.Size = UDim2.new(1, -60, 1, 0)
        Label.Position = UDim2.new(0, 10, 0, 0)
        Label.BackgroundTransparency = 1
        Label.Text = text
        Label.TextColor3 = Color3.fromRGB(220, 220, 220)
        Label.TextSize = 14
        Label.TextXAlignment = Enum.TextXAlignment.Left
        Label.Font = Enum.Font.Gotham
        Label.Parent = ToggleFrame
        
        local ToggleBtn = Instance.new("TextButton")
        ToggleBtn.Size = UDim2.new(0, 40, 0, 25)
        ToggleBtn.Position = UDim2.new(1, -50, 0.5, -12.5)
        ToggleBtn.BackgroundColor3 = defaultValue and Color3.fromRGB(0, 200, 0) or Color3.fromRGB(200, 0, 0)
        ToggleBtn.Text = defaultValue and "ON" or "OFF"
        ToggleBtn.TextColor3 = Color3.fromRGB(255, 255, 255)
        ToggleBtn.TextSize = 12
        ToggleBtn.Font = Enum.Font.GothamBold
        ToggleBtn.BorderSizePixel = 0
        ToggleBtn.Parent = ToggleFrame
        
        local BtnCorner = Instance.new("UICorner")
        BtnCorner.CornerRadius = UDim.new(0, 4)
        BtnCorner.Parent = ToggleBtn
        
        local isOn = defaultValue
        
        ToggleBtn.MouseButton1Click:Connect(function()
            isOn = not isOn
            ToggleBtn.BackgroundColor3 = isOn and Color3.fromRGB(0, 200, 0) or Color3.fromRGB(200, 0, 0)
            ToggleBtn.Text = isOn and "ON" or "OFF"
            callback(isOn)
        end)
        
        callback(defaultValue)
        
        return ToggleBtn
    end
    
    -- Função para criar slider
    local function CreateSlider(text, min, max, default, suffix, callback)
        local SliderFrame = Instance.new("Frame")
        SliderFrame.Size = UDim2.new(1, 0, 0, 70)
        SliderFrame.BackgroundColor3 = Color3.fromRGB(35, 35, 45)
        SliderFrame.BackgroundTransparency = 0.3
        SliderFrame.BorderSizePixel = 0
        SliderFrame.Parent = ContentFrame
        
        local SliderCorner = Instance.new("UICorner")
        SliderCorner.CornerRadius = UDim.new(0, 5)
        SliderCorner.Parent = SliderFrame
        
        local Label = Instance.new("TextLabel")
        Label.Size = UDim2.new(1, -20, 0, 25)
        Label.Position = UDim2.new(0, 10, 0, 5)
        Label.BackgroundTransparency = 1
        Label.Text = text
        Label.TextColor3 = Color3.fromRGB(220, 220, 220)
        Label.TextSize = 14
        Label.TextXAlignment = Enum.TextXAlignment.Left
        Label.Font = Enum.Font.Gotham
        Label.Parent = SliderFrame
        
        local ValueLabel = Instance.new("TextLabel")
        ValueLabel.Size = UDim2.new(0, 50, 0, 25)
        ValueLabel.Position = UDim2.new(1, -60, 0, 5)
        ValueLabel.BackgroundTransparency = 1
        ValueLabel.Text = tostring(default) .. suffix
        ValueLabel.TextColor3 = Color3.fromRGB(100, 200, 255)
        ValueLabel.TextSize = 14
        ValueLabel.TextXAlignment = Enum.TextXAlignment.Right
        ValueLabel.Font = Enum.Font.Gotham
        ValueLabel.Parent = SliderFrame
        
        local SliderBar = Instance.new("Frame")
        SliderBar.Size = UDim2.new(1, -20, 0, 4)
        SliderBar.Position = UDim2.new(0, 10, 0, 40)
        SliderBar.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
        SliderBar.BorderSizePixel = 0
        SliderBar.Parent = SliderFrame
        
        local SliderBarCorner = Instance.new("UICorner")
        SliderBarCorner.CornerRadius = UDim.new(1, 0)
        SliderBarCorner.Parent = SliderBar
        
        local FillBar = Instance.new("Frame")
        FillBar.Size = UDim2.new((default - min) / (max - min), 0, 1, 0)
        FillBar.BackgroundColor3 = Color3.fromRGB(100, 150, 255)
        FillBar.BorderSizePixel = 0
        FillBar.Parent = SliderBar
        
        local FillBarCorner = Instance.new("UICorner")
        FillBarCorner.CornerRadius = UDim.new(1, 0)
        FillBarCorner.Parent = FillBar
        
        local currentValue = default
        
        local function UpdateValue(value)
            currentValue = math.clamp(value, min, max)
            local percent = (currentValue - min) / (max - min)
            FillBar.Size = UDim2.new(percent, 0, 1, 0)
            ValueLabel.Text = string.format("%.0f", currentValue) .. suffix
            callback(currentValue)
        end
        
        local dragging = false
        
        SliderBar.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 then
                dragging = true
                local percent = math.clamp((input.Position.X - SliderBar.AbsolutePosition.X) / SliderBar.AbsoluteSize.X, 0, 1)
                local newValue = min + (max - min) * percent
                UpdateValue(newValue)
            end
        end)
        
        UserInputService.InputEnded:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 then
                dragging = false
            end
        end)
        
        UserInputService.InputChanged:Connect(function(input)
            if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
                local percent = math.clamp((input.Position.X - SliderBar.AbsolutePosition.X) / SliderBar.AbsoluteSize.X, 0, 1)
                local newValue = min + (max - min) * percent
                UpdateValue(newValue)
            end
        end)
        
        UpdateValue(default)
        
        return SliderBar
    end
    
    -- Criar os elementos da UI
    CreateToggle("🔫 Aimbot", false, function(value)
        AimbotEnabled = value
        print("[Dark Aura] Aimbot:", value and "Ativado" or "Desativado")
        if not value then
            CurrentTarget = nil
        end
    end)
    
    CreateSlider("🎯 FOV Radius", 100, 800, 200, "px", function(value)
        FOVRadius = value
        print("[Dark Aura] FOV:", value)
    end)
    
    CreateToggle("👁️ ESP", false, function(value)
        ESPEnabled = value
        print("[Dark Aura] ESP:", value and "Ativado" or "Desativado")
        if value then
            SetupESP()
        else
            ClearESP()
        end
    end)
    
    CreateToggle("📏 ESP Lines", false, function(value)
        ESPLinesEnabled = value
        print("[Dark Aura] ESP Lines:", value and "Ativado" or "Desativado")
    end)
    
    CreateSlider("✨ Smoothness", 1, 100, 70, "%", function(value)
        Smoothness = 1 - (value / 100)
        print("[Dark Aura] Smoothness:", value, "%")
    end)
    
    -- Controle de visibilidade do menu
    local menuVisible = false
    ToggleButton.MouseButton1Click:Connect(function()
        menuVisible = not menuVisible
        MainFrame.Visible = menuVisible
    end)
    
    MinimizeBtn.MouseButton1Click:Connect(function()
        MainFrame.Visible = false
        menuVisible = false
    end)
    
    CloseBtn.MouseButton1Click:Connect(function()
        MainFrame.Visible = false
        menuVisible = false
    end)
    
    -- Permitir arrastar a janela
    local dragging = false
    local dragStart = nil
    local frameStart = nil
    
    TitleBar.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = true
            dragStart = input.Position
            frameStart = MainFrame.Position
        end
    end)
    
    UserInputService.InputChanged:Connect(function(input)
        if dragging and input.UserInputType == Enum.UserInputType.MouseMovement then
            local delta = input.Position - dragStart
            MainFrame.Position = UDim2.new(frameStart.X.Scale, frameStart.X.Offset + delta.X, frameStart.Y.Scale, frameStart.Y.Offset + delta.Y)
        end
    end)
    
    UserInputService.InputEnded:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 then
            dragging = false
        end
    end)
    
    print("[Dark Aura] Interface carregada com sucesso!")
end

-- ============================================
-- SEÇÃO: LÓGICA DO AIMBOT (CORRIGIDA)
-- ============================================

local function GetClosestPlayerToCursor()
    local closestDistance = FOVRadius
    local closestPlayer = nil
    local mousePosition = UserInputService:GetMouseLocation()
    
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and player.Character and player.Character:FindFirstChild("Humanoid") and player.Character.Humanoid.Health > 0 then
            local head = player.Character:FindFirstChild("Head") or player.Character:FindFirstChild("HumanoidRootPart")
            if head then
                local screenPos, onScreen = Camera:WorldToViewportPoint(head.Position)
                if onScreen then
                    local distance = (Vector2.new(screenPos.X, screenPos.Y) - mousePosition).Magnitude
                    if distance < closestDistance then
                        closestDistance = distance
                        closestPlayer = player
                    end
                end
            end
        end
    end
    
    return closestPlayer
end

local function ApplySmoothAimbot(target)
    if not target or not target.Character then return end
    
    local targetPart = target.Character:FindFirstChild("Head") or target.Character:FindFirstChild("HumanoidRootPart")
    if not targetPart then return end
    
    local targetCFrame = CFrame.new(Camera.CFrame.Position, targetPart.Position)
    
    -- Aplicar suavização
    local newCFrame = Camera.CFrame:Lerp(targetCFrame, Smoothness)
    Camera.CFrame = newCFrame
end

-- ============================================
-- SEÇÃO: LÓGICA DO ESP (CORRIGIDA)
-- ============================================

local function SetupESP()
    ClearESP()
    
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local box = Drawing.new("Square")
            box.Thickness = 2
            box.Color = Color3.fromRGB(255, 0, 0)
            box.Filled = false
            box.Visible = false
            
            local nameText = Drawing.new("Text")
            nameText.Size = 14
            nameText.Center = true
            nameText.Outline = true
            nameText.Color = Color3.fromRGB(255, 255, 255)
            nameText.Visible = false
            
            local healthBar = Drawing.new("Line")
            healthBar.Thickness = 3
            healthBar.Color = Color3.fromRGB(0, 255, 0)
            healthBar.Visible = false
            
            local espLine = Drawing.new("Line")
            espLine.Thickness = 1
            espLine.Color = Color3.fromRGB(255, 0, 255)
            espLine.Visible = false
            
            ESPObjects[player.UserId] = {
                box = box,
                name = nameText,
                health = healthBar,
                line = espLine
            }
        end
    end
end

local function ClearESP()
    for _, objects in pairs(ESPObjects) do
        if objects.box then objects.box:Remove() end
        if objects.name then objects.name:Remove() end
        if objects.health then objects.health:Remove() end
        if objects.line then objects.line:Remove() end
    end
    ESPObjects = {}
end

local function UpdateESP()
    if not ESPEnabled then return end
    
    local screenCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y)
    
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer and ESPObjects[player.UserId] then
            local objects = ESPObjects[player.UserId]
            
            if player.Character and player.Character:FindFirstChild("HumanoidRootPart") and player.Character:FindFirstChild("Humanoid") then
                local hrp = player.Character.HumanoidRootPart
                local humanoid = player.Character.Humanoid
                local pos, onScreen = Camera:WorldToViewportPoint(hrp.Position)
                
                if onScreen and humanoid.Health > 0 then
                    local distance = (Camera.CFrame.Position - hrp.Position).Magnitude
                    local boxSize = math.clamp(200 / distance, 30, 120)
                    
                    -- Atualizar box
                    objects.box.Size = Vector2.new(boxSize, boxSize)
                    objects.box.Position = Vector2.new(pos.X - boxSize/2, pos.Y - boxSize/2)
                    objects.box.Visible = true
                    
                    -- Atualizar nome
                    objects.name.Text = player.Name
                    objects.name.Position = Vector2.new(pos.X, pos.Y - boxSize/2 - 10)
                    objects.name.Visible = true
                    
                    -- Atualizar barra de vida
                    local healthPercent = humanoid.Health / humanoid.MaxHealth
                    objects.health.From = Vector2.new(pos.X - boxSize/2, pos.Y - boxSize/2 - 5)
                    objects.health.To = Vector2.new(pos.X - boxSize/2 + (boxSize * healthPercent), pos.Y - boxSize/2 - 5)
                    objects.health.Visible = true
                    
                    -- Atualizar linha ESP
                    if ESPLinesEnabled then
                        objects.line.From = screenCenter
                        objects.line.To = Vector2.new(pos.X, pos.Y)
                        objects.line.Visible = true
                    else
                        objects.line.Visible = false
                    end
                else
                    objects.box.Visible = false
                    objects.name.Visible = false
                    objects.health.Visible = false
                    objects.line.Visible = false
                end
            else
                objects.box.Visible = false
                objects.name.Visible = false
                objects.health.Visible = false
                objects.line.Visible = false
            end
        end
    end
end

-- ============================================
-- SEÇÃO: EVENTOS E LOOP PRINCIPAL
-- ============================================

-- Detectar novos players
Players.PlayerAdded:Connect(function(player)
    if ESPEnabled and player ~= LocalPlayer then
        local box = Drawing.new("Square")
        box.Thickness = 2
        box.Color = Color3.fromRGB(255, 0, 0)
        box.Filled = false
        box.Visible = false
        
        local nameText = Drawing.new("Text")
        nameText.Size = 14
        nameText.Center = true
        nameText.Outline = true
        nameText.Color = Color3.fromRGB(255, 255, 255)
        nameText.Visible = false
        
        local healthBar = Drawing.new("Line")
        healthBar.Thickness = 3
        healthBar.Color = Color3.fromRGB(0, 255, 0)
        healthBar.Visible = false
        
        local espLine = Drawing.new("Line")
        espLine.Thickness = 1
        espLine.Color = Color3.fromRGB(255, 0, 255)
        espLine.Visible = false
        
        ESPObjects[player.UserId] = {
            box = box,
            name = nameText,
            health = healthBar,
            line = espLine
        }
    end
end)

Players.PlayerRemoving:Connect(function(player)
    if ESPObjects[player.UserId] then
        if ESPObjects[player.UserId].box then ESPObjects[player.UserId].box:Remove() end
        if ESPObjects[player.UserId].name then ESPObjects[player.UserId].name:Remove() end
        if ESPObjects[player.UserId].health then ESPObjects[player.UserId].health:Remove() end
        if ESPObjects[player.UserId].line then ESPObjects[player.UserId].line:Remove() end
        ESPObjects[player.UserId] = nil
    end
end)

-- Loop principal
RunService.RenderStepped:Connect(function()
    -- Aimbot
    if AimbotEnabled then
        local target = GetClosestPlayerToCursor()
        if target then
            ApplySmoothAimbot(target)
        end
    end
    
    -- ESP
    UpdateESP()
end)

-- ============================================
-- SEÇÃO: INICIALIZAÇÃO
-- ============================================

-- Iniciar a UI
CreateCustomUI()

-- Mensagem de boas-vindas
game:GetService("StarterGui"):SetCore("SendNotification", {
    Title = "Dark Aura BR 🇧🇷",
    Text = "Sistema carregado! Pressione o botão flutuante para abrir o menu.",
    Duration = 5
})

print("[Dark Aura BR 🇧🇷] Script carregado com sucesso!")
print("[Dark Aura BR 🇧🇷] Use o botão flutuante para abrir o menu")
