--[[
    SCRIPT COMPLETO: AIMBOT + ESP + INTERFACE PROFISSIONAL
    Autor: Assistente Roblox
    Versão: 1.0
    
    REQUISITOS ATENDIDOS:
    ✓ Sistema de Key/Password (tela de login com key 5609)
    ✓ Interface principal moderna com tema escuro/glassmorphism
    ✓ Aimbot com círculo central (toggle ON/OFF)
    ✓ Mira automática na cabeça do jogador dentro do círculo
    ✓ FOV configurável (Slider)
    ✓ Smoothness configurável (Slider)
    ✓ ESP através de paredes (Toggle)
    ✓ ESP Lines da câmera até os players (Toggle)
    ✓ Keybind para abrir/fechar interface (Insert)
    ✓ Interface arrastável, minimizável e com botão fechar
    ✓ Código organizado, comentado e com otimizações
--]]

-- ============================================
-- SEÇÃO 1: SERVIÇOS E VARIÁVEIS GLOBAIS
-- ============================================

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local TweenService = game:GetService("TweenService")
local Lighting = game:GetService("Lighting")

local player = Players.LocalPlayer
local camera = workspace.CurrentCamera

-- Configurações do Aimbot
local aimbotActive = false
local aimbotFOV = 150  -- Raio do círculo em pixels
local aimbotSmoothness = 0.3  -- Suavidade da mira (0 = instantâneo, 1 = muito suave)
local currentTarget = nil

-- Configurações do ESP
local espActive = false
local espLinesActive = false

-- Estado da UI
local mainUIVisible = true
local loginUIVisible = true

-- Objetos da UI
local loginScreen = nil
local mainUI = nil
local fovCircle = nil
local espObjects = {}  -- Para armazenar objetos ESP
local espLines = {}    -- Para armazenar linhas ESP

-- ============================================
-- SEÇÃO 2: FUNÇÕES AUXILIARES
-- ============================================

-- Função para criar som de clique (opcional)
local function playClickSound()
    local sound = Instance.new("Sound")
    sound.SoundId = "rbxasset://sounds/ui_button.mp3"
    sound.Volume = 0.3
    sound.Parent = game:GetService("Players").LocalPlayer.Character or workspace
    sound:Play()
    game:Debris:AddItem(sound, 1)
end

-- Função para animação de fade in
local function fadeIn(guiObject, duration)
    guiObject.BackgroundTransparency = 1
    local tween = TweenService:Create(guiObject, TweenInfo.new(duration), {BackgroundTransparency = 0})
    tween:Play()
end

-- Função para animação de fade out
local function fadeOut(guiObject, duration)
    local tween = TweenService:Create(guiObject, TweenInfo.new(duration), {BackgroundTransparency = 1})
    tween:Play()
    return tween
end

-- Função para obter a posição da cabeça de um jogador
local function getHeadPosition(targetPlayer)
    if not targetPlayer or not targetPlayer.Character then
        return nil
    end
    
    local character = targetPlayer.Character
    local humanoidRootPart = character:FindFirstChild("HumanoidRootPart")
    local head = character:FindFirstChild("Head")
    
    if not humanoidRootPart or not head then
        return nil
    end
    
    -- Verifica se o jogador está vivo
    local humanoid = character:FindFirstChild("Humanoid")
    if not humanoid or humanoid.Health <= 0 then
        return nil
    end
    
    return head.Position
end

-- Função para verificar se um ponto está dentro do círculo FOV
local function isInFOV(screenPosition, centerScreen)
    local distance = (screenPosition - centerScreen).Magnitude
    return distance <= aimbotFOV
end

-- Função para converter posição 3D para 2D na tela
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

-- Função para encontrar o melhor alvo dentro do círculo
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

-- Função para aplicar mira suave
local function smoothAim(targetCFrame, currentCFrame, smoothness)
    if smoothness <= 0 then
        return targetCFrame
    end
    
    -- Quanto menor o smoothness, mais rápido
    local alpha = math.max(0, math.min(1, 1 - smoothness))
    return currentCFrame:Lerp(targetCFrame, alpha)
end

-- ============================================
-- SEÇÃO 3: SISTEMA DE AIMBOT
-- ============================================

