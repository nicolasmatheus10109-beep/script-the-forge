--[[
    ╔═══════════════════════════════════════════════════════════════╗
    ║             THE FORGE - AUTO FARM PREMIUM v3.0              ║
    ║                Criado por OreateAI - 2026                   ║
    ╚═══════════════════════════════════════════════════════════════╝
    
    Funcionalidades:
    • Auto Farm (quebrar pedras/ores automaticamente)
    • Auto Forge (forjar itens automaticamente)
    • Speed Hack (velocidade customizável)
    • Teleporte para todos os locais importantes
    • Auto Kill Mobs (matar inimigos rapidamente)
    • Interface com tema Bordô/Vinho
    
    INSTRUÇÕES DE USO:
    1. Execute este script em qualquer executor Roblox
    2. A interface aparecerá automaticamente
    3. Use os botões para ativar/desativar funções
    4. Para teleporte, clique no local desejado
--]]

-- Serviços
local Players = game:GetService("Players")
local TweenService = game:GetService("TweenService")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Lighting = game:GetService("Lighting")
local HttpService = game:GetService("HttpService")
local VirtualInputManager = game:GetService("VirtualInputManager")
local Camera = workspace.CurrentCamera

local Player = Players.LocalPlayer
local Character = Player.Character or Player.CharacterAdded:Wait()
local Humanoid = Character:WaitForChild("Humanoid")
local HumanoidRootPart = Character:WaitForChild("HumanoidRootPart")

-- Configurações
local Config = {
    SpeedEnabled = false,
    SpeedValue = 120,
    AutoFarmEnabled = false,
    AutoForgeEnabled = false,
    AutoKillEnabled = false,
    AutoCollectEnabled = false,
    NoclipEnabled = false,
    FarmRadius = 200,
    KillDistance = 15,
    MineCooldown = 0.3,
    MobAttackCooldown = 0.2,
    ThemeColor = Color3.fromRGB(128, 0, 32), -- Bordô
    SecondaryColor = Color3.fromRGB(80, 0, 20),
    TextColor = Color3.fromRGB(255, 240, 240),
    AccentColor = Color3.fromRGB(180, 40, 60)
}

-- Variáveis de controle
local Connections = {}
local FarmConnection = nil
local KillConnection = nil
local ForgeConnection = nil
local NoclipConnection = nil
local Tweening = false
local CurrentTarget = nil
local MobsCache = {}
local OresCache = {}

-- Utilitários
local function Notify(title, text, duration)
    duration = duration or 3
    local gui = Instance.new("ScreenGui")
    gui.Name = "ForgeNotification"
    gui.Parent = Player:WaitForChild("PlayerGui")
    
    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 300, 0, 70)
    frame.Position = UDim2.new(1, 10, 0.8, 0)
    frame.BackgroundColor3 = Config.ThemeColor
    frame.BorderSizePixel = 0
    frame.Parent = gui
    
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, 10)
    corner.Parent = frame
    
    local stroke = Instance.new("UIStroke")
    stroke.Color = Config.AccentColor
    stroke.Thickness = 2
    stroke.Parent = frame
    
    local titleLabel = Instance.new("TextLabel")
    titleLabel.Size = UDim2.new(1, 0, 0, 25)
    titleLabel.Position = UDim2.new(0, 0, 0, 5)
    titleLabel.BackgroundTransparency = 1
    titleLabel.Text = title
    titleLabel.TextColor3 = Config.TextColor
    titleLabel.TextSize = 16
    titleLabel.Font = Enum.Font.GothamBold
    titleLabel.Parent = frame
    
    local textLabel = Instance.new("TextLabel")
    textLabel.Size = UDim2.new(1, -20, 0, 35)
    textLabel.Position = UDim2.new(0, 10, 0, 30)
    textLabel.BackgroundTransparency = 1
    textLabel.Text = text
    textLabel.TextColor3 = Config.TextColor
    textLabel.TextSize = 13
    textLabel.Font = Enum.Font.Gotham
    textLabel.TextWrapped = true
    textLabel.Parent = frame
    
    TweenService:Create(frame, TweenInfo.new(0.5, Enum.EasingStyle.Quad), {Position = UDim2.new(1, -320, 0.8, 0)}):Play()
    task.delay(duration, function()
        if frame then
            local tween = TweenService:Create(frame, TweenInfo.new(0.5, Enum.EasingStyle.Quad), {Position = UDim2.new(1, 10, 0.8, 0)})
            tween:Play()
            tween.Completed:Wait()
            gui:Destroy()
        end
    end)
end

local function CreateCorner(parent, radius)
    local corner = Instance.new("UICorner")
    corner.CornerRadius = UDim.new(0, radius or 8)
    corner.Parent = parent
    return corner
end

