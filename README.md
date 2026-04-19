--[[
    Script: Aimbot & ESP System - VERSÃO COMPLETAMENTE FUNCIONAL
    Como usar: Execute em um executor compatível (Synapse X, Krnl, Scriptware, etc.)
]]

-- Carregar Rayfield
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

-- ============================================
-- CONFIGURAÇÕES INICIAIS
-- ============================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer
local UserInputService = game:GetService("UserInputService")

-- Variáveis do sistema
local AimbotEnabled = false
local ESPEnabled = false
local ESPLinesEnabled = false
local FOVRadius = 120
local Smoothness = 0.3

-- Objetos de desenho
local FOVCircle = nil
local ESPObjects = {}
local CurrentTarget = nil

-- ============================================
-- FUNÇÕES DE UTILIDADE
-- ============================================

-- Função para obter a posição da cabeça
local function GetHeadPosition(Character)
    if not Character then return nil end
    
    local Head = Character:FindFirstChild("Head")
    if Head then
        return Head.Position
    end
    
    local HumanoidRootPart = Character:FindFirstChild("HumanoidRootPart")
    if HumanoidRootPart then
        return HumanoidRootPart.Position + Vector3.new(0, 1.5, 0)
    end
    
    return nil
end

-- Verificar se player é válido
local function IsValidPlayer(Player)
    if not Player then return false end
    if Player == LocalPlayer then return false end
    if not Player.Character then return false end
    
    local Humanoid = Player.Character:FindFirstChildWhichIsA("Humanoid")
    if not Humanoid or Humanoid.Health <= 0 then return false end
    
    return true
end

-- Calcular distância do centro da tela
local function GetDistanceFromCenter(WorldPosition)
    local ScreenPos, OnScreen = Camera:WorldToViewportPoint(WorldPosition)
    if not OnScreen then return math.huge end
    
    local Center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    local Pos = Vector2.new(ScreenPos.X, ScreenPos.Y)
    
    return (Pos - Center).Magnitude
end

-- Encontrar player mais próximo do centro da tela
local function GetClosestPlayer()
    local ClosestDistance = FOVRadius
    local ClosestPlayer = nil
    
    for _, Player in ipairs(Players:GetPlayers()) do
        if IsValidPlayer(Player) then
            local HeadPos = GetHeadPosition(Player.Character)
            if HeadPos then
                local Distance = GetDistanceFromCenter(HeadPos)
                
                if Distance < ClosestDistance then
                    ClosestDistance = Distance
                    ClosestPlayer = Player
                end
            end
        end
    end
    
    return ClosestPlayer
end

-- ============================================
-- FUNÇÕES DO FOV (CÍRCULO VISUAL)
-- ============================================

local function CreateFOVCircle()
    if FOVCircle then
        FOVCircle:Remove()
    end
    
    FOVCircle = Drawing.new("Circle")
    FOVCircle.Radius = FOVRadius
    FOVCircle.Thickness = 2
    FOVCircle.Color = Color3.fromRGB(255, 255, 255)
    FOVCircle.Filled = false
    FOVCircle.NumSides = 64
    FOVCircle.Visible = AimbotEnabled
    FOVCircle.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
end

local function UpdateFOVCircle()
    if FOVCircle then
        FOVCircle.Position = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
        FOVCircle.Radius = FOVRadius
        FOVCircle.Visible = AimbotEnabled
    end
end

-- ============================================
-- FUNÇÕES DO AIMBOT
-- ============================================

local function ApplyAimbot(TargetPlayer)
    if not TargetPlayer or not TargetPlayer.Character then return end
    
    local HeadPos = GetHeadPosition(TargetPlayer.Character)
    if not HeadPos then return end
    
    -- CFrame alvo olhando para a cabeça
    local TargetCFrame = CFrame.new(Camera.CFrame.Position, HeadPos)
    
    -- Aplicar suavização
    local NewCFrame = Camera.CFrame:Lerp(TargetCFrame, Smoothness)
    Camera.CFrame = NewCFrame
end

-- ============================================
-- FUNÇÕES DO ESP
-- ============================================

