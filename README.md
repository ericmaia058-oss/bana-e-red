-- ====================================================================
--   BLOX FRUITS - FRUIT SNIPER v9 (FULL)
--   - Fruit ESP (nome + distância)
--   - Auto Collect
--   - Auto Marine (funcional)
--   - Auto Server Hop (v2 com delay 4s)
--   - Auto Re-Execute (detector automático)
--   - Leve (sem Heartbeat)
-- ====================================================================

local Players = game:GetService("Players")
local TeleportService = game:GetService("TeleportService")
local HttpService = game:GetService("HttpService")
local StarterGui = game:GetService("StarterGui")
local CoreGui = game:GetService("CoreGui")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local VirtualInputManager = game:GetService("VirtualInputManager")
local LP = Players.LocalPlayer
local Workspace = game:GetService("Workspace")

-- ================== CONFIG ==================
local Config = {
    Enabled = true,
    AutoServerHop = true,
    HopDelay = 8,
    InitialScanDelay = 4,
    AutoMarine = true,
    ScanInterval = 1,
    ESPEnabled = true,
    ESPMaxDistance = 3000,
    AutoReExecute = true,
    ScriptUrl = "",  -- ⚠️ COLOQUE A URL RAW AQUI
}

-- ================== AUTO RE-EXECUTE (v2) ==================
local function getQueueFunc()
    if type(queue_on_teleport) == "function" then return queue_on_teleport end
    if syn and type(syn.queue_on_teleport) == "function" then return syn.queue_on_teleport end
    if fluxus and type(fluxus.queue_on_teleport) == "function" then return fluxus.queue_on_teleport end
    if krnl and type(krnl.queue_on_teleport) == "function" then return krnl.queue_on_teleport end
    return nil
end

local queueFunc = getQueueFunc()

local reExecCode = [[
    repeat task.wait(0.5) until game:IsLoaded()
    task.wait(3)
    if getgenv and getgenv().FRUIT_SNIPER_URL and getgenv().FRUIT_SNIPER_URL ~= "" then
        loadstring(game:HttpGet(getgenv().FRUIT_SNIPER_URL))()
    end
]]

if Config.AutoReExecute and Config.ScriptUrl ~= "" then
    getgenv().FRUIT_SNIPER_URL = Config.ScriptUrl
    if queueFunc then
        pcall(function() queueFunc(reExecCode) end)
        print("[Auto Re-Execute] Ativado via queue_on_teleport")
    else
        warn("[Auto Re-Execute] Executor sem queue_on_teleport. Use autoexec!")
    end
end

-- ================== BLACKLIST ==================
local BlacklistFruits = {
    ["Rocket"] = true, ["Spin"] = true, ["Chop"] = true, ["Spring"] = true,
    ["Bomb"] = true, ["Smoke"] = true, ["Spike"] = true, ["Flame"] = true,
    ["Falcon"] = true, ["Eagle"] = true, ["Ice"] = true, ["Sand"] = true,
    ["Dark"] = true, ["Diamond"] = true, ["Light"] = true, ["Rubber"] = true,
    ["Barrier"] = true, ["Ghost"] = true, ["Magma"] = true, ["Door"] = true,
    ["Buddha"] = true, ["Love"] = true, ["Spider"] = true, ["Sound"] = true,
    ["Phoenix"] = true, ["Portal"] = true, ["Rumble"] = true, ["Pain"] = true,
    ["Blizzard"] = true, ["Gravity"] = true, ["Mammoth"] = true, ["T-Rex"] = true,
    ["Dough"] = true, ["Shadow"] = true, ["Venom"] = true, ["Control"] = true,
    ["Spirit"] = true, ["Dragon"] = true, ["Leopard"] = true, ["Tiger"] = true,
    ["Kitsune"] = true, ["Yeti"] = true,
}

-- ================== AUTO MARINE ==================
local marineStatus = "Aguardando..."
local marineAttempts = 0
local marineEnabled = true