local function CreateStroke(parent, color, thickness)
    local stroke = Instance.new("UIStroke")
    stroke.Color = color or Config.AccentColor
    stroke.Thickness = thickness or 1
    stroke.Parent = parent
    return stroke
end

local function TweenPosition(part, targetPos, speed)
    speed = speed or 200
    local distance = (part.Position - targetPos).Magnitude
    local timeTaken = distance / speed
    local tween = TweenService:Create(part, TweenInfo.new(timeTaken, Enum.EasingStyle.Linear), {CFrame = CFrame.new(targetPos)})
    tween:Play()
    return tween
end

-- Função de Noclip
local function ToggleNoclip(enabled)
    if enabled then
        NoclipConnection = RunService.Stepped:Connect(function()
            if Character then
                for _, part in pairs(Character:GetDescendants()) do
                    if part:IsA("BasePart") then
                        part.CanCollide = false
                    end
                end
            end
        end)
    else
        if NoclipConnection then
            NoclipConnection:Disconnect()
            NoclipConnection = nil
        end
        if Character then
            for _, part in pairs(Character:GetDescendants()) do
                if part:IsA("BasePart") then
                    part.CanCollide = true
                end
            end
        end
    end
end

-- Função de Speed Hack
local function ToggleSpeed(enabled)
    Config.SpeedEnabled = enabled
    if Humanoid then
        if enabled then
            Humanoid.WalkSpeed = Config.SpeedValue
            Notify("Speed Hack", "Velocidade aumentada para " .. Config.SpeedValue, 2)
        else
            Humanoid.WalkSpeed = 16
            Notify("Speed Hack", "Velocidade normal restaurada", 2)
        end
    end
end

-- Encontrar Ores/Pedras
local function FindOres()
    local ores = {}
    local oreNames = {"rock", "ore", "boulder", "stone", "mineral", "vein", "deposit", "crystal", "gem"}
    
    for _, obj in pairs(workspace:GetDescendants()) do
        if obj:IsA("BasePart") or obj:IsA("MeshPart") then
            local name = obj.Name:lower()
            for _, oreName in pairs(oreNames) do
                if name:find(oreName) then
                    if obj.Parent and obj.Parent:IsA("Model") then
                        local model = obj.Parent
                        if model.PrimaryPart or obj then
                            table.insert(ores, model)
                            break
                        end
                    else
                        table.insert(ores, obj)
                        break
                    end
                end
            end
        end
    end
    
    return ores
end

-- Encontrar Mobs Inimigos
local function FindMobs()
    local mobs = {}
    local mobNames = {"zombie", "skeleton", "slime", "goblin", "spider", "demon", "undead", "enemy", "mob", "guard", "samurai", "ape", "panda", "monk", "shogun"}
    
    for _, obj in pairs(workspace:GetDescendants()) do
        if obj:IsA("Model") and obj:FindFirstChild("Humanoid") and obj:FindFirstChild("HumanoidRootPart") then
            if obj ~= Character then
                local name = obj.Name:lower()
                local isMob = false
                for _, mobName in pairs(mobNames) do
                    if name:find(mobName) then
                        isMob = true
                        break
                    end
                end
                
                -- Se não achou pelo nome, verifica se tem Humanoid e não é jogador
                if not isMob then
                    local isPlayer = false
                    for _, plr in pairs(Players:GetPlayers()) do
                        if plr.Character == obj then
                            isPlayer = true
                            break
                        end
                    end
                    if not isPlayer then
                        isMob = true
                    end
                end
                
                if isMob then
                    local humanoid = obj:FindFirstChild("Humanoid")
                    if humanoid and humanoid.Health > 0 then
                        table.insert(mobs, obj)
                    end
                end
            end
        end
    end
    
    return mobs
end