local function CreateESPObjects(Player)
    local Objects = {
        Box = Drawing.new("Square"),
        Name = Drawing.new("Text"),
        HealthBar = Drawing.new("Line"),
        Line = Drawing.new("Line")
    }
    
    -- Configurar Box
    Objects.Box.Thickness = 2
    Objects.Box.Color = Color3.fromRGB(255, 0, 0)
    Objects.Box.Filled = false
    Objects.Box.Visible = false
    
    -- Configurar Nome
    Objects.Name.Size = 14
    Objects.Name.Center = true
    Objects.Name.Outline = true
    Objects.Name.Color = Color3.fromRGB(255, 255, 255)
    Objects.Name.Visible = false
    
    -- Configurar Barra de Vida
    Objects.HealthBar.Thickness = 3
    Objects.HealthBar.Color = Color3.fromRGB(0, 255, 0)
    Objects.HealthBar.Visible = false
    
    -- Configurar Linha
    Objects.Line.Thickness = 1
    Objects.Line.Color = Color3.fromRGB(255, 0, 0)
    Objects.Line.Visible = false
    
    return Objects
end

local function UpdateESP()
    if not ESPEnabled then return end
    
    for _, Player in ipairs(Players:GetPlayers()) do
        if Player ~= LocalPlayer and IsValidPlayer(Player) then
            -- Criar objetos se não existirem
            if not ESPObjects[Player.UserId] then
                ESPObjects[Player.UserId] = CreateESPObjects(Player)
            end
            
            local Objects = ESPObjects[Player.UserId]
            local Character = Player.Character
            local HumanoidRootPart = Character:FindFirstChild("HumanoidRootPart")
            local Humanoid = Character:FindFirstChildWhichIsA("Humanoid")
            
            if HumanoidRootPart and Humanoid then
                local ScreenPos, OnScreen = Camera:WorldToViewportPoint(HumanoidRootPart.Position)
                
                if OnScreen then
                    -- Calcular tamanho da box baseado na distância
                    local Distance = (Camera.CFrame.Position - HumanoidRootPart.Position).Magnitude
                    local BoxSize = math.clamp(200 / Distance, 40, 150)
                    
                    -- Atualizar Box
                    Objects.Box.Size = Vector2.new(BoxSize, BoxSize)
                    Objects.Box.Position = Vector2.new(ScreenPos.X - BoxSize/2, ScreenPos.Y - BoxSize)
                    Objects.Box.Visible = true
                    
                    -- Atualizar Nome
                    Objects.Name.Text = Player.Name
                    Objects.Name.Position = Vector2.new(ScreenPos.X, ScreenPos.Y - BoxSize - 10)
                    Objects.Name.Visible = true
                    
                    -- Atualizar Barra de Vida
                    local HealthPercent = Humanoid.Health / Humanoid.MaxHealth
                    Objects.HealthBar.From = Vector2.new(ScreenPos.X - BoxSize/2, ScreenPos.Y - BoxSize - 2)
                    Objects.HealthBar.To = Vector2.new(ScreenPos.X - BoxSize/2 + (BoxSize * HealthPercent), ScreenPos.Y - BoxSize - 2)
                    Objects.HealthBar.Visible = true
                    
                    -- Atualizar Linha (ESP Lines)
                    if ESPLinesEnabled then
                        local Center = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y)
                        Objects.Line.From = Center
                        Objects.Line.To = Vector2.new(ScreenPos.X, ScreenPos.Y)
                        Objects.Line.Visible = true
                    else
                        Objects.Line.Visible = false
                    end
                else
                    -- Esconder objetos se estiver fora da tela
                    Objects.Box.Visible = false
                    Objects.Name.Visible = false
                    Objects.HealthBar.Visible = false
                    Objects.Line.Visible = false
                end
            end
        end
    end
end

local function ClearESP()
    for _, Objects in pairs(ESPObjects) do
        pcall(function()
            Objects.Box:Remove()
            Objects.Name:Remove()
            Objects.HealthBar:Remove()
            Objects.Line:Remove()
        end)
    end
    ESPObjects = {}
end

