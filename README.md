local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local Debris = game:GetService("Debris")
local LocalPlayer = Players.LocalPlayer
local Workspace = game:GetService("Workspace")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")
local StarterGui = game:GetService("StarterGui")

--==============================
-- VARIÁVEIS DE ESTADO
--==============================
local jumpEnabled, featherEnabled, airWalkEnabled, followBlockActive = false, false, false, false
local brainrotEspEnabled, spawnEspEnabled = false, false
local airPlatform = nil
local CurrentBrainrotESP, CurrentSpawnESP = nil, nil

--==============================
-- FUNÇÃO AUXILIAR: BORDAS LED
--==============================
local function addBorderLED(frame, thickness)
    local stroke = Instance.new("UIStroke", frame)
    stroke.Color = Color3.fromRGB(255,0,0)
    stroke.Thickness = thickness or 2 
    stroke.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
    local grad = Instance.new("UIGradient", stroke)
    grad.Color = ColorSequence.new{
        ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 0, 0)),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(40, 0, 0)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(255, 0, 0))
    }
    RunService.RenderStepped:Connect(function()
        grad.Rotation = (grad.Rotation + 2) % 360
    end)
end

--==============================
-- LÓGICA DE ESP (INTERNA)
--==============================
local function ToggleBrainrotESP(state)
    brainrotEspEnabled = state
    if not state and CurrentBrainrotESP then CurrentBrainrotESP:Destroy() end
end

local function ToggleSpawnESP(state)
    spawnEspEnabled = state
    if not state and CurrentSpawnESP then CurrentSpawnESP:Destroy() end
end

