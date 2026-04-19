--[[
    Script: Advanced Aimbot & ESP System
    Autor: Desenvolvedor
    Descrição: Sistema completo de aimbot com suavização e ESP para uso próprio.
    Biblioteca: Rayfield UI
]]

-- Carregar a biblioteca Rayfield
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

-- ============================================
-- SEÇÃO: CONFIGURAÇÕES INICIAIS
-- ============================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Camera = workspace.CurrentCamera
local LocalPlayer = Players.LocalPlayer

-- Estado dos recursos
local state = {
    aimbotEnabled = false,
    espEnabled = false,
    espLinesEnabled = false,
    fovRadius = 120,  -- Raio do FOV em pixels
    currentTarget = nil,
    espObjects = {},   -- Armazena objetos Drawing para ESP
}

-- Configurações de suavização do aimbot
local smoothSettings = {
    enabled = true,
    factor = 0.15,    -- Fator de suavização (0 = muito suave, 1 = instantâneo)
}

-- ============================================
-- SEÇÃO: INTERFACE RAYFIELD
-- ============================================

-- Criar janela principal
local Window = Rayfield:CreateWindow({
    Name = "Aimbot System v1.0",
    Icon = 0,
    LoadingTitle = "Carregando Aimbot System",
    LoadingSubtitle = "by Desenvolvedor",
    Theme = "Default",
    ToggleUIKeybind = "K",
    ConfigurationSaving = {
        Enabled = true,
        FolderName = "AimbotSystem",
        FileName = "Config"
    },
    KeySystem = false,  -- Desativado para uso próprio
})

-- Criar aba principal
local AimbotTab = Window:CreateTab("Aimbot Settings", 4483362458)

-- Criar seção de aimbot
local AimbotSection = AimbotTab:CreateSection("Aimbot Configuration")

-- Toggle do Aimbot
AimbotTab:CreateToggle({
    Name = "Aimbot",
    CurrentValue = false,
    Flag = "AimbotToggle",
    Callback = function(Value)
        state.aimbotEnabled = Value
        if not Value then
            state.currentTarget = nil
        end
        print("[Aimbot] Estado:", Value and "Ativado" or "Desativado")
    end
})

-- Slider do FOV
AimbotTab:CreateSlider({
    Name = "FOV Radius",
    Min = 120,
    Max = 1000,
    Increment = 10,
    Suffix = "px",
    CurrentValue = 120,
    Flag = "FOVSlider",
    Callback = function(Value)
        state.fovRadius = Value
        print("[FOV] Raio ajustado para:", Value)
    end
})

-- Toggle do ESP
AimbotTab:CreateToggle({
    Name = "ESP (Ver através de paredes)",
    CurrentValue = false,
    Flag = "ESPToggle",
    Callback = function(Value)
        state.espEnabled = Value
        if Value then
            setupESP()
        else
            clearESP()
        end
        print("[ESP] Estado:", Value and "Ativado" or "Desativado")
    end
})

-- Toggle das linhas do ESP
AimbotTab:CreateToggle({
    Name = "ESP Lines (Linhas para os players)",
    CurrentValue = false,
    Flag = "ESPLinesToggle",
    Callback = function(Value)
        state.espLinesEnabled = Value
        print("[ESP Lines] Estado:", Value and "Ativado" or "Desativado")
    end
})

-- Criar seção de configurações avançadas
local AdvancedSection = AimbotTab:CreateSection("Advanced Settings")

-- Slider de suavização do aimbot
AimbotTab:CreateSlider({
    Name = "Aimbot Smoothness",
    Min = 1,
    Max = 100,
    Increment = 1,
    Suffix = "%",
    CurrentValue = 85,
    Flag = "SmoothSlider",
    Callback = function(Value)
        -- Inverter o valor: quanto maior o slider, mais suave
        smoothSettings.factor = math.clamp(Value / 100, 0.05, 0.95)
        smoothSettings.factor = 1 - smoothSettings.factor  -- Inverte para comportamento intuitivo
        print("[Smooth] Fator de suavização:", smoothSettings.factor)
    end
})

-- Notificação de inicialização
Rayfield:Notify({
    Title = "Sistema Carregado",
    Content = "Aimbot e ESP prontos para uso!",
    Duration = 3,
})

-- ============================================
-- SEÇÃO: LÓGICA DO AIMBOT
-- ============================================

-- Função para obter a posição da cabeça do alvo
local function getHeadPosition(targetCharacter)
    if not targetCharacter then return nil end
    
    -- Tenta encontrar a cabeça
    local head = targetCharacter:FindFirstChild("Head")
    if head then
        return head.Position
    end
    
    -- Fallback: usa o HumanoidRootPart
    local hrp = targetCharacter:FindFirstChild("HumanoidRootPart")
    if hrp then
        return hrp.Position + Vector3.new(0, 1.5, 0)
    end
    
    return nil
end

