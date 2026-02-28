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
local airPlatform = nil
local airHighlight = nil

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
-- FUNÇÃO CRIAR BLOCO (ESTILO DRASŦAK)
--==============================
local function spawnBlock(pos)
	local block = Instance.new("Part")
	block.Size = Vector3.new(4,0.5,4)
	block.Anchored = true
	block.CanCollide = true
	block.Material = Enum.Material.Neon
	block.Color = Color3.fromRGB(255, 0, 0) -- VERMELHO
	block.CFrame = CFrame.new(pos)
	block.Parent = Workspace
	Debris:AddItem(block, 1.5)
	return block
end

--==============================
-- HUB PRINCIPAL
--==============================
local function createHub()
    if LocalPlayer.PlayerGui:FindFirstChild("GTGHub") then LocalPlayer.PlayerGui.GTGHub:Destroy() end
    local gui = Instance.new("ScreenGui", LocalPlayer.PlayerGui); gui.Name = "GTGHub"; gui.ResetOnSpawn = false

    local openBtn = Instance.new("TextButton", gui)
    openBtn.Size, openBtn.Position = UDim2.new(0, 60, 0, 30), UDim2.new(0, 10, 0.1, 0)
    openBtn.Text, openBtn.Visible = "GTG", false
    openBtn.BackgroundColor3, openBtn.TextColor3, openBtn.Font = Color3.new(0,0,0), Color3.new(1,0,0), "GothamBold"
    Instance.new("UICorner", openBtn); addBorderLED(openBtn, 2)

    local main = Instance.new("Frame", gui)
    main.Size, main.Position = UDim2.new(0, 380, 0, 250), UDim2.new(0.5, -190, 0.5, -125)
    main.BackgroundColor3 = Color3.fromRGB(10,10,10); Instance.new("UICorner", main)
    addBorderLED(main, 3)

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

    local pages = { LTM = Instance.new("ScrollingFrame", content), Rastreio = Instance.new("ScrollingFrame", content), Servidores = Instance.new("ScrollingFrame", content) }

    for name, page in pairs(pages) do
        page.Size, page.BackgroundTransparency, page.Visible = UDim2.new(1, 0, 1, 0), 1, (name == "LTM")
        page.ScrollBarThickness, page.AutomaticCanvasSize = 0, "Y"
        Instance.new("UIListLayout", page).Padding = UDim.new(0, 8)
    end

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

    -- ABA LTM
    createToggle("Jump 3.8X", pages.LTM, function(s) jumpEnabled = s end)
    createToggle("Cair Leve", pages.LTM, function(s) featherEnabled = s end)
    createToggle("Andar no Ar", pages.LTM, function(s) 
        airWalkEnabled = s 
        if not s then
            if airPlatform then airPlatform:Destroy() airPlatform=nil end
            if airHighlight then airHighlight:Destroy() airHighlight=nil end
        end
    end)
    createToggle("Bloco Seguidor", pages.LTM, function(s) followBlockActive = s end)

    -- ABA RASTREIO (SERVIDOR RARO)
    local huntBtn = Instance.new("TextButton", pages.Rastreio)
    huntBtn.Size, huntBtn.Text = UDim2.new(0, 230, 0, 45), "BUSCAR SERVIDOR RARO"
    huntBtn.BackgroundColor3, huntBtn.TextColor3, huntBtn.Font = Color3.fromRGB(80, 0, 0), Color3.new(1,1,1), "GothamBold"
    Instance.new("UICorner", huntBtn); addBorderLED(huntBtn, 2)
    
    huntBtn.MouseButton1Click:Connect(function()
        huntBtn.Text = "RASTREANDO..."
        task.wait(3)
        local res = game:HttpGet("https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Desc&limit=100")
        local data = HttpService:JSONDecode(res)
        if data and data.data then
            local target = data.data[math.random(1, #data.data)]
            TeleportService:TeleportToPlaceInstance(game.PlaceId, target.id)
        end
    end)

    -- ABA SERVIDORES (FILTRO 1-4 PLAYERS)
    local refreshBtn = Instance.new("TextButton", pages.Servidores)
    refreshBtn.Size, refreshBtn.Text = UDim2.new(0, 230, 0, 30), "ATUALIZAR LISTA (1-4 PLAYERS)"
    refreshBtn.BackgroundColor3, refreshBtn.TextColor3 = Color3.fromRGB(60, 0, 0), Color3.new(1,1,1)
    Instance.new("UICorner", refreshBtn)

    local function loadServers()
        for _, v in pairs(pages.Servidores:GetChildren()) do if v:IsA("Frame") then v:Destroy() end end
        local success, res = pcall(function() return game:HttpGet("https://games.roblox.com/v1/games/"..game.PlaceId.."/servers/Public?sortOrder=Asc&limit=100") end)
        if success then
            local data = HttpService:JSONDecode(res)
            for _, sv in pairs(data.data) do
                if sv.playing >= 1 and sv.playing <= 4 then
                    local slot = Instance.new("Frame", pages.Servidores); slot.Size = UDim2.new(0, 230, 0, 40); slot.BackgroundColor3 = Color3.fromRGB(25,25,25)
                    Instance.new("UICorner", slot); addBorderLED(slot, 1)
                    local txt = Instance.new("TextLabel", slot); txt.Size = UDim2.new(0.6,0,1,0); txt.Text = "Players: "..sv.playing.."/12"; txt.TextColor3, txt.BackgroundTransparency = Color3.new(1,1,1), 1
                    local btn = Instance.new("TextButton", slot); btn.Size, btn.Position, btn.Text = UDim2.new(0, 70, 0, 25), UDim2.new(1,-80,0.5,-12), "ENTRAR"
                    btn.BackgroundColor3, btn.TextColor3 = Color3.fromRGB(120,0,0), Color3.new(1,1,1); Instance.new("UICorner", btn)
                    btn.MouseButton1Click:Connect(function() TeleportService:TeleportToPlaceInstance(game.PlaceId, sv.id) end)
                end
            end
        end
    end
    refreshBtn.MouseButton1Click:Connect(loadServers)
    loadServers()

    local function tab(n, y)
        local b = Instance.new("TextButton", left); b.Size, b.Position, b.Text = UDim2.new(0, 90, 0, 30), UDim2.new(0, 5, 0, y), n
        b.BackgroundColor3, b.TextColor3, b.Font = Color3.fromRGB(30,30,30), Color3.new(1,1,1), "GothamBold"
        Instance.new("UICorner", b); b.MouseButton1Click:Connect(function() for pn, pg in pairs(pages) do pg.Visible = (pn == n) end end)
    end
    tab("LTM", 5); tab("Rastreio", 40); tab("Servidores", 75)
end

--==============================
-- LOOP DE EXECUÇÃO (LÓGICA DRASŦAK)
--==============================
RunService.RenderStepped:Connect(function()
    local char = LocalPlayer.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChildOfClass("Humanoid")

    if hum then 
        hum.UseJumpPower = true
        hum.JumpPower = jumpEnabled and 190 or 50 
    end

    if hrp then
        -- CAIR LEVE
        if featherEnabled and hrp.Velocity.Y < -6 then hrp.Velocity = Vector3.new(hrp.Velocity.X, -3.2, hrp.Velocity.Z) end
        
        -- ANDAR NO AR (ESTILO DRASŦAK COM HIGHLIGHT)
        if airWalkEnabled then
            if not airPlatform then
                airPlatform = Instance.new("Part", Workspace)
                airPlatform.Anchored, airPlatform.CanCollide, airPlatform.Transparency = true, true, 1
                airPlatform.Size = Vector3.new(30, 1, 30)
                airPlatform.Position = hrp.Position - Vector3.new(0, 3, 0)
            end
            airPlatform.Position = Vector3.new(hrp.Position.X, airPlatform.Position.Y, hrp.Position.Z)
            
            if not airHighlight then
                airHighlight = Instance.new("Highlight", char)
                airHighlight.FillTransparency, airHighlight.OutlineColor = 1, Color3.fromRGB(255, 0, 0)
            end
        end

        -- BLOCO SEGUIDOR (LÓGICA LERP + SPAWNBLOCK)
        if followBlockActive then
            local targetPos = hrp.Position - Vector3.new(0, 3, 0)
            if not airPlatform then
                spawnBlock(targetPos)
            else
                airPlatform.Position = airPlatform.Position:Lerp(targetPos, 0.1)
            end
        end
    end
end)

createHub()