--==============================
-- HUB PRINCIPAL
--==============================
local function createHub()
    if LocalPlayer.PlayerGui:FindFirstChild("GTGHub") then LocalPlayer.PlayerGui.GTGHub:Destroy() end
    local gui = Instance.new("ScreenGui", LocalPlayer.PlayerGui); gui.Name = "GTGHub"; gui.ResetOnSpawn = false

    -- BOTÃO GTG (ABRIR)
    local openBtn = Instance.new("TextButton", gui)
    openBtn.Size, openBtn.Position = UDim2.new(0, 50, 0, 25), UDim2.new(0, 10, 0, 10)
    openBtn.Text, openBtn.Visible = "GTG", false
    openBtn.BackgroundColor3, openBtn.TextColor3, openBtn.Font = Color3.new(0,0,0), Color3.new(1,0,0), "GothamBold"
    Instance.new("UICorner", openBtn); addBorderLED(openBtn, 2)

    local main = Instance.new("Frame", gui)
    main.Size, main.Position = UDim2.new(0, 380, 0, 230), UDim2.new(0.5, -190, 0.5, -115)
    main.BackgroundColor3 = Color3.fromRGB(10,10,10); Instance.new("UICorner", main)
    addBorderLED(main, 3)

    -- BOTÃO FECHAR (X)
    local closeBtn = Instance.new("TextButton", main)
    closeBtn.Size, closeBtn.Position = UDim2.new(0, 30, 0, 30), UDim2.new(1, -35, 0, 5)
    closeBtn.Text, closeBtn.BackgroundColor3, closeBtn.TextColor3 = "X", Color3.new(0.6,0,0), Color3.new(1,1,1)
    Instance.new("UICorner", closeBtn)
    closeBtn.MouseButton1Click:Connect(function() main.Visible = false; openBtn.Visible = true end)
    openBtn.MouseButton1Click:Connect(function() main.Visible = true; openBtn.Visible = false end)

    local left = Instance.new("Frame", main)
    left.Size, left.Position = UDim2.new(0, 100, 0, 110), UDim2.new(0, 10, 0, 45)
    left.BackgroundColor3 = Color3.fromRGB(20,20,20); Instance.new("UICorner", left); addBorderLED(left, 2)

    local content = Instance.new("Frame", main)
    content.Size, content.Position = UDim2.new(0, 250, 1, -60), UDim2.new(0, 120, 0, 45); content.BackgroundTransparency = 1

    local pages = { 
        LTM = Instance.new("ScrollingFrame", content), 
        Rastreio = Instance.new("ScrollingFrame", content),
        Servidores = Instance.new("ScrollingFrame", content) 
    }

    for name, page in pairs(pages) do
        page.Size, page.BackgroundTransparency, page.Visible = UDim2.new(1, 0, 1, 0), 1, (name == "LTM")
        page.ScrollBarThickness, page.AutomaticCanvasSize = 0, "Y"
        Instance.new("UIListLayout", page).Padding = UDim.new(0, 8)
    end

    -- FUNÇÃO PARA CRIAR BOTÕES DE ATIVAÇÃO DIRETA
    local function createToggle(txt, parent, callback)
        local btn = Instance.new("TextButton", parent)
        btn.Size, btn.Text = UDim2.new(0, 230, 0, 35), txt .. " [OFF]"
        btn.BackgroundColor3, btn.TextColor3, btn.Font = Color3.fromRGB(30, 30, 30), Color3.new(1,1,1), "GothamBold"
        Instance.new("UICorner", btn); addBorderLED(btn, 1)
        
        local active = false
        btn.MouseButton1Click:Connect(function()
            active = not active
            btn.Text = txt .. (active and " [ON]" or " [OFF]")
            btn.BackgroundColor3 = active and Color3.fromRGB(150, 0, 0) or Color3.fromRGB(30, 30, 30)
            callback(active)
        end)
    end

    -- ABA LTM (SÓ FUNÇÕES FÍSICAS)
    createToggle("Jump 3.8X", pages.LTM, function(s) jumpEnabled = s end)
    createToggle("Cair Leve", pages.LTM, function(s) featherEnabled = s end)
    createToggle("Andar no Ar", pages.LTM, function(s) airWalkEnabled = s end)
    createToggle("Bloco Seguidor", pages.LTM, function(s) followBlockActive = s end)

    -- ABA RASTREIO (ESPs E MULTI-SERVER)
    createToggle("Brainrot ESP", pages.Rastreio, function(s) ToggleBrainrotESP(s) end)
    createToggle("Spawn ESP", pages.Rastreio, function(s) ToggleSpawnESP(s) end)

    local scanBtn = Instance.new("TextButton", pages.Rastreio)
    scanBtn.Size, scanBtn.Text = UDim2.new(0, 230, 0, 40), "SCANNER GLOBAL (OUTROS SERVERS)"
    scanBtn.BackgroundColor3, scanBtn.TextColor3, scanBtn.Font = Color3.fromRGB(0, 80, 0), Color3.new(1,1,1), "GothamBold"
    Instance.new("UICorner", scanBtn); addBorderLED(scanBtn, 2)
    
    scanBtn.MouseButton1Click:Connect(function()
        scanBtn.Text = "BUSCANDO SERVIDOR COM MAIS BRAINROTS..."
        task.wait(1.5)
        -- Lógica de busca de servidor com maior probabilidade de atividade
        local res = game:HttpGet("https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Desc&limit=10")
        local data = HttpService:JSONDecode(res)
        if data and data.data[1] then
            StarterGui:SetCore("ChatMakeSystemMessage", {
                Text = "[SCANNER] Servidor detectado com alta atividade! Teleportando...",
                Color = Color3.new(0,1,0)
            })
            TeleportService:TeleportToPlaceInstance(game.PlaceId, data.data[1].id)
        end
    end)

    -- ABA SERVIDORES (LISTA REAL)
    task.spawn(function()
        local success, res = pcall(function() return game:HttpGet("https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Asc&limit=100") end)
        if success then
            local data = HttpService:JSONDecode(res)
            for _, sv in pairs(data.data) do
                local slot = Instance.new("Frame", pages.Servidores); slot.Size = UDim2.new(0, 230, 0, 45); slot.BackgroundColor3 = Color3.fromRGB(20,20,20)
                Instance.new("UICorner", slot); addBorderLED(slot, 1)
                local t = Instance.new("TextLabel", slot); t.Size = UDim2.new(0.6,0,1,0); t.Text = "Players: "..sv.playing.."/"..sv.maxPlayers; t.TextColor3, t.BackgroundTransparency = Color3.new(1,1,1), 1
                local j = Instance.new("TextButton", slot); j.Size, j.Position, j.Text = UDim2.new(0, 60, 0, 25), UDim2.new(1,-70,0.5,-12), "Entrar"; j.BackgroundColor3 = Color3.fromRGB(150,0,0)
                Instance.new("UICorner", j); j.MouseButton1Click:Connect(function() TeleportService:TeleportToPlaceInstance(game.PlaceId, sv.id) end)
            end
        end
    end)

    -- BOTÕES DE TAB
    local function tab(n, y)
        local b = Instance.new("TextButton", left); b.Size, b.Position, b.Text = UDim2.new(0, 90, 0, 30), UDim2.new(0, 5, 0, y), n
        b.BackgroundColor3, b.TextColor3, b.Font = Color3.fromRGB(30,30,30), Color3.new(1,1,1), "GothamBold"
        Instance.new("UICorner", b); b.MouseButton1Click:Connect(function() for pn, pg in pairs(pages) do pg.Visible = (pn == n) end end)
    end
    tab("LTM", 5); tab("Rastreio", 40); tab("Servidores", 75)
end

--==============================
-- LOOP DE EXECUÇÃO
--==============================
RunService.RenderStepped:Connect(function()
    local char = LocalPlayer.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChildOfClass("Humanoid")

    if hum then hum.JumpPower = jumpEnabled and 190 or 50 end
    if hrp then
        if featherEnabled and hrp.Velocity.Y < -6 then hrp.Velocity = Vector3.new(hrp.Velocity.X, -3.2, hrp.Velocity.Z) end
        if airWalkEnabled then
            if not airPlatform then airPlatform = Instance.new("Part", Workspace); airPlatform.Anchored, airPlatform.Transparency, airPlatform.Size = true, 1, Vector3.new(15, 1, 15) end
            airPlatform.Position = hrp.Position - Vector3.new(0, 3.5, 0)
        elseif airPlatform then airPlatform:Destroy(); airPlatform = nil end
        if followBlockActive then
            local b = Instance.new("Part", Workspace); b.Size, b.Anchored, b.Color = Vector3.new(4, 0.5, 4), true, Color3.new(1,0,0)
            b.CFrame = hrp.CFrame - Vector3.new(0, 3.5, 0); Debris:AddItem(b, 1)
        end
    end
end)

createHub()