-- Função para verificar se um player é válido (não é o próprio jogador e está vivo)
local function isValidPlayer(targetPlayer)
    if not targetPlayer then return false end
    if targetPlayer == LocalPlayer then return false end
    
    local character = targetPlayer.Character
    if not character then return false end
    
    local humanoid = character:FindFirstChildWhichIsA("Humanoid")
    if not humanoid or humanoid.Health <= 0 then return false end
    
    return true
end

-- Função para calcular a distância do centro da tela até um ponto 3D
local function getDistanceFromScreenCenter(worldPosition)
    local vector, onScreen = Camera:WorldToViewportPoint(worldPosition)
    if not onScreen then return math.huge end
    
    local screenCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y / 2)
    local screenPos = Vector2.new(vector.X, vector.Y)
    return (screenPos - screenCenter).Magnitude
end

-- Função para encontrar o player mais próximo do centro da tela dentro do FOV
local function findClosestTarget()
    local closestDistance = state.fovRadius
    local closestPlayer = nil
    
    for _, player in ipairs(Players:GetPlayers()) do
        if isValidPlayer(player) then
            local character = player.Character
            local headPos = getHeadPosition(character)
            
            if headPos then
                local distance = getDistanceFromScreenCenter(headPos)
                
                -- Verifica se está dentro do FOV
                if distance <= state.fovRadius and distance < closestDistance then
                    closestDistance = distance
                    closestPlayer = player
                end
            end
        end
    end
    
    return closestPlayer
end

-- Função para aplicar o aimbot com suavização
local function applySmoothAimbot(targetPlayer)
    if not targetPlayer or not targetPlayer.Character then return end
    
    local headPos = getHeadPosition(targetPlayer.Character)
    if not headPos then return end
    
    -- Calcula o CFrame alvo olhando para a cabeça do jogador
    local targetCFrame = CFrame.new(Camera.CFrame.Position, headPos)
    
    -- Aplica suavização usando Lerp
    local newCFrame = Camera.CFrame:Lerp(targetCFrame, smoothSettings.factor)
    Camera.CFrame = newCFrame
end

-- Função para aplicar aimbot instantâneo
local function applyInstantAimbot(targetPlayer)
    if not targetPlayer or not targetPlayer.Character then return end
    
    local headPos = getHeadPosition(targetPlayer.Character)
    if not headPos then return end
    
    -- Aplica instantaneamente
    Camera.CFrame = CFrame.new(Camera.CFrame.Position, headPos)
end

-- ============================================
-- SEÇÃO: LÓGICA DO ESP
-- ============================================

-- Função para criar elementos de desenho (ESP)
local function createDrawingObjects()
    return {
        box = Drawing.new("Square"),
        name = Drawing.new("Text"),
        healthBar = Drawing.new("Line"),
        lineToPlayer = Drawing.new("Line"),  -- Para as linhas do ESP
    }
end

-- Função para configurar as propriedades dos elementos de desenho
local function setupDrawingProperties(drawings, color)
    -- Configurar box (quadrado)
    drawings.box.Thickness = 2
    drawings.box.Color = color
    drawings.box.Filled = false
    drawings.box.Visible = true
    
    -- Configurar texto do nome
    drawings.name.Size = 14
    drawings.name.Center = true
    drawings.name.Outline = true
    drawings.name.Color = Color3.fromRGB(255, 255, 255)
    drawings.name.Visible = true
    
    -- Configurar barra de vida
    drawings.healthBar.Thickness = 3
    drawings.healthBar.Color = Color3.fromRGB(0, 255, 0)
    drawings.healthBar.Visible = true
    
    -- Configurar linha (ESP Lines)
    drawings.lineToPlayer.Thickness = 1
    drawings.lineToPlayer.Color = color
    drawings.lineToPlayer.Visible = state.espLinesEnabled
end