local function isMarine()
    if LP.Team and (LP.Team.Name == "Marines" or LP.Team.Name == "Marine") then return true end
    local stats = LP:FindFirstChild("leaderstats")
    if stats then
        for _, stat in ipairs(stats:GetChildren()) do
            local n = stat.Name:lower()
            if n:find("team") or n:find("faction") or n:find("side") then
                local v = tostring(stat.Value):lower()
                if v == "marines" or v == "marine" or v == "marinheiros" then return true end
            end
        end
    end
    return false
end

local function findMarineButton()
    local PG = LP:FindFirstChild("PlayerGui")
    if not PG then return nil end

    local main = PG:FindFirstChild("Main (minimal)")
    if main then
        local ct = main:FindFirstChild("ChooseTeam")
        if ct then
            local c = ct:FindFirstChild("Container")
            if c then
                local m = c:FindFirstChild("Marines")
                if m then
                    local f = m:FindFirstChild("Frame")
                    if f then
                        local b = f:FindFirstChild("TextButton")
                        if b and b:IsA("TextButton") and b.Visible then return b end
                    end
                end
            end
        end
    end

    local ts = PG:FindFirstChild("TeamSelection")
    if ts then
        local r = ts:FindFirstChild("ROOT")
        if r then
            local f = r:FindFirstChild("Frame")
            if f then
                local c = f:FindFirstChild("content")
                if c then
                    local mf = c:FindFirstChild("marineFrame")
                    if mf then
                        local m = mf:FindFirstChild("marine")
                        if m and m:IsA("TextButton") and m.Visible then return m end
                    end
                end
            end
        end
    end
    return nil
end

local function clickMarine(btn)
    if not btn then return false end
    pcall(function() btn.MouseButton1Click:Fire() end)
    pcall(function()
        local pos = btn.AbsolutePosition
        local size = btn.AbsoluteSize
        VirtualInputManager:SendMouseButtonEvent(pos.X + size.X/2, pos.Y + size.Y/2, 0, true, game, 1)
        task.wait(0.05)
        VirtualInputManager:SendMouseButtonEvent(pos.X + size.X/2, pos.Y + size.Y/2, 0, false, game, 1)
    end)
    return true
end

-- ================== FRUTAS ==================
local function isRealFruit(obj)
    if not obj or not obj.Parent then return false end
    if not obj:IsA("Tool") and not obj:IsA("Model") then return false end
    local n = obj.Name
    if not (n:find("Fruit") or n:find("fruit")) then return false end
    local cleanName = n:gsub("Fruit", ""):gsub("fruit", ""):gsub("%s+", ""):lower()
    for blName, _ in pairs(BlacklistFruits) do
        if cleanName == blName:lower() then return false end
    end
    local part = nil
    if obj:IsA("Tool") then
        part = obj:FindFirstChild("Handle")
    elseif obj:IsA("Model") then
        part = obj.PrimaryPart or obj:FindFirstChildWhichIsA("BasePart")
    end
    if not part or not part:IsA("BasePart") then return false end
    if part.Transparency >= 1 then return false end
    return true
end

local function getFruitPosition(fruit)
    if not fruit or not fruit.Parent then return nil end
    if fruit:IsA("Tool") then
        local h = fruit:FindFirstChild("Handle")
        return h and h.Position or nil
    end
    if fruit:IsA("Model") then
        if fruit.PrimaryPart then return fruit.PrimaryPart.Position end
        local p = fruit:FindFirstChildWhichIsA("BasePart")
        return p and p.Position or nil
    end
    return nil
end

local fruitCache = {}
local lastFruitScan = 0

local function refreshFruits()
    if tick() - lastFruitScan < Config.ScanInterval then return end
    lastFruitScan = tick()

    local hrp = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then fruitCache = {}; return end
    local myPos = hrp.Position

    local fruits = {}
    local folder = Workspace:FindFirstChild("Fruits")

    if folder then
        for _, obj in ipairs(folder:GetChildren()) do
            if isRealFruit(obj) then
                local pos = getFruitPosition(obj)
                if pos then
                    table.insert(fruits, {obj = obj, pos = pos, dist = (pos - myPos).Magnitude, name = obj.Name})
                end
            end
        end
    else
        for _, obj in ipairs(Workspace:GetChildren()) do
            if isRealFruit(obj) then
                local pos = getFruitPosition(obj)
                if pos then
                    table.insert(fruits, {obj = obj, pos = pos, dist = (pos - myPos).Magnitude, name = obj.Name})
                end
            end
        end
    end

    table.sort(fruits, function(a, b) return a.dist < b.dist end)
    fruitCache = fruits