-- Função principal de atualização da mira
local function updateAimbot()
    if not aimbotActive then
        currentTarget = nil
        return
    end
    
    -- Encontra o melhor alvo
    local bestTarget = findBestTarget()
    
    if bestTarget then
        currentTarget = bestTarget
        local headPos = getHeadPosition(currentTarget)
        
        if headPos then
            -- Define a câmera como controlável por script
            camera.CameraType = Enum.CameraType.Scriptable
            
            -- Calcula o CFrame alvo (câmera mirando na cabeça)
            local cameraPos = camera.CFrame.Position
            local targetCFrame = CFrame.lookAt(cameraPos, headPos)
            
            -- Aplica suavidade se configurada
            if aimbotSmoothness > 0 then
                targetCFrame = smoothAim(targetCFrame, camera.CFrame, aimbotSmoothness)
            end
            
            camera.CFrame = targetCFrame
        end
    else
        currentTarget = nil
        -- Restaura a câmera para o modo normal quando não há alvo
        if aimbotActive then
            camera.CameraType = Enum.CameraType.Custom
        end
    end
end

-- ============================================
-- SEÇÃO 4: SISTEMA DE ESP
-- ============================================

-- Cria um quadro ESP ao redor do jogador
local function createESPBox(targetPlayer)
    if not espActive then
        return nil
    end
    
    -- Verifica se já existe ESP para este jogador
    if espObjects[targetPlayer] then
        return espObjects[targetPlayer]
    end
    
    -- Cria um BillboardGui para o ESP
    local espGui = Instance.new("BillboardGui")
    espGui.Name = "ESPBox"
    espGui.AlwaysOnTop = true
    espGui.Size = UDim2.new(0, 4, 0, 5)
    espGui.StudsOffset = Vector3.new(0, 2.5, 0)
    espGui.Adornee = targetPlayer.Character
    espGui.Parent = targetPlayer.Character
    
    -- Frame do ESP (borda colorida)
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(1, 0, 1, 0)
    frame.BackgroundTransparency = 0.8
    frame.BackgroundColor3 = Color3.fromRGB(255, 0, 0)
    frame.BorderSizePixel = 2
    frame.BorderColor3 = Color3.fromRGB(255, 50, 50)
    frame.Parent = espGui
    
    -- Efeito de gradiente/brilho
    local uiGradient = Instance.new("UIGradient")
    uiGradient.Color = ColorSequence.new(Color3.fromRGB(255, 0, 0), Color3.fromRGB(255, 100, 0))
    uiGradient.Rotation = 45
    uiGradient.Parent = frame
    
    -- Texto com nome e distância
    local nameLabel = Instance.new("TextLabel")
    nameLabel.Size = UDim2.new(1, 0, 0, 20)
    nameLabel.Position = UDim2.new(0, 0, 1, 0)
    nameLabel.BackgroundTransparency = 1
    nameLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    nameLabel.TextStrokeTransparency = 0.5
    nameLabel.Font = Enum.Font.Gothom
    nameLabel.TextSize = 12
    nameLabel.Text = targetPlayer.Name
    nameLabel.Parent = espGui
    
    espObjects[targetPlayer] = espGui
    
    return espGui
end

-- Atualiza a distância no texto do ESP
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
            nameLabel.Text = string.format("%s | %.1fm", targetPlayer.Name, distance)
        end
    end
end

-- Remove ESP de um jogador
local function removeESP(targetPlayer)
    if espObjects[targetPlayer] then
        espObjects[targetPlayer]:Destroy()
        espObjects[targetPlayer] = nil
    end
end

-- Cria linha da câmera até o jogador
local function createESPLine(targetPlayer)
    if not espLinesActive then
        return nil
    end
    
    if espLines[targetPlayer] then
        return espLines[targetPlayer]
    end
    
    -- Para linhas 3D, usamos um sistema simples de desenho
    -- Nota: Para linhas mais precisas, seria necessário usar Drawing API ou um sistema mais complexo
    -- Esta é uma implementação simplificada usando um ponto de referência
    local linePart = Instance.new("Part")
    linePart.Name = "ESPLine"
    linePart.Size = Vector3.new(0.1, 0.1, 0.1)
    linePart.Anchored = true
    linePart.CanCollide = false
    linePart.Transparency = 0.5
    linePart.BrickColor = BrickColor.new("Bright red")
    linePart.Material = Enum.Material.Neon
    linePart.Parent = workspace
    
    espLines[targetPlayer] = linePart
    return linePart
end

