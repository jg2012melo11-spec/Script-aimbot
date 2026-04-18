--[[
    Script: Aimbot + ESP System com Autenticação
    Autor: Desenvolvedor
    Uso: Apenas para jogos próprios / desenvolvimento
    Key de Acesso: 5609
--]]

-- Carregar a biblioteca Rayfield
local Rayfield = loadstring(game:HttpGet('https://sirius.menu/rayfield'))()

-- Serviços necessários
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local LocalPlayer = Players.LocalPlayer
local Camera = workspace.CurrentCamera

-- ============================================
-- 1. SISTEMA DE AUTENTICAÇÃO
-- ============================================

-- Variável para controlar autenticação
local IsAuthenticated = false

-- Variáveis de configuração (serão preenchidas após autenticação)
local Settings = {
    AimbotEnabled = false,
    FOVRadius = 120,
    ESPEnabled = false,
    ESPLinesEnabled = false,
    SmoothAim = true,
    Smoothness = 0.15,
}

-- Variáveis do sistema (inicializadas após autenticação)
local MainWindow = nil
local FOVCircle = nil
local ESPObjects = {}
local CurrentTarget = nil

-- Criar janela de autenticação
local AuthWindow = Rayfield:CreateWindow({
    Name = "Sistema de Autenticação",
    Icon = 0,
    LoadingTitle = "Verificando Acesso...",
    LoadingSubtitle = "Por favor, insira sua key",
    ConfigurationSaving = {
        Enabled = false  -- Não salvar configurações antes da autenticação
    },
    KeySystem = false
})

-- Criar aba de autenticação
local AuthTab = AuthWindow:CreateTab("Autenticação", nil)
local AuthSection = AuthTab:CreateSection("Login")

-- Criar campo de texto para a Key
local KeyInput = AuthTab:CreateInput({
    Name = "Chave de Acesso",
    PlaceholderText = "Digite sua key aqui...",
    RemoveTextAfterFocusLoss = false,
    CurrentValue = "",
    Flag = "KeyInput",
    Callback = function(Value)
        -- Apenas armazena o valor
    end
})

-- Criar botão de confirmação
local ConfirmButton = AuthTab:CreateButton({
    Name = "Confirmar",
    Callback = function()
        local EnteredKey = KeyInput.CurrentValue
        
        -- Verificar se a key está correta
        if EnteredKey == "5609" then
            -- Autenticação bem-sucedida
            IsAuthenticated = true
            
            -- Mostrar mensagem de sucesso
            Rayfield:Notify({
                Title = "✅ Autenticação Bem-Sucedida",
                Content = "Key válida! Carregando sistema...",
                Duration = 2
            })
            
            -- Fechar janela de autenticação após delay
            task.wait(1)
            AuthWindow:Destroy()
            
            -- Carregar a interface principal
            LoadMainGUI()
        else
            -- Key inválida
            Rayfield:Notify({
                Title = "❌ Erro de Autenticação",
                Content = "Key inválida! Tente novamente.",
                Duration = 3,
                Actions = {
                    Ignore = {
                        Name = "Ok",
                        Callback = function()
                            -- Limpar o campo de texto
                            KeyInput:SetValue("")
                        end
                    }
                }
            })
            
            -- Limpar o campo de texto
            KeyInput:SetValue("")
            
            -- Opcional: piscar o campo em vermelho (feedback visual)
            print("[Sistema] Tentativa de acesso com key inválida")
        end
    end
})

-- Adicionar informação sobre a key
AuthTab:CreateParagraph({
    Name = "Info",
    Content = "🔑 Este sistema requer autenticação.\n\nEntre em contato com o desenvolvedor para obter sua key de acesso."
})

-- ============================================
-- 2. FUNÇÕES DO SISTEMA (Aimbot + ESP)
-- ============================================

-- Função para verificar se um jogador é válido
local function IsValidPlayer(Player)
    if not Player then return false end
    if Player == LocalPlayer then return false end
    
    local Character = Player.Character
    if not Character then return false end
    
    local Humanoid = Character:FindFirstChild("Humanoid")
    if not Humanoid or Humanoid.Health <= 0 then return false end
    
    local Head = Character:FindFirstChild("Head")
    if not Head then return false end
    
    return true