end

-- ================== FRUIT ESP ==================
local espFolder = nil
local espData = {}

local function createESP(fruit, pos, dist)
    if not fruit or not fruit.Parent then return end

    if not espFolder or not espFolder.Parent then
        espFolder = Instance.new("Folder")
        espFolder.Name = "FruitESP"
        espFolder.Parent = (pcall(function() return CoreGui end) and CoreGui) or LP:WaitForChild("PlayerGui")
    end

    if espData[fruit] then
        if espData[fruit].distLabel then
            espData[fruit].distLabel.Text = string.format("%d studs", math.floor(dist))
        end
        return
    end

    local hl = Instance.new("Highlight")
    hl.FillColor = Color3.fromRGB(255, 100, 100)
    hl.FillTransparency = 0.5
    hl.OutlineColor = Color3.fromRGB(255, 255, 0)
    hl.OutlineTransparency = 0
    hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
    hl.Adornee = fruit
    hl.Parent = espFolder

    local bill = Instance.new("BillboardGui")
    bill.Size = UDim2.new(0, 150, 0, 40)
    bill.StudsOffset = Vector3.new(0, 3, 0)
    bill.AlwaysOnTop = true
    bill.MaxDistance = Config.ESPMaxDistance
    bill.Adornee = fruit
    bill.Parent = espFolder

    local nameLbl = Instance.new("TextLabel")
    nameLbl.Size = UDim2.new(1, 0, 0.55, 0)
    nameLbl.BackgroundTransparency = 1
    nameLbl.Text = fruit.Name:gsub("Fruit", ""):gsub("fruit", ""):gsub("^%s+", ""):gsub("%s+$", "")
    nameLbl.TextColor3 = Color3.fromRGB(255, 220, 60)
    nameLbl.TextStrokeTransparency = 0
    nameLbl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
    nameLbl.Font = Enum.Font.GothamBold
    nameLbl.TextSize = 14
    nameLbl.Parent = bill

    local distLbl = Instance.new("TextLabel")
    distLbl.Size = UDim2.new(1, 0, 0.45, 0)
    distLbl.Position = UDim2.new(0, 0, 0.55, 0)
    distLbl.BackgroundTransparency = 1
    distLbl.Text = string.format("%d studs", math.floor(dist))
    distLbl.TextColor3 = Color3.fromRGB(255, 255, 255)
    distLbl.TextStrokeTransparency = 0
    distLbl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
    distLbl.Font = Enum.Font.Gotham
    distLbl.TextSize = 12
    distLbl.Parent = bill

    espData[fruit] = {highlight = hl, billboard = bill, distLabel = distLbl}
end

local function destroyESP(fruit)
    local data = espData[fruit]
    if not data then return end
    if data.highlight then data.highlight:Destroy() end
    if data.billboard then data.billboard:Destroy() end
    espData[fruit] = nil
end

local function clearAllESP()
    for fruit in pairs(espData) do destroyESP(fruit) end
    espData = {}
end

local function collectFruit(fruit)
    local hrp = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    local pos = getFruitPosition(fruit.obj)
    if not pos then return end
    hrp.CFrame = CFrame.new(pos + Vector3.new(0, 3, 0))
end

-- ================== SERVER HOP v2 ==================
local AllIDs = {}
local foundAnything = ""
local actualHour = os.date("!*t").hour

local function loadIDs()
    pcall(function()
        AllIDs = HttpService:JSONDecode(readfile("server-hop-temp.json"))
    end)
    if type(AllIDs) ~= "table" or #AllIDs == 0 then
        AllIDs = {actualHour}
        pcall(function()
            writefile("server-hop-temp.json", HttpService:JSONEncode(AllIDs))
        end)
    end
end