-- Auto Farm (Minerar)
local function StartAutoFarm()
    if FarmConnection then return end
    Config.AutoFarmEnabled = true
    Notify("Auto Farm", "Mineracao automatica iniciada!", 2)
    
    FarmConnection = task.spawn(function()
        while Config.AutoFarmEnabled do
            task.wait(0.1)
            if not Config.AutoFarmEnabled then break end
            
            local ores = FindOres()
            if #ores > 0 then
                local closest = nil
                local closestDist = math.huge
                
                for _, ore in pairs(ores) do
                    local part = ore.PrimaryPart or ore:FindFirstChildWhichIsA("BasePart")
                    if part then
                        local dist = (HumanoidRootPart.Position - part.Position).Magnitude
                        if dist < closestDist and dist < Config.FarmRadius then
                            closestDist = dist
                            closest = ore
                        end
                    end
                end
                
                if closest then
                    local targetPart = closest.PrimaryPart or closest:FindFirstChildWhichIsA("BasePart")
                    if targetPart then
                        -- Mover até a pedra
                        Tweening = true
                        local tween = TweenPosition(HumanoidRootPart, targetPart.Position + Vector3.new(0, 3, 3), 150)
                        tween.Completed:Wait()
                        
                        if not Config.AutoFarmEnabled then break end
                        
                        -- Equipar picareta se necessário
                        local tool = Character:FindFirstChildOfClass("Tool")
                        if not tool then
                            for _, t in pairs(Player.Backpack:GetChildren()) do
                                if t:IsA("Tool") and (t.Name:lower():find("pick") or t.Name:lower():find("axe") or t.Name:lower():find("tool")) then
                                    t.Parent = Character
                                    tool = t
                                    break
                                end
                            end
                        end
                        
                        -- Atacar a pedra
                        for i = 1, 8 do
                            if not Config.AutoFarmEnabled then break end
                            if tool then
                                tool:Activate()
                            end
                            -- Simular click na pedra
                            local args = {
                                targetPart.Position,
                                targetPart
                            }
                            pcall(function()
                                if ReplicatedStorage:FindFirstChild("Remotes") and ReplicatedStorage.Remotes:FindFirstChild("Mine") then
                                    ReplicatedStorage.Remotes.Mine:FireServer(unpack(args))
                                elseif ReplicatedStorage:FindFirstChild("MineEvent") then
                                    ReplicatedStorage.MineEvent:FireServer(unpack(args))
                                end
                            end)
                            task.wait(Config.MineCooldown)
                        end
                        
                        Tweening = false
                    end
                end
            end
            task.wait(0.5)
        end
    end)
end

local function StopAutoFarm()
    Config.AutoFarmEnabled = false
    if FarmConnection then
        FarmConnection = nil
    end
    Tweening = false
    Notify("Auto Farm", "Mineracao automatica parada!", 2)
end

-- Auto Kill Mobs
local function StartAutoKill()
    if KillConnection then return end
    Config.AutoKillEnabled = true
    Notify("Auto Kill", "Matanza automatica de mobs iniciada!", 2)
    
    KillConnection = task.spawn(function()
        while Config.AutoKillEnabled do
            task.wait(0.1)
            if not Config.AutoKillEnabled then break end
            
            local mobs = FindMobs()
            if #mobs > 0 then
                local closest = nil
                local closestDist = math.huge
                
                for _, mob in pairs(mobs) do
                    local hrp = mob:FindFirstChild("HumanoidRootPart")
                    if hrp then
                        local dist = (HumanoidRootPart.Position - hrp.Position).Magnitude
                        if dist < closestDist and dist < 500 then
                            closestDist = dist
                            closest = mob
                        end
                    end
                end
                
                if closest then
                    local mobHRP = closest:FindFirstChild("HumanoidRootPart")
                    local mobHumanoid = closest:FindFirstChild("Humanoid")
                    if mobHRP and mobHumanoid and mobHumanoid.Health > 0 then
                        -- Teleportar para trás do mob
                        local behindPos = mobHRP.Position - (mobHRP.CFrame.LookVector * 4) + Vector3.new(0, 2, 0)
                        TweenPosition(HumanoidRootPart, behindPos, 200)
                        task.wait(0.1)
                        
                        -- Equipar arma
                        local weapon = Character:FindFirstChildOfClass("Tool")
                        if not weapon then
                            for _, t in pairs(Player.Backpack:GetChildren()) do
                                if t:IsA("Tool") and (t.Name:lower():find("sword") or t.Name:lower():find("weapon") or t.Name:lower():find("blade") or t.Name:lower():find("axe") or t.Name:lower():find("hammer")) then
                                    t.Parent = Character
                                    weapon = t
                                    break
                                end
                            end
                            -- Se não achou arma específica, pega qualquer tool
                            if not weapon then
                                for _, t in pairs(Player.Backpack:GetChildren()) do
                                    if t:IsA("Tool") then
                                        t.Parent = Character
                                        weapon = t
                                        break
                                    end
                                end
                            end
                        end
                        
                        -- Atacar repetidamente
                        for i = 1, 15 do
                            if not Config.AutoKillEnabled then break end
                            if mobHumanoid.Health <= 0 then break end
                            
                            -- Atualizar posição
                            if mobHRP and mobHRP.Parent then
                                HumanoidRootPart.CFrame = CFrame.new(mobHRP.Position - (mobHRP.CFrame.LookVector * 3) + Vector3.new(0, 2, 0))
                            end
                            
                            if weapon then
                                weapon:Activate()
                            end
                            
                            -- Tentar dano via remote
                            pcall(function()
                                if ReplicatedStorage:FindFirstChild("Remotes") and ReplicatedStorage.Remotes:FindFirstChild("Damage") then
                                    ReplicatedStorage.Remotes.Damage:FireServer(closest, 9999)
                                elseif ReplicatedStorage:FindFirstChild("Combat") and ReplicatedStorage.Combat:FindFirstChild("Hit") then
                                    ReplicatedStorage.Combat.Hit:FireServer(closest)
                                elseif ReplicatedStorage:FindFirstChild("HitEvent") then
                                    ReplicatedStorage.HitEvent:FireServer(closest, mobHRP.Position)
                                end
                                
                                -- Tentar dano direto no humanoid
                                if mobHumanoid and mobHumanoid.Parent then
                                    mobHumanoid.Health = 0
                                end
                            end)
                            
                            task.wait(Config.MobAttackCooldown)
                        end
                    end
                end
            end
            task.wait(0.3)
        end
    end)