end

-- Função para encontrar o jogador mais próximo do centro da tela
local function GetClosestPlayerToCenter()
    local ViewportSize = Camera.ViewportSize
    local Center = Vector2.new(ViewportSize.X / 2, ViewportSize.Y / 2)
    
    local ClosestPlayer = nil
    local ClosestDistance = Settings.FOVRadius
    
    for _, Player in ipairs(Players:GetPlayers()) do
        if IsValidPlayer(Player) then
            local Head = Player.Character.Head
            local ScreenPos, OnScreen = Camera:WorldToViewportPoint(Head.Position)
            
            if OnScreen then
                local ScreenVector = Vector2.new(ScreenPos.X, ScreenPos.Y)
                local Distance = (ScreenVector - Center).Magnitude
                
                if Distance < ClosestDistance then
                    ClosestDistance = Distance
                    ClosestPlayer = Player
                end
            end
        end
    end
    
    return ClosestPlayer
end

-- Função para suavizar a rotação da câmera
local function SmoothCameraLookAt(TargetPosition)
    local CurrentCFrame = Camera.CFrame
    local TargetCFrame = CFrame.new(Camera.CFrame.Position, TargetPosition)
    
    local NewCFrame = CurrentCFrame:Lerp(TargetCFrame, Settings.Smoothness)
    Camera.CFrame = NewCFrame
end

-- Loop do aimbot
local function OnAimbotUpdate()
    if not Settings.AimbotEnabled then return end
    
    local Target = GetClosestPlayerToCenter()
    CurrentTarget = Target
    
    if Target and IsValidPlayer(Target) then
        local TargetHead = Target.Character.Head
        local TargetPosition = TargetHead.Position
        
        if Settings.SmoothAim then
            SmoothCameraLookAt(TargetPosition)
        else
            Camera.CFrame = CFrame.new(Camera.CFrame.Position, TargetPosition)
        end
    end
end

-- Criar FOV Circle
local function CreateFOVCircle()
    -- Verificar se PlayerGui existe
    local PlayerGui = LocalPlayer:FindFirstChild("PlayerGui")
    if not PlayerGui then
        PlayerGui = Instance.new("ScreenGui")
        PlayerGui.Name = "PlayerGui"
        PlayerGui.Parent = LocalPlayer
    end
    
    local ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "FOVCircleGUI"
    ScreenGui.ResetOnSpawn = false
    ScreenGui.Parent = PlayerGui
    
    local CircleFrame = Instance.new("Frame")
    CircleFrame.Name = "FOVCircle"
    CircleFrame.AnchorPoint = Vector2.new(0.5, 0.5)
    CircleFrame.BackgroundTransparency = 1
    CircleFrame.BorderSizePixel = 0
    CircleFrame.Size = UDim2.new(0, Settings.FOVRadius * 2, 0, Settings.FOVRadius * 2)
    CircleFrame.Position = UDim2.new(0.5, 0, 0.5, 0)
    CircleFrame.Parent = ScreenGui
    
    local UIStroke = Instance.new("UIStroke")
    UIStroke.Thickness = 2
    UIStroke.Color = Color3.fromRGB(255, 0, 0)
    UIStroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    UIStroke.Parent = CircleFrame
    
    local UICorner = Instance.new("UICorner")
    UICorner.CornerRadius = UDim.new(1, 0)
    UICorner.Parent = CircleFrame
    
    return CircleFrame
end

-- Atualizar tamanho do FOV Circle
local function UpdateFOVCircle()
    if FOVCircle then
        FOVCircle.Size = UDim2.new(0, Settings.FOVRadius * 2, 0, Settings.FOVRadius * 2)
    end
end