-- Função para atualizar o ESP de um jogador
local function updateESPForPlayer(player, drawings)
    if not player.Character then return end
    
    local hrp = player.Character:FindFirstChild("HumanoidRootPart")
    local humanoid = player.Character:FindFirstChildWhichIsA("Humanoid")
    
    if not hrp or not humanoid then
        -- Esconder elementos se o personagem não estiver válido
        drawings.box.Visible = false
        drawings.name.Visible = false
        drawings.healthBar.Visible = false
        drawings.lineToPlayer.Visible = false
        return
    end
    
    -- Converter posição 3D para 2D
    local pos, onScreen = Camera:WorldToViewportPoint(hrp.Position)
    
    if onScreen and state.espEnabled then
        -- Calcular tamanho da box baseado na distância
        local distance = (Camera.CFrame.Position - hrp.Position).Magnitude
        local boxSize = math.clamp(300 / distance, 30, 150)
        
        -- Atualizar box (quadrado ao redor do player)
        drawings.box.Size = Vector2.new(boxSize, boxSize)
        drawings.box.Position = Vector2.new(pos.X - boxSize/2, pos.Y - boxSize)
        drawings.box.Visible = true
        
        -- Atualizar nome do player
        drawings.name.Text = player.Name
        drawings.name.Position = Vector2.new(pos.X, pos.Y - boxSize - 10)
        drawings.name.Visible = true
        
        -- Atualizar barra de vida
        local healthPercent = humanoid.Health / humanoid.MaxHealth
        local barWidth = boxSize
        local barHeight = 4
        
        drawings.healthBar.From = Vector2.new(pos.X - barWidth/2, pos.Y - boxSize - 2)
        drawings.healthBar.To = Vector2.new(pos.X - barWidth/2 + (barWidth * healthPercent), pos.Y - boxSize - 2)
        drawings.healthBar.Visible = true
        
        -- Atualizar linha do centro da tela ao player (ESP Lines)
        if state.espLinesEnabled then
            local screenCenter = Vector2.new(Camera.ViewportSize.X / 2, Camera.ViewportSize.Y)
            drawings.lineToPlayer.From = screenCenter
            drawings.lineToPlayer.To = Vector2.new(pos.X, pos.Y)
            drawings.lineToPlayer.Visible = true
        else
            drawings.lineToPlayer.Visible = false
        end
    else
        -- Esconder elementos se o player estiver fora da tela
        drawings.box.Visible = false
        drawings.name.Visible = false
        drawings.healthBar.Visible = false
        drawings.lineToPlayer.Visible = false
    end
end

-- Função para configurar o ESP para todos os players
local function setupESP()
    -- Limpar ESP existente
    clearESP()
    
    -- Criar objetos de desenho para cada player
    for _, player in ipairs(Players:GetPlayers()) do
        if player ~= LocalPlayer then
            local drawings = createDrawingObjects()
            setupDrawingProperties(drawings, Color3.fromRGB(255, 0, 0)) -- Vermelho para inimigos
            state.espObjects[player.UserId] = drawings
        end
    end
end

-- Função para limpar todos os elementos do ESP
local function clearESP()
    for userId, drawings in pairs(state.espObjects) do
        -- Remover todos os objetos de desenho
        if drawings.box then drawings.box:Remove() end
        if drawings.name then drawings.name:Remove() end
        if drawings.healthBar then drawings.healthBar:Remove() end
        if drawings.lineToPlayer then drawings.lineToPlayer:Remove() end
    end
    state.espObjects = {}
end

-- Função para lidar com novos players que entram no jogo
local function onPlayerAdded(player)
    if player ~= LocalPlayer and state.espEnabled then
        local drawings = createDrawingObjects()
        setupDrawingProperties(drawings, Color3.fromRGB(255, 0, 0))
        state.espObjects[player.UserId] = drawings
    end
end

-- Função para lidar com players que saem do jogo
local function onPlayerRemoving(player)
    local drawings = state.espObjects[player.UserId]
    if drawings then
        if drawings.box then drawings.box:Remove() end
        if drawings.name then drawings.name:Remove() end
        if drawings.healthBar then drawings.healthBar:Remove() end
        if drawings.lineToPlayer then drawings.lineToPlayer:Remove() end
        state.espObjects[player.UserId] = nil
    end
end

-- ============================================
-- SEÇÃO: LOOP PRINCIPAL (RENDER STEP)
-- ============================================

-- Loop principal executado a cada frame
local function onRenderStep()
    -- Aimbot logic
    if state.aimbotEnabled then
        local target = findClosestTarget()
        state.currentTarget = target
        
        if target then
            -- Aplica aimbot com suavização
            applySmoothAimbot(target)
        end
    end
    
    -- ESP update logic
    if state.espEnabled then
        for _, player in ipairs(Players:GetPlayers()) do
            if player ~= LocalPlayer then
                local drawings = state.espObjects[player.UserId]
                if drawings then
                    updateESPForPlayer(player, drawings)
                end
            end
        end
    end
end

-- ============================================
-- SEÇÃO: INICIALIZAÇÃO E EVENTOS
-- ============================================

-- Conectar eventos de players
Players.PlayerAdded:Connect(onPlayerAdded)
Players.PlayerRemoving:Connect(onPlayerRemoving)

-- Conectar o loop principal ao RenderStepped
RunService.RenderStepped:Connect(onRenderStep)

-- Inicializar ESP se já estiver ativo (caso o jogador ative via configuração salva)
task.wait(1)  -- Aguardar um pouco para garantir que o personagem carregou
if state.espEnabled then
    setupESP()
end

print("[Sistema] Aimbot System inicializado com sucesso!")
print("[Sistema] Pressione 'K' para abrir/fechar o menu")

-- ============================================
-- SEÇÃO: FUNÇÕES UTILITÁRIAS
-- ============================================

-- Função de clamp para valores (adicionada ao globals)
if not math.clamp then
    function math.clamp(value, min, max)
        return math.max(min, math.min(max, value))
    end
end