-- Atualiza a posição da linha ESP
local function updateESPLine(targetPlayer)
    if not espLinesActive or not espLines[targetPlayer] then
        return
    end
    
    local linePart = espLines[targetPlayer]
    local headPos = getHeadPosition(targetPlayer)
    
    if headPos and camera.CFrame then
        -- Posiciona a linha entre a câmera e o jogador
        local midPoint = (camera.CFrame.Position + headPos) / 2
        local distance = (headPos - camera.CFrame.Position).Magnitude
        
        linePart.Size = Vector3.new(0.1, 0.1, distance)
        linePart.CFrame = CFrame.lookAt(midPoint, headPos) * CFrame.new(0, 0, -distance/2)
        linePart.Transparency = 0.3
    else
        linePart.Transparency = 1
    end
end

-- Remove linha ESP
local function removeESPLine(targetPlayer)
    if espLines[targetPlayer] then
        espLines[targetPlayer]:Destroy()
        espLines[targetPlayer] = nil
    end
end

-- Atualiza todos os elementos ESP
local function updateESP()
    if not espActive then
        -- Remove todos os ESPs se desativado
        for playerObj, espGui in pairs(espObjects) do
            espGui:Destroy()
        end
        espObjects = {}
        return
    end
    
    for _, otherPlayer in ipairs(Players:GetPlayers()) do
        if otherPlayer ~= player and otherPlayer.Character then
            -- Cria/Atualiza ESP Box
            createESPBox(otherPlayer)
            updateESPDistance(otherPlayer)
            
            -- Cria/Atualiza ESP Lines
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

-- ============================================
-- SEÇÃO 5: INTERFACE DO CÍRCULO FOV
-- ============================================

-- Cria o círculo central (FOV)
local function createFOVCircle()
    -- Cria ScreenGui para o círculo
    local fovGui = Instance.new("ScreenGui")
    fovGui.Name = "FOVCircleGUI"
    fovGui.ResetOnSpawn = false
    fovGui.IgnoreGuiInset = true
    fovGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
    fovGui.Parent = player:WaitForChild("PlayerGui")
    
    -- Frame do círculo
    local circleFrame = Instance.new("Frame")
    circleFrame.Name = "FOVCircle"
    circleFrame.AnchorPoint = Vector2.new(0.5, 0.5)
    circleFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
    circleFrame.Size = UDim2.new(0, aimbotFOV * 2, 0, aimbotFOV * 2)
    circleFrame.BackgroundTransparency = 1
    circleFrame.BorderSizePixel = 0
    circleFrame.Parent = fovGui
    
    -- Borda do círculo (UIStroke)
    local stroke = Instance.new("UIStroke")
    stroke.Thickness = 3
    stroke.Color = Color3.fromRGB(0, 255, 255)  -- Ciano
    stroke.Transparency = 0.3
    stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    stroke.Parent = circleFrame
    
    -- Cantos arredondados (para fazer círculo)
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(1, 0)
    corner.Parent = circleFrame
    
    -- Efeito de brilho (gradiente)
    local gradient = Instance.new("UIGradient")
    gradient.Color = ColorSequence.new(Color3.fromRGB(0, 255, 255), Color3.fromRGB(0, 100, 255))
    gradient.Rotation = 45
    gradient.Parent = stroke
    
    -- Animação de pulsação
    local tweenInfo = TweenInfo.new(1, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true)
    local pulseTween = TweenService:Create(stroke, tweenInfo, {Transparency = 0.1})
    pulseTween:Play()
    
    return fovGui, circleFrame
end

-- Atualiza o tamanho do círculo quando o FOV muda
local function updateFOVCircleSize()
    if fovCircle then
        local circleFrame = fovCircle:FindFirstChild("FOVCircle")
        if circleFrame then
            circleFrame.Size = UDim2.new(0, aimbotFOV * 2, 0, aimbotFOV * 2)
        end
    end
end

-- Remove o círculo FOV
local function removeFOVCircle()
    if fovCircle then
        fovCircle:Destroy()
        fovCircle = nil
    end
end

-- ============================================
-- SEÇÃO 6: INTERFACE PRINCIPAL (APÓS LOGIN)
-- ============================================

-- Variáveis para arrastar a UI
local dragging = false
local dragStart = nil
local startPos = nil