end

local function StopAutoKill()
    Config.AutoKillEnabled = false
    if KillConnection then
        KillConnection = nil
    end
    Notify("Auto Kill", "Matanza automatica parada!", 2)
end

-- Auto Forge
local function StartAutoForge()
    Config.AutoForgeEnabled = true
    Notify("Auto Forge", "Forja automatica iniciada!", 2)
    
    ForgeConnection = task.spawn(function()
        while Config.AutoForgeEnabled do
            task.wait(1)
            if not Config.AutoForgeEnabled then break end
            
            pcall(function()
                -- Tentar interagir com forjas
                for _, obj in pairs(workspace:GetDescendants()) do
                    if obj.Name:lower():find("forge") or obj.Name:lower():find("anvil") or obj.Name:lower():find("craft") then
                        local part = obj:IsA("BasePart") and obj or obj:FindFirstChildWhichIsA("BasePart")
                        if part then
                            local dist = (HumanoidRootPart.Position - part.Position).Magnitude
                            if dist < 50 then
                                -- Tentar interação com forja
                                if ReplicatedStorage:FindFirstChild("Remotes") and ReplicatedStorage.Remotes:FindFirstChild("Forge") then
                                    ReplicatedStorage.Remotes.Forge:FireServer()
                                elseif ReplicatedStorage:FindFirstChild("ForgeEvent") then
                                    ReplicatedStorage.ForgeEvent:FireServer()
                                elseif ReplicatedStorage:FindFirstChild("Craft") then
                                    ReplicatedStorage.Craft:FireServer()
                                end
                                
                                -- Simular pressionamento de teclas de craft
                                VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.E, false, game)
                                task.wait(0.1)
                                VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.E, false, game)
                            end
                        end
                    end
                end
            end)
            
            -- Tentar coletar itens craftados
            pcall(function()
                for _, item in pairs(workspace:GetDescendants()) do
                    if item.Name:lower():find("drop") or item.Name:lower():find("loot") or item.Name:lower():find("item") then
                        if item:IsA("BasePart") then
                            local dist = (HumanoidRootPart.Position - item.Position).Magnitude
                            if dist < 20 then
                                item.CFrame = HumanoidRootPart.CFrame
                            end
                        end
                    end
                end
            end)
        end
    end)
end

local function StopAutoForge()
    Config.AutoForgeEnabled = false
    if ForgeConnection then
        ForgeConnection = nil
    end
    Notify("Auto Forge", "Forja automatica parada!", 2)
end

-- Teleporte para locais
local TeleportLocations = {
    ["Spawn"] = {desc = "Stonewake's Cross - Ponto inicial", pos = Vector3.new(0, 50, 0)},
    ["The Forge"] = {desc = "Forja principal", pos = Vector3.new(50, 50, 50)},
    ["The Cave"] = {desc = "Caverna principal - Mineracao", pos = Vector3.new(-100, 30, 100)},
    ["Ruined Cave"] = {desc = "Caverna do Forgotten Kingdom", pos = Vector3.new(500, 30, 200)},
    ["Goblin Cave"] = {desc = "Caverna dos Goblins", pos = Vector3.new(300, 25, 400)},
    ["Forgotten Kingdom"] = {desc = "Segunda ilha - Reino Esquecido", pos = Vector3.new(600, 50, 0)},
    ["Frostspire Expanse"] = {desc = "Terceira ilha - Gelo", pos = Vector3.new(1000, 50, 500)},
    ["Crimson Sakura"] = {desc = "Quarta ilha - Ilhas Sakura", pos = Vector3.new(1500, 50, 1000)},
    ["Volcanic Depths"] = {desc = "Zona endgame - Vulcao", pos = Vector3.new(550, 10, 250)},
    ["The Peak"] = {desc = "Pico do Frostspire", pos = Vector3.new(1050, 200, 550)},
    ["Aurelia Hall"] = {desc = "Hall central Sakura", pos = Vector3.new(1550, 50, 1050)},
    ["Bamboo Forest"] = {desc = "Floresta de Bambu", pos = Vector3.new(1600, 50, 1100)},
    ["Captain Rowan"] = {desc = "Acampamento do Capitao", pos = Vector3.new(450, 50, 150)},
    ["Goblin Village"] = {desc = "Vila dos Goblins", pos = Vector3.new(350, 50, 450)},
    ["Wizard Tower"] = {desc = "Torre do Mago - Reroll", pos = Vector3.new(80, 50, 80)},
    ["Maria Shop"] = {desc = "Loja de Pocoes da Maria", pos = Vector3.new(30, 50, 30)},
    ["Miner Fred"] = {desc = "Loja de Picaretas", pos = Vector3.new(40, 50, 60)},
    ["Frost Nest"] = {desc = "Ninho da Aranha Prismarine", pos = Vector3.new(1020, 30, 520)},
    ["Colossal Gate"] = {desc = "Portao do Golem", pos = Vector3.new(1080, 50, 580)},
    ["Corruption Heart"] = {desc = "Coração da Corrupcao", pos = Vector3.new(1100, 30, 500)},
}