-- ============================================
-- CRIAR INTERFACE RAYFIELD
-- ============================================

local Window = Rayfield:CreateWindow({
    Name = "Aimbot & ESP System",
    Icon = 0,
    LoadingTitle = "Carregando...",
    LoadingSubtitle = "Aimbot System",
    Theme = "Default",
    ToggleUIKeybind = "K",
    ConfigurationSaving = {
        Enabled = true,
        FolderName = "AimbotSystem",
        FileName = "Config"
    }
})

local MainTab = Window:CreateTab("Main", 4483362458)

-- Seção Aimbot
local AimbotSection = MainTab:CreateSection("Aimbot")

-- Toggle Aimbot
MainTab:CreateToggle({
    Name = "Ativar Aimbot",
    CurrentValue = false,
    Flag = "AimbotToggle",
    Callback = function(Value)
        AimbotEnabled = Value
        if FOVCircle then
            FOVCircle.Visible = Value
        end
        if not Value then
            CurrentTarget = nil
        end
        print("[Aimbot] " .. (Value and "Ativado" or "Desativado"))
    end
})

-- Slider FOV
MainTab:CreateSlider({
    Name = "Raio do FOV",
    Min = 50,
    Max = 500,
    Increment = 10,
    Suffix = "px",
    CurrentValue = 120,
    Flag = "FOVSlider",
    Callback = function(Value)
        FOVRadius = Value
        if FOVCircle then
            FOVCircle.Radius = Value
        end
        print("[FOV] Raio: " .. Value)
    end
})

-- Slider Suavização
MainTab:CreateSlider({
    Name = "Suavização do Aimbot",
    Min = 0,
    Max = 100,
    Increment = 5,
    Suffix = "%",
    CurrentValue = 70,
    Flag = "SmoothSlider",
    Callback = function(Value)
        Smoothness = Value / 100
        print("[Smooth] Suavização: " .. Smoothness)
    end
})

-- Seção ESP
local ESPection = MainTab:CreateSection("ESP")

-- Toggle ESP
MainTab:CreateToggle({
    Name = "Ativar ESP",
    CurrentValue = false,
    Flag = "ESPToggle",
    Callback = function(Value)
        ESPEnabled = Value
        if not Value then
            ClearESP()
        end
        print("[ESP] " .. (Value and "Ativado" or "Desativado"))
    end
})

-- Toggle ESP Lines
MainTab:CreateToggle({
    Name = "Ativar Linhas do ESP",
    CurrentValue = false,
    Flag = "ESPLinesToggle",
    Callback = function(Value)
        ESPLinesEnabled = Value
        print("[ESP Lines] " .. (Value and "Ativado" or "Desativado"))
    end
})

-- ============================================
-- LOOP PRINCIPAL
-- ============================================

-- Criar círculo do FOV
CreateFOVCircle()

-- Loop de atualização
RunService.RenderStepped:Connect(function()
    -- Atualizar posição do círculo FOV
    UpdateFOVCircle()
    
    -- Aimbot
    if AimbotEnabled then
        local Target = GetClosestPlayer()
        if Target then
            ApplyAimbot(Target)
        end
    end
    
    -- ESP
    if ESPEnabled then
        UpdateESP()
    end
end)

-- Atualizar círculo quando a tela redimensionar
Camera:GetPropertyChangedSignal("ViewportSize"):Connect(function()
    UpdateFOVCircle()
end)

-- Limpar ESP quando jogadores saírem
Players.PlayerRemoving:Connect(function(Player)
    if ESPObjects[Player.UserId] then
        local Objects = ESPObjects[Player.UserId]
        pcall(function()
            Objects.Box:Remove()
            Objects.Name:Remove()
            Objects.HealthBar:Remove()
            Objects.Line:Remove()
        end)
        ESPObjects[Player.UserId] = nil
    end
end)

-- Notificação de início
Rayfield:Notify({
    Title = "Sistema Carregado",
    Content = "Pressione K para abrir o menu",
    Duration = 5
})

print("[Sistema] Aimbot & ESP System carregado com sucesso!")
print("[Sistema] Pressione K para abrir o menu")