-- Cria a interface principal com design glassmorphism
local function createMainUI()
    -- ScreenGui principal
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "MainUI"
    screenGui.ResetOnSpawn = false
    screenGui.ZIndexBehavior = Enum.ZIndexBehavior.Global
    screenGui.Parent = player:WaitForChild("PlayerGui")
    
    -- Frame principal (container)
    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 400, 0, 500)
    mainFrame.Position = UDim2.new(0.5, -200, 0.5, -250)
    mainFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 30)
    mainFrame.BackgroundTransparency = 0.15
    mainFrame.BorderSizePixel = 0
    mainFrame.ClipsDescendants = true
    mainFrame.Parent = screenGui
    
    -- Efeito de blur (glassmorphism)
    local blur = Instance.new("BlurEffect")
    blur.Size = 0
    blur.Parent = mainFrame
    
    -- Cantos arredondados
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 12)
    corner.Parent = mainFrame
    
    -- Borda com gradiente
    local stroke = Instance.new("UIStroke")
    stroke.Thickness = 1
    stroke.Color = Color3.fromRGB(255, 255, 255)
    stroke.Transparency = 0.5
    stroke.Parent = mainFrame
    
    -- Sombra
    local shadow = Instance.new("UIShadow")
    shadow.Size = 15
    shadow.Color = Color3.fromRGB(0, 0, 0)
    shadow.Transparency = 0.5
    shadow.Parent = mainFrame
    
    -- Barra de título (para arrastar)
    local titleBar = Instance.new("Frame")
    titleBar.Name = "TitleBar"
    titleBar.Size = UDim2.new(1, 0, 0, 40)
    titleBar.BackgroundColor3 = Color3.fromRGB(30, 30, 40)
    titleBar.BackgroundTransparency = 0.3
    titleBar.BorderSizePixel = 0
    titleBar.Parent = mainFrame
    
    local titleCorner = Instance.new("UICorner")
    titleCorner.CornerRadius = UDim.new(0, 12)
    titleCorner.Parent = titleBar
    
    -- Título
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1, -80, 1, 0)
    title.Position = UDim2.new(0, 15, 0, 0)
    title.BackgroundTransparency = 1
    title.Text = "⚡ CYBER AIMBOT v1.0"
    title.TextColor3 = Color3.fromRGB(0, 255, 255)
    title.TextSize = 16
    title.Font = Enum.Font.GothamBold
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = titleBar
    
    -- Botão minimizar
    local minButton = Instance.new("TextButton")
    minButton.Size = UDim2.new(0, 30, 1, 0)
    minButton.Position = UDim2.new(1, -60, 0, 0)
    minButton.BackgroundTransparency = 1
    minButton.Text = "─"
    minButton.TextColor3 = Color3.fromRGB(255, 255, 255)
    minButton.TextSize = 20
    minButton.Font = Enum.Font.GothamBold
    minButton.Parent = titleBar
    
    -- Botão fechar
    local closeButton = Instance.new("TextButton")
    closeButton.Size = UDim2.new(0, 30, 1, 0)
    closeButton.Position = UDim2.new(1, -30, 0, 0)
    closeButton.BackgroundTransparency = 1
    closeButton.Text = "✕"
    closeButton.TextColor3 = Color3.fromRGB(255, 100, 100)
    closeButton.TextSize = 18
    closeButton.Font = Enum.Font.GothamBold
    closeButton.Parent = titleBar
    
    -- Container de conteúdo (ScrollFrame)
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
    layout.Padding = UDim.new(0, 10)
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Parent = scrollFrame
    
    -- ===== SEÇÃO: AIMBOT CONTROLS =====
    local aimbotSection = Instance.new("Frame")
    aimbotSection.Size = UDim2.new(1, 0, 0, 120)
    aimbotSection.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
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
    
    -- Toggle Aimbot
    local aimbotToggle = Instance.new("TextButton")
    aimbotToggle.Size = UDim2.new(0, 120, 0, 35)
    aimbotToggle.Position = UDim2.new(0, 10, 0, 40)
    aimbotToggle.BackgroundColor3 = Color3.fromRGB(60, 60, 70)
    aimbotToggle.Text = "🔴 OFF"
    aimbotToggle.TextColor3 = Color3.fromRGB(255, 255, 255)
    aimbotToggle.TextSize = 14
    aimbotToggle.Font = Enum.Font.GothamSemibold
    aimbotToggle.Parent = aimbotSection
    
    local toggleCorner = Instance.new("UICorner")
    toggleCorner.CornerRadius = UDim.new(0, 6)
    toggleCorner.Parent = aimbotToggle
    
    -- ===== SEÇÃO: CONFIGURAÇÕES =====
    local settingsSection = Instance.new("Frame")
    settingsSection.Size = UDim2.new(1, 0, 0, 150)
    settingsSection.BackgroundColor3 = Color3.fromRGB(40, 40, 50)
    settingsSection.BackgroundTransparency = 0.5
    settingsSection.BorderSizePixel = 0
    settingsSection.Parent =