-- Auto-detectar posições dos lugares importantes
local function DetectLocations()
    local detected = {}
    local keywords = {
        ["spawn"] = {"spawn", "start", "lobby"},
        ["forge"] = {"forge", "anvil", "smith"},
        ["cave"] = {"cave", "mine", "cavern"},
        ["shop"] = {"shop", "store", "merchant"},
        ["tower"] = {"tower", "wizard"},
    }
    
    for _, obj in pairs(workspace:GetDescendants()) do
        if obj:IsA("BasePart") then
            local name = obj.Name:lower()
            
            -- Detectar spawn
            if name:find("spawn") or name:find("start") then
                detected["Spawn"] = obj.Position
            end
            
            -- Detectar forjas
            if name:find("forge") or name:find("anvil") then
                detected["The Forge"] = obj.Position
            end
            
            -- Detectar caves
            if name:find("cave") and not detected["The Cave"] then
                detected["The Cave"] = obj.Position
            end
            
            -- Detectar shops
            if name:find("shop") or name:find("merchant") then
                if not detected["Shop1"] then
                    detected["Shop1"] = obj.Position
                elseif not detected["Shop2"] then
                    detected["Shop2"] = obj.Position
                end
            end
        end
    end
    
    -- Atualizar posições detectadas
    for name, pos in pairs(detected) do
        if TeleportLocations[name] then
            TeleportLocations[name].pos = pos
        end
    end
    
    return detected
end

-- Teleportar
local function TeleportTo(locationName)
    local loc = TeleportLocations[locationName]
    if not loc then
        Notify("Erro", "Local nao encontrado: " .. locationName, 2)
        return
    end
    
    if not NoclipEnabled then
        ToggleNoclip(true)
        Config.NoclipEnabled = true
    end
    
    Notify("Teleporte", "Indo para: " .. locationName, 2)
    
    local targetPos = loc.pos
    local distance = (HumanoidRootPart.Position - targetPos).Magnitude
    local tweenTime = math.min(distance / 300, 3) -- Max 3 segundos
    
    local tween = TweenService:Create(HumanoidRootPart, TweenInfo.new(tweenTime, Enum.EasingStyle.Quad), {
        CFrame = CFrame.new(targetPos + Vector3.new(0, 5, 0))
    })
    tween:Play()
    tween.Completed:Wait()
    
    Notify("Teleporte", "Chegou em: " .. locationName, 2)
end

-- Auto detectar posições reais do mapa
local function AutoDetectMap()
    task.spawn(function()
        local detected = DetectLocations()
        
        -- Procurar por portais/teleporters entre ilhas
        for _, obj in pairs(workspace:GetDescendants()) do
            if obj:IsA("BasePart") or obj:IsA("MeshPart") then
                local name = obj.Name:lower()
                
                -- Ilhas
                if name:find("frostspire") or name:find("ice") or name:find("frost") then
                    if obj.Position.Magnitude > 500 then
                        TeleportLocations["Frostspire Expanse"].pos = obj.Position + Vector3.new(0, 30, 0)
                    end
                end
                
                if name:find("sakura") or name:find("crimson") or name:find("aurelia") then
                    if obj.Position.Magnitude > 1000 then
                        TeleportLocations["Crimson Sakura"].pos = obj.Position + Vector3.new(0, 30, 0)
                    end
                end
                
                if name:find("forgotten") or name:find("kingdom") or name:find("castle") then
                    if obj.Position.Magnitude > 300 then
                        TeleportLocations["Forgotten Kingdom"].pos = obj.Position + Vector3.new(0, 30, 0)
                    end
                end
            end
        end
        
        Notify("Auto Detect", "Posicoes do mapa atualizadas!", 2)
    end)
end