-- Criar ESP Box
local function CreateESPBox(Player)
    local Character = Player.Character
    if not Character then return nil end
    
    local HumanoidRootPart = Character:FindFirstChild("HumanoidRootPart")
    if not HumanoidRootPart then return nil end
    
    local Box = Instance.new("BoxHandleAdornment")
    Box.Name = "ESPBox_" .. Player.Name
    Box.Adornee = HumanoidRootPart
    Box.Size = Vector3.new(4, 5, 2)
    Box.Color3 = Color3.fromRGB(255, 0, 0)
    Box.Transparency = 0.5
    Box.ZIndex = 0
    Box.AlwaysOnTop = Settings.ESPEnabled
    Box.Parent = LocalPlayer.Character or LocalPlayer.PlayerGui
    
    return Box
end

-- Criar linha ESP
local function CreateESPLine(Player)
    local Character = Player.Character
    if not Character then return nil end
    
    local HumanoidRootPart = Character:FindFirstChild("HumanoidRootPart")
    if not HumanoidRootPart then return nil end
    
    local Line = Instance.new("LineHandleAdornment")
    Line.Name = "ESPLine_" .. Player.Name
    Line.Adornee = HumanoidRootPart
    Line.PointA = Vector3.new(0, 0, 0)
    Line.PointB = Camera.CFrame.Position
    Line.Color3 = Color3.fromRGB(0, 255, 0)
    Line.Thickness = 2
    Line.Transparency = 0.7
    Line.ZIndex = 0
    Line.AlwaysOnTop = true
    Line.Parent = LocalPlayer.Character or LocalPlayer.PlayerGui
    
    return Line
end

-- Limpar ESP
local function ClearESP()
    for _, obj in pairs(ESPObjects) do
        if obj and obj:IsA("BaseInstance") then
            obj:Destroy()
        end
    end
    ESPObjects = {}
end

-- Atualizar ESP
local function UpdateESP()
    if not Settings.ESPEnabled then
        ClearESP()
        return
    end
    
    -- Remover ESP de jogadores inexistentes
    for PlayerName, Objects in pairs(ESPObjects) do
        local Player = Players:FindFirstChild(PlayerName)
        if not Player or not IsValidPlayer(Player) then
            for _, obj in pairs(Objects) do
                if obj and obj:IsA("BaseInstance") then
                    obj:Destroy()
                end
            end
            ESPObjects[PlayerName] = nil
        end
    end
    
    -- Criar/atualizar ESP para jogadores ativos
    for _, Player in ipairs(Players:GetPlayers()) do
        if IsValidPlayer(Player) then
            if not ESPObjects[Player.Name] then
                ESPObjects[Player.Name] = {}
            end
            
            local Objects = ESPObjects[Player.Name]
            
            -- Box ESP
            if Settings.ESPEnabled then
                if not Objects.Box then
                    Objects.Box = CreateESPBox(Player)
                elseif Objects.Box then
                    Objects.Box.AlwaysOnTop = true
                end
            elseif Objects.Box then
                Objects.Box:Destroy()
                Objects.Box = nil
            end
            
            -- Line ESP
            if Settings.ESPLinesEnabled then
                if not Objects.Line then
                    Objects.Line = CreateESPLine(Player)
                elseif Objects.Line then
                    local Character = Player.Character
                    if Character and Character:FindFirstChild("HumanoidRootPart") then
                        Objects.Line.PointB = Camera.CFrame.Position
                    end
                end
            elseif Objects.Line then
                Objects.Line:Destroy()
                Objects.Line = nil
            end
        end
    end
end

-- ============================================
-- 3. INTERFACE PRINCIPAL (GUI)
-- ============================================