local function saveIDs()
    pcall(function()
        writefile("server-hop-temp.json", HttpService:JSONEncode(AllIDs))
    end)
end

local function TPReturner(placeId)
    local Site
    if foundAnything == "" then
        Site = HttpService:JSONDecode(game:HttpGet(
            'https://games.roblox.com/v1/games/' .. placeId .. '/servers/Public?sortOrder=Asc&limit=100'
        ))
    else
        Site = HttpService:JSONDecode(game:HttpGet(
            'https://games.roblox.com/v1/games/' .. placeId .. '/servers/Public?sortOrder=Asc&limit=100&cursor=' .. foundAnything
        ))
    end

    local ID = ""
    if Site.nextPageCursor and Site.nextPageCursor ~= "null" and Site.nextPageCursor ~= nil then
        foundAnything = Site.nextPageCursor
    end

    local num = 0
    for i, v in pairs(Site.data) do
        local Possible = true
        ID = tostring(v.id)
        if tonumber(v.maxPlayers) > tonumber(v.playing) then
            for _, Existing in pairs(AllIDs) do
                if num ~= 0 then
                    if ID == tostring(Existing) then Possible = false end
                else
                    if tonumber(actualHour) ~= tonumber(Existing) then
                        pcall(function()
                            delfile("server-hop-temp.json")
                            AllIDs = {}
                            table.insert(AllIDs, actualHour)
                        end)
                    end
                end
                num = num + 1
            end
            if Possible == true then
                table.insert(AllIDs, ID)
                task.wait()
                pcall(function()
                    saveIDs()
                    task.wait()
                    TeleportService:TeleportToPlaceInstance(placeId, ID, LP)
                end)
                task.wait(4)
            end
        end
    end
end

local function doServerHop()
    pcall(function()
        TPReturner(game.PlaceId)
        if foundAnything ~= "" then TPReturner(game.PlaceId) end
    end)
end

loadIDs()