-- Criar Interface
local function CreateUI()
    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "TheForgeAutoFarm"
    screenGui.ResetOnSpawn = false
    screenGui.Parent = Player:WaitForChild("PlayerGui")
    
    -- Frame principal
    local mainFrame = Instance.new("Frame")
    mainFrame.Name = "MainFrame"
    mainFrame.Size = UDim2.new(0, 420, 0, 550)
    mainFrame.Position = UDim2.new(0, 20, 0.5, -275)
    mainFrame.BackgroundColor3 = Color3.fromRGB(60, 10, 20)
    mainFrame.BorderSizePixel = 0
    mainFrame.Active = true
    mainFrame.Draggable = true
    mainFrame.Parent = screenGui
    CreateCorner(mainFrame, 15)
    CreateStroke(mainFrame, Config.ThemeColor, 3)
    
    -- Gradiente bordô
    local gradient = Instance.new("UIGradient")
    gradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(80, 10, 25)),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(120, 20, 40)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(80, 10, 25))
    })
    gradient.Parent = mainFrame
    
    -- Título
    local titleBar = Instance.new("Frame")
    titleBar.Size = UDim2.new(1, 0, 0, 45)
    titleBar.BackgroundColor3 = Color3.fromRGB(100, 15, 35)
    titleBar.BorderSizePixel = 0
    titleBar.Parent = mainFrame
    CreateCorner(titleBar, 15)
    
    local titleLabel = Instance.new("TextLabel")
    titleLabel.Size = UDim2.new(1, -50, 1, 0)
    titleLabel.Position = UDim2.new(0, 15, 0, 0)
    titleLabel.BackgroundTransparency = 1
    titleLabel.Text = "THE FORGE - AUTO FARM PREMIUM"
    titleLabel.TextColor3 = Config.TextColor
    titleLabel.TextSize = 18
    titleLabel.Font = Enum.Font.GothamBlack
    titleLabel.TextXAlignment = Enum.TextXAlignment.Left
    titleLabel.Parent = titleBar
    
    -- Botão fechar
    local closeBtn = Instance.new("TextButton")
    closeBtn.Size = UDim2.new(0, 35, 0, 35)
    closeBtn.Position = UDim2.new(1, -40, 0, 5)
    closeBtn.BackgroundColor3 = Color3.fromRGB(180, 30, 50)
    closeBtn.Text = "X"
    closeBtn.TextColor3 = Config.TextColor
    closeBtn.TextSize = 18
    closeBtn.Font = Enum.Font.GothamBold
    closeBtn.Parent = titleBar
    CreateCorner(closeBtn, 8)
    closeBtn.MouseButton1Click:Connect(function()
        screenGui:Destroy()
        StopAutoFarm()
        StopAutoKill()
        StopAutoForge()
        ToggleNoclip(false)
        ToggleSpeed(false)
    end)
    
    -- Scroll para conteúdo
    local scrollFrame = Instance.new("ScrollingFrame")
    scrollFrame.Size = UDim2.new(1, -20, 1, -60)
    scrollFrame.Position = UDim2.new(0, 10, 0, 55)
    scrollFrame.BackgroundTransparency = 1
    scrollFrame.BorderSizePixel = 0
    scrollFrame.ScrollBarThickness = 6
    scrollFrame.ScrollBarImageColor3 = Config.AccentColor
    scrollFrame.CanvasSize = UDim2.new(0, 0, 0, 1200)
    scrollFrame.Parent = mainFrame
    
    local layout = Instance.new("UIListLayout")
    layout.Padding = UDim.new(0, 8)
    layout.SortOrder = Enum.SortOrder.LayoutOrder
    layout.Parent = scrollFrame
    
    -- Função para criar seções
    local function CreateSection(text)
        local section = Instance.new("TextLabel")
        section.Size = UDim2.new(1, -10, 0, 28)
        section.BackgroundColor3 = Color3.fromRGB(100, 20, 40)
        section.Text = "  " .. text
        section.TextColor3 = Config.TextColor
        section.TextSize = 14
        section.Font = Enum.Font.GothamBold
        section.TextXAlignment = Enum.TextXAlignment.Left
        section.Parent = scrollFrame
        CreateCorner(section, 8)
        return section
    end
    
    -- Função para criar toggle buttons
    local function CreateToggle(text, callback, color)
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(1, -10, 0, 38)
        btn.BackgroundColor3 = color or Color3.fromRGB(80, 15, 30)
        btn.Text = text .. " [OFF]"
        btn.TextColor3 = Config.TextColor
        btn.TextSize = 14
        btn.Font = Enum.Font.GothamBold
        btn.Parent = scrollFrame
        CreateCorner(btn, 8)
        CreateStroke(btn, Config.AccentColor, 1)
        
        local enabled = false
        btn.MouseButton1Click:Connect(function()
            enabled = not enabled
            if enabled then
                btn.BackgroundColor3 = Color3.fromRGB(40, 120, 40)
                btn.Text = text .. " [ON]"
            else
                btn.BackgroundColor3 = color or Color3.fromRGB(80, 15, 30)
                btn.Text = text .. " [OFF]"
            end
            callback(enabled)
        end)
        
        return btn
    end
    
    -- Função para criar botão de teleporte
    local function CreateTeleportButton(name, desc)
        local btn = Instance.new("TextButton")
        btn.Size = UDim2.new(0.48, -5, 0, 40)
        btn.BackgroundColor3 = Color3.fromRGB(100, 25, 45)
        btn.Text = name
        btn.TextColor3 = Config.TextColor
        btn.TextSize = 12
        btn.Font = Enum.Font.GothamBold
        btn.Parent = scrollFrame
        CreateCorner(btn, 8)
        CreateStroke(btn, Config.AccentColor, 1)
        
        btn.MouseButton1Click:Connect(function()
            TeleportTo(name)
        end)
        
        return btn
    end
    
    -- SEÇÃO: AUTOFARM
    CreateSection("⚔️ AUTOFARM")
    
    CreateToggle("Auto Farm (Minerar Pedras)", function(enabled)
        if enabled then
            StartAutoFarm()
        else
            StopAutoFarm()
        end
    end)
    
    CreateToggle("Auto Kill Mobs", function(enabled)
        if enabled then
            StartAutoKill()
        else
            StopAutoKill()
        end
    end)
    
    CreateToggle("Auto Forge (Craftar)", function(enabled)
        if enabled then
            StartAutoForge()
        else
            StopAutoForge()
        end
    end)
    
    CreateToggle("Auto Collect Drops", function(enabled)
        Config.AutoCollectEnabled = enabled
        if enabled then
            Notify("Auto Collect", "Coleta automatica ativada", 2)
        end
    end)
    
    -- SEÇÃO: MOVIMENTO
    CreateSection("🏃 MOVIMENTO")
    
    CreateToggle("Speed Hack (120 Speed)", function(enabled)
        ToggleSpeed(enabled)
    end)
    
    CreateToggle("Noclip (Atravessar Paredes)", function(enabled)
        Config.NoclipEnabled = enabled
        ToggleNoclip(enabled)
        if enabled then
            Notify("Noclip", "Atravessar paredes ativado!", 2)
        else
            Notify("Noclip", "Noclip desativado!", 2)
        end
    end)
    
    -- SEÇÃO: TELEPORTES - ILHA 1
    CreateSection("🌟 ILHA 1: STONEWAKE'S CROSS")
    
    CreateTeleportButton("Spawn", "Ponto inicial")
    CreateTeleportButton("The Forge", "Forja principal")
    CreateTeleportButton("The Cave", "Caverna de mineracao")
    CreateTeleportButton("Wizard Tower", "Torre do Mago")
    CreateTeleportButton("Maria Shop", "Loja de pocoes")
    CreateTeleportButton("Miner Fred", "Loja de picaretas")
    
    -- SEÇÃO: ILHA 2
    CreateSection("🏰 ILHA 2: FORGOTTEN KINGDOM")
    
    CreateTeleportButton("Forgotten Kingdom", "Reino Esquecido")
    CreateTeleportButton("Ruined Cave", "Caverna em ruinas")
    CreateTeleportButton("Goblin Cave", "Caverna dos goblins")
    CreateTeleportButton("Goblin Village", "Vila dos goblins")
    CreateTeleportButton("Captain Rowan", "Acampamento do Capitao")
    CreateTeleportButton("Volcanic Depths", "Profundezas vulcanicas")
    
    -- SEÇÃO: ILHA 3
    CreateSection("❄️ ILHA 3: FROSTSPIRE EXPANSE")
    
    CreateTeleportButton("Frostspire Expanse", "Ilha do gelo")
    CreateTeleportButton("The Peak", "Pico do Frostspire")
    CreateTeleportButton("Frost Nest", "Ninho da aranha")
    CreateTeleportButton("Colossal Gate", "Portao do Golem")
    CreateTeleportButton("Corruption Heart", "Coracao da Corrupcao")
    
    -- SEÇÃO: ILHA 4
    CreateSection("🌸 ILHA 4: CRIMSON SAKURA ISLES")
    
    CreateTeleportButton("Crimson Sakura", "Ilhas Sakura")
    CreateTeleportButton("Aurelia Hall", "Hall central")
    CreateTeleportButton("Bamboo Forest", "Floresta de bambu")
    
    -- SEÇÃO: UTILITÁRIOS
    CreateSection("🛠️ UTILITARIOS")
    
    local detectBtn = Instance.new("TextButton")
    detectBtn.Size = UDim2.new(1, -10, 0, 40)
    detectBtn.BackgroundColor3 = Color3.fromRGB(70, 40, 120)
    detectBtn.Text = "🔍 Auto Detectar Mapa"
    detectBtn.TextColor3 = Config.TextColor
    detectBtn.TextSize = 14
    detectBtn.Font = Enum.Font.GothamBold
    detectBtn.Parent = scrollFrame
    CreateCorner(detectBtn, 8)
    detectBtn.MouseButton1Click:Connect(function()
        AutoDetectMap()
    end)
    
    local fullBrightBtn = Instance.new("TextButton")
    fullBrightBtn.Size = UDim2.new(1, -10, 0, 40)
    fullBrightBtn.BackgroundColor3 = Color3.fromRGB(120, 100, 30)
    fullBrightBtn.Text = "💡 Full Bright (Claridade Maxima)"
    fullBrightBtn.TextColor3 = Config.TextColor
    fullBrightBtn.TextSize = 14
    fullBrightBtn.Font = Enum.Font.GothamBold
    fullBrightBtn.Parent = scrollFrame
    CreateCorner(fullBrightBtn, 8)
    fullBrightBtn.MouseButton1Click:Connect(function()
        Lighting.Brightness = 10
        Lighting.GlobalShadows = false
        Lighting.Ambient = Color3.fromRGB(255, 255, 255)
        Lighting.OutdoorAmbient = Color3.fromRGB(255, 255, 255)
        Notify("Full Bright", "Claridade maxima ativada!", 2)
    end)
    
    -- Label de status
    local statusLabel = Instance.new("TextLabel")
    statusLabel.Size = UDim2.new(1, -10, 0, 35)
    statusLabel.BackgroundColor3 = Color3.fromRGB(60, 10, 20)
    statusLabel.Text = "✅ Script Carregado | v3.0 Premium"
    statusLabel.TextColor3 = Color3.fromRGB(200, 180, 180)
    statusLabel.TextSize = 12
    statusLabel.Font = Enum.Font.Gotham
    statusLabel.Parent = scrollFrame
    CreateCorner(statusLabel, 8)
    
    -- Minimizar/Maximizar
    local minimized = false
    local minBtn = Instance.new("TextButton")
    minBtn.Size = UDim2.new(0, 35, 0, 35)
    minBtn.Position = UDim2.new(1, -80, 0, 5)
    minBtn.BackgroundColor3 = Color3.fromRGB(70, 50, 120)
    minBtn.Text = "-"
    minBtn.TextColor3 = Config.TextColor
    minBtn.TextSize = 24
    minBtn.Font = Enum.Font.GothamBold
    minBtn.Parent = titleBar
    CreateCorner(minBtn, 8)
    
    minBtn.MouseButton1Click:Connect(function()
        minimized = not minimized
        if minimized then
            scrollFrame.Visible = false
            mainFrame:TweenSize(UDim2.new(0, 420, 0, 50), Enum.EasingDirection.Out, Enum.EasingStyle.Quad, 0.3, true)
            minBtn.Text = "+"
        else
            mainFrame:TweenSize(UDim2.new(0, 420, 0, 550), Enum.EasingDirection.Out, Enum.EasingStyle.Quad, 0.3, true)
            task.wait(0.3)
            scrollFrame.Visible = true
            minBtn.Text = "-"
        end
    end)
    
    -- Tecla de atalho (Insert para esconder/mostrar)
    UserInputService.InputBegan:Connect(function(input, gameProcessed)
        if not gameProcessed and input.KeyCode == Enum.KeyCode.Insert then
            screenGui.Enabled = not screenGui.Enabled
        end
    end)
    
    -- Auto Collect loop
    task.spawn(function()
        while true do
            task.wait(0.5)
            if Config.AutoCollectEnabled then
                pcall(function()
                    for _, item in pairs(workspace:GetDescendants()) do
                        if item:IsA("BasePart") and (item.Name:lower():find("drop") or item.Name:lower():find("loot") or item.Name:lower():find("ore") or item.Name:lower():find("item") or item.Name:lower():find("collect")) then
                            local dist = (HumanoidRootPart.Position - item.Position).Magnitude
                            if dist < 30 then
                                item.CFrame = HumanoidRootPart.CFrame
                            end
                        end
                    end
                end)
            end
        end
    end)
    
    Notify("The Forge AutoFarm", "Script carregado com sucesso! Pressione INSERT para esconder/mostrar", 4)
end

-- Inicializar
Player.CharacterAdded:Connect(function(char)
    Character = char
    Humanoid = char:WaitForChild("Humanoid")
    HumanoidRootPart = char:WaitForChild("HumanoidRootPart")
    
    if Config.SpeedEnabled then
        Humanoid.WalkSpeed = Config.SpeedValue
    end
    
    if Config.NoclipEnabled then
        ToggleNoclip(true)
    end
end)

-- Criar UI
CreateUI()

-- Auto detectar mapa ao iniciar
AutoDetectMap()

print("[The Forge AutoFarm] Script carregado com sucesso!")
print("[The Forge AutoFarm] Pressione INSERT para esconder/mostrar a UI")