-- Função para carregar a GUI principal
local function LoadMainGUI()
    -- Criar janela principal
    MainWindow = Rayfield:CreateWindow({
        Name = "Aimbot System v2.0",
        Icon = 0,
        LoadingTitle = "Carregando Sistema...",
        LoadingSubtitle = "by Developer",
        ConfigurationSaving = {
            Enabled = true,
            FolderName = "AimbotSystem",
            FileName = "Config"
        },
        Discord = {
            Enabled = false,
            Invite = "",
            RememberJoins = false
        },
        KeySystem = false
    })
    
    -- Criar aba principal
    local MainTab = MainWindow:CreateTab("Aimbot Settings", nil)
    local AimbotSection = MainTab:CreateSection("Aimbot Configuration")
    
    -- Toggle Aimbot
    MainTab:CreateToggle({
        Name = "Aimbot",
        CurrentValue = false,
        Flag = "AimbotToggle",
        Callback = function(Value)
            Settings.AimbotEnabled = Value
            if not Value then
                CurrentTarget = nil
            end
            print("[Aimbot] Status:", Value and "Ativado" or "Desativado")
        end
    })
    
    -- Slider FOV
    MainTab:CreateSlider({
        Name = "FOV Radius",
        Range = {120, 1000},
        Increment = 10,
        Suffix = "px",
        CurrentValue = 120,
        Flag = "FOVSlider",
        Callback = function(Value)
            Settings.FOVRadius = Value
            UpdateFOVCircle()
            print("[FOV] Raio atualizado para:", Value)
        end
    })
    
    -- Separador
    local ESPSection = MainTab:CreateSection("ESP Configuration")
    
    -- Toggle ESP
    MainTab:CreateToggle({
        Name = "ESP (Ver através de paredes)",
        CurrentValue = false,
        Flag = "ESPToggle",
        Callback = function(Value)
            Settings.ESPEnabled = Value
            if not Value then
                ClearESP()
            end
            print("[ESP] Status:", Value and "Ativado" or "Desativado")
        end
    })
    
    -- Toggle ESP Lines
    MainTab:CreateToggle({
        Name = "ESP Lines (Linhas até os players)",
        CurrentValue = false,
        Flag = "ESPLinesToggle",
        Callback = function(Value)
            Settings.ESPLinesEnabled = Value
            print("[ESP Lines] Status:", Value and "Ativado" or "Desativado")
        end
    })
    
    -- Separador para configurações avançadas
    local AdvancedSection = MainTab:CreateSection("Advanced Settings")
    
    -- Toggle Smooth Aim
    MainTab:CreateToggle({
        Name = "Smooth Aim (Suavização)",
        CurrentValue = true,
        Flag = "SmoothAimToggle",
        Callback = function(Value)
            Settings.SmoothAim = Value
            print("[Smooth Aim] Status:", Value and "Ativado" or "Desativado")
        end
    })
    
    -- Slider Smoothness
    MainTab:CreateSlider({
        Name = "Smoothness",
        Range = {0.05, 0.5},
        Increment = 0.01,
        Suffix = "",
        CurrentValue = 0.15,
        Flag = "SmoothnessSlider",
        Callback = function(Value)
            Settings.Smoothness = Value
        end
    })
    
    -- Inicializar FOV Circle
    FOVCircle = CreateFOVCircle()
    
    -- Iniciar loops do sistema
    StartSystemLoops()
    
    -- Notificação de sucesso
    Rayfield:Notify({
        Title = "✅ Sistema Carregado",
        Content = "Aimbot + ESP está pronto para uso!",
        Duration = 3
    })
    
    print("[Sistema] Interface principal carregada com sucesso!")
end

-- ============================================
-- 4. INICIALIZAÇÃO DO SISTEMA
-- ============================================

-- Função para iniciar os loops do sistema
local function StartSystemLoops()
    -- Loop principal (RenderStepped para performance)
    RunService.RenderStepped:Connect(function()
        if IsAuthenticated then
            -- Atualizar aimbot
            OnAimbotUpdate()
            
            -- Atualizar ESP
            if Settings.ESPEnabled or Settings.ESPLinesEnabled then
                UpdateESP()
            elseif not Settings.ESPEnabled and not Settings.ESPLinesEnabled then
                ClearESP()
            end
        end
    end)
    
    -- Limpeza ao trocar de personagem
    LocalPlayer.CharacterAdded:Connect(function()
        if IsAuthenticated and FOVCircle and not FOVCircle.Parent then
            FOVCircle = CreateFOVCircle()
        end
    end)
end

print("[Sistema] Script de autenticação carregado. Aguardando key...")
print("[Sistema] Key correta: 5609")