-- ================== UI ==================
local function buildUI()
    local parentGui = (pcall(function() return CoreGui end) and CoreGui) or LP:WaitForChild("PlayerGui")

    for _, obj in ipairs(parentGui:GetChildren()) do
        if obj:IsA("ScreenGui") and obj.Name == "FruitSniperUI" then obj:Destroy() end
    end

    local gui = Instance.new("ScreenGui")
    gui.Name = "FruitSniperUI"
    gui.ResetOnSpawn = false
    gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
    gui.Parent = parentGui

    local frame = Instance.new("Frame")
    frame.Size = UDim2.fromOffset(330, 410)
    frame.Position = UDim2.new(0, 20, 0.3, 0)
    frame.BackgroundColor3 = Color3.fromRGB(18, 18, 22)
    frame.BorderSizePixel = 0
    frame.Active = true
    frame.Draggable = true
    frame.Parent = gui
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0, 8)
    local st = Instance.new("UIStroke", frame)
    st.Color = Color3.fromRGB(60, 60, 70); st.Thickness = 1

    local header = Instance.new("Frame", frame)
    header.Size = UDim2.new(1, 0, 0, 34)
    header.BackgroundColor3 = Color3.fromRGB(28, 28, 34)
    header.BorderSizePixel = 0
    Instance.new("UICorner", header).CornerRadius = UDim.new(0, 8)

    local title = Instance.new("TextLabel", header)
    title.Size = UDim2.new(1, -40, 1, 0); title.Position = UDim2.fromOffset(12, 0)
    title.BackgroundTransparency = 1
    title.Text = "🍎 Fruit Sniper v9"
    title.TextColor3 = Color3.fromRGB(240, 240, 245)
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Font = Enum.Font.GothamBold; title.TextSize = 14

    local closeBtn = Instance.new("TextButton", header)
    closeBtn.Size = UDim2.fromOffset(24, 24); closeBtn.Position = UDim2.new(1, -30, 0.5, -12)
    closeBtn.BackgroundColor3 = Color3.fromRGB(50, 50, 58)
    closeBtn.Text = "X"; closeBtn.TextColor3 = Color3.fromRGB(240, 240, 245)
    closeBtn.Font = Enum.Font.GothamBold; closeBtn.TextSize = 12; closeBtn.BorderSizePixel = 0
    Instance.new("UICorner", closeBtn).CornerRadius = UDim.new(0, 6)
    closeBtn.MouseButton1Click:Connect(function() gui.Enabled = false end)

    local status = Instance.new("TextLabel", frame)
    status.Size = UDim2.new(1, -24, 0, 36); status.Position = UDim2.fromOffset(12, 44)
    status.BackgroundTransparency = 1
    status.Text = "Iniciando..."
    status.TextColor3 = Color3.fromRGB(180, 180, 190)
    status.TextXAlignment = Enum.TextXAlignment.Left
    status.TextYAlignment = Enum.TextYAlignment.Top
    status.TextWrapped = true
    status.Font = Enum.Font.Gotham; status.TextSize = 11

    local fruitsLbl = Instance.new("TextLabel", frame)
    fruitsLbl.Size = UDim2.new(1, -24, 0, 18); fruitsLbl.Position = UDim2.fromOffset(12, 82)
    fruitsLbl.BackgroundTransparency = 1
    fruitsLbl.Text = "Frutas: 0"
    fruitsLbl.TextColor3 = Color3.fromRGB(255, 200, 80)
    fruitsLbl.TextXAlignment = Enum.TextXAlignment.Left
    fruitsLbl.Font = Enum.Font.Gotham; fruitsLbl.TextSize = 11

    local scanLbl = Instance.new("TextLabel", frame)
    scanLbl.Size = UDim2.new(1, -24, 0, 18); scanLbl.Position = UDim2.fromOffset(12, 100)
    scanLbl.BackgroundTransparency = 1
    scanLbl.Text = "Scan: -"
    scanLbl.TextColor3 = Color3.fromRGB(200, 180, 255)
    scanLbl.TextXAlignment = Enum.TextXAlignment.Left
    scanLbl.Font = Enum.Font.Gotham; scanLbl.TextSize = 11

    local hopLbl = Instance.new("TextLabel", frame)
    hopLbl.Size = UDim2.new(1, -24, 0, 18); hopLbl.Position = UDim2.fromOffset(12, 118)
    hopLbl.BackgroundTransparency = 1
    hopLbl.Text = "Hop em: -"
    hopLbl.TextColor3 = Color3.fromRGB(140, 200, 255)
    hopLbl.TextXAlignment = Enum.TextXAlignment.Left
    hopLbl.Font = Enum.Font.Gotham; hopLbl.TextSize = 11

    local marineLbl = Instance.new("TextLabel", frame)
    marineLbl.Size = UDim2.new(1, -24, 0, 18); marineLbl.Position = UDim2.fromOffset(12, 136)
    marineLbl.BackgroundTransparency = 1
    marineLbl.Text = "Marine: " .. marineStatus
    marineLbl.TextColor3 = Color3.fromRGB(140, 220, 140)
    marineLbl.TextXAlignment = Enum.TextXAlignment.Left
    marineLbl.Font = Enum.Font.Gotham; marineLbl.TextSize = 11

    local function makeToggleBtn(text, yPos, getter, setter)
        local btn = Instance.new("TextButton", frame)
        btn.Size = UDim2.new(1, -24, 0, 26); btn.Position = UDim2.fromOffset(12, yPos)
        btn.BackgroundColor3 = getter() and Color3.fromRGB(60, 180, 80) or Color3.fromRGB(120, 60, 60)
        btn.Text = text .. ": " .. (getter() and "ON" or "OFF")
        btn.TextColor3 = Color3.fromRGB(240, 240, 245)
        btn.Font = Enum.Font.GothamBold; btn.TextSize = 11; btn.BorderSizePixel = 0
        Instance.new("UICorner", btn).CornerRadius = UDim.new(0, 6)
        btn.MouseButton1Click:Connect(function()
            setter(not getter())
            btn.BackgroundColor3 = getter() and Color3.fromRGB(60, 180, 80) or Color3.fromRGB(120, 60, 60)
            btn.Text = text .. ": " .. (getter() and "ON" or "OFF")
        end)
        return btn
    end

    makeToggleBtn("Sniper", 162, function() return Config.Enabled end,
        function(v) Config.Enabled = v end)
    makeToggleBtn("Fruit ESP", 192, function() return Config.ESPEnabled end,
        function(v)
            Config.ESPEnabled = v
            if not v then clearAllESP() end
        end)
    makeToggleBtn("Auto Server Hop", 222, function() return Config.AutoServerHop end,
        function(v) Config.AutoServerHop = v end)
    makeToggleBtn("Auto Marine", 252, function() return Config.AutoMarine end,
        function(v)
            Config.AutoMarine = v
            marineEnabled = v
        end)
    makeToggleBtn("Auto Re-Execute", 282, function() return Config.AutoReExecute end,
        function(v) Config.AutoReExecute = v end)

    local hopBtn = Instance.new("TextButton", frame)
    hopBtn.Size = UDim2.new(1, -24, 0, 26); hopBtn.Position = UDim2.fromOffset(12, 312)
    hopBtn.BackgroundColor3 = Color3.fromRGB(80, 120, 220)
    hopBtn.Text = "TROCAR DE SERVER AGORA"
    hopBtn.TextColor3 = Color3.fromRGB(240, 240, 245)
    hopBtn.Font = Enum.Font.GothamBold; hopBtn.TextSize = 11; hopBtn.BorderSizePixel = 0
    Instance.new("UICorner", hopBtn).CornerRadius = UDim.new(0, 6)
    hopBtn.MouseButton1Click:Connect(function() task.spawn(doServerHop) end)

    local debugBtn = Instance.new("TextButton", frame)
    debugBtn.Size = UDim2.new(1, -24, 0, 22); debugBtn.Position = UDim2.fromOffset(12, 342)
    debugBtn.BackgroundColor3 = Color3.fromRGB(90, 90, 150)
    debugBtn.Text = "🔍 TESTAR RE-EXECUTE"
    debugBtn.TextColor3 = Color3.fromRGB(240, 240, 245)
    debugBtn.Font = Enum.Font.GothamBold; debugBtn.TextSize = 10; debugBtn.BorderSizePixel = 0
    Instance.new("UICorner", debugBtn).CornerRadius = UDim.new(0, 6)
    debugBtn.MouseButton1Click:Connect(function()
        print("========== DEBUG RE-EXECUTE ==========")
        print("ScriptUrl = '" .. Config.ScriptUrl .. "'")
        print("queue_on_teleport = " .. tostring(queue_on_teleport))
        print("syn.queue_on_teleport = " .. tostring(syn and syn.queue_on_teleport))
        if getexecutorname then
            pcall(function() print("Executor: " .. getexecutorname()) end)
        end
        if Config.ScriptUrl ~= "" then
            local ok, err = pcall(function()
                local test = game:HttpGet(Config.ScriptUrl)
                print("URL OK! Tamanho: " .. #test .. " chars")
            end)
            if not ok then print("URL FALHOU: " .. tostring(err)) end
        else
            print("URL vazia! Preencha Config.ScriptUrl")
        end
        print("=======================================")
    end)

    local clearBtn = Instance.new("TextButton", frame)
    clearBtn.Size = UDim2.new(1, -24, 0, 22); clearBtn.Position = UDim2.fromOffset(12, 372)
    clearBtn.BackgroundColor3 = Color3.fromRGB(120, 60, 60)
    clearBtn.Text = "Limpar ESP"
    clearBtn.TextColor3 = Color3.fromRGB(240, 240, 245)
    clearBtn.Font = Enum.Font.GothamBold; clearBtn.TextSize = 10; clearBtn.BorderSizePixel = 0
    Instance.new("UICorner", clearBtn).CornerRadius = UDim.new(0, 6)
    clearBtn.MouseButton1Click:Connect(clearAllESP)

    task.spawn(function()
        while gui.Parent do
            fruitsLbl.Text = "Frutas: " .. #fruitCache
            marineLbl.Text = "Marine: " .. marineStatus
            task.wait(1)
        end
    end)

    _G.__FruitSniperUI = {
        setStatus = function(t) status.Text = t end,
        setScanTimer = function(t) scanLbl.Text = t end,
        setHopTimer = function(t) hopLbl.Text = t end,
    }
end

buildUI()

-- ================== ESP UPDATE ==================
task.spawn(function()
    while true do
        task.wait(0.3)
        if not Config.ESPEnabled then task.wait(1); continue end

        for fruit in pairs(espData) do
            if not fruit or not fruit.Parent then destroyESP(fruit) end
        end

        local hrp = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
        if not hrp then continue end
        local myPos = hrp.Position

        for _, fruit in ipairs(fruitCache) do
            if fruit.obj and fruit.obj.Parent then
                local dist = (fruit.pos - myPos).Magnitude
                if dist <= Config.ESPMaxDistance then
                    createESP(fruit.obj, fruit.pos, dist)
                end
            end
        end
    end
end)

-- ================== AUTO MARINE LOOP ==================
task.spawn(function()
    while true do
        task.wait(0.8)
        if not marineEnabled then continue end

        if isMarine() then
            marineStatus = "Já é Marine ✓"
        else
            local btn = findMarineButton()
            if btn then
                marineAttempts = marineAttempts + 1
                marineStatus = string.format("Clicando #%d", marineAttempts)
                clickMarine(btn)
            else
                marineStatus = "Aguardando Pick A Side"
            end
        end
    end
end)

-- ================== LOOP PRINCIPAL ==================
task.spawn(function()
    local lastHop = 0
    local fruitFound = false

    if _G.__FruitSniperUI then _G.__FruitSniperUI.setStatus("Aguardando jogo...") end

    if not LP.Character then LP.CharacterAdded:Wait() end
    LP.Character:WaitForChild("HumanoidRootPart", 10)

    local startTime = tick()
    while tick() - startTime < Config.InitialScanDelay do
        local rem = Config.InitialScanDelay - (tick() - startTime)
        if _G.__FruitSniperUI then
            _G.__FruitSniperUI.setScanTimer(string.format("Scan em: %.1fs", rem))
        end
        task.wait(0.1)
    end

    lastFruitScan = 0
    refreshFruits()

    if _G.__FruitSniperUI then
        _G.__FruitSniperUI.setScanTimer("Scan: ok")
        _G.__FruitSniperUI.setStatus("Scan: " .. #fruitCache .. " frutas")
    end

    lastHop = tick()

    while true do
        task.wait(0.5)
        if not Config.Enabled then continue end
        if not LP.Character then continue end

        refreshFruits()

        if #fruitCache > 0 then
            fruitFound = true
            local closest = fruitCache[1]
            if _G.__FruitSniperUI then
                _G.__FruitSniperUI.setStatus("Pegando: " .. closest.name)
            end
            pcall(collectFruit, closest)
        else
            if _G.__FruitSniperUI then _G.__FruitSniperUI.setStatus("Sem fruta") end

            if Config.AutoServerHop and not fruitFound then
                local wait = Config.HopDelay - (tick() - lastHop)
                if wait <= 0 then
                    lastHop = tick()
                    if _G.__FruitSniperUI then _G.__FruitSniperUI.setStatus("Hopando...") end
                    task.spawn(doServerHop)
                else
                    if _G.__FruitSniperUI then
                        _G.__FruitSniperUI.setHopTimer("Hop em: " .. math.floor(wait) .. "s")
                    end
                end
            end
        end
    end
end)

-- ================== RE-EXECUTE POR MORTE ==================
if Config.AutoReExecute and Config.ScriptUrl ~= "" then
    LP.CharacterAdded:Connect(function()
        task.wait(3)
        pcall(function()
            loadstring(game:HttpGet(Config.ScriptUrl))()
        end)
    end)
end

print("[Fruit Sniper v9] Carregado!")
print("Auto Re-Execute: " .. tostring(Config.AutoReExecute))
print("queue_on_teleport disponível: " .. tostring(queueFunc ~= nil))
print("ScriptUrl: " .. (Config.ScriptUrl == "" and "NÃO CONFIGURADA" or Config.ScriptUrl))
