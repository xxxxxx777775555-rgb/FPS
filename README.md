-- RoBoost | Ultra FPS & Lag Reducer
-- Target: Roblox (Luau) | Client-side LocalScript / Executor
-- Removes shadows, effects, reduces render quality to max performance

local RunService     = game:GetService("RunService")
local Players        = game:GetService("Players")
local Lighting       = game:GetService("Lighting")
local StarterGui     = game:GetService("StarterGui")
local TweenService   = game:GetService("TweenService")

local lp  = Players.LocalPlayer
local lpg = lp.PlayerGui

-- ============================================================
-- SETTINGS (แก้ได้)
-- ============================================================
local CFG = {
    removeShadows      = true,
    removeEffects      = true,   -- bloom, blur, colorgrade ฯลฯ
    removeParticles    = true,
    removeDecals       = true,
    removeTextures     = false,  -- true = สุดโต่ง อาจทำให้แผนที่ดูแปลก
    removeWater        = false,
    disableSky         = true,
    lowerRenderFidelity= true,
    lowerMeshDetail    = true,
    removeSound        = false,  -- ปิดเสียงทั้งหมด (เปิดถ้าอยากได้ fps สุดๆ)
    unlockFPS          = true,
    targetFPS          = 144,
    pruneTick          = 10,     -- วินาที scan ลบ particle/effect ใหม่
}

-- ============================================================
-- FPS UNLOCK
-- ============================================================
if CFG.unlockFPS then
    local ok, svc = pcall(function() return game:GetService("RenderSettings") end)
    if ok and svc then
        pcall(function() svc.FrameRateManager = Enum.FrameRateManager.Unlocked end)
    end
    -- executor method fallback
    pcall(function() setfpscap(CFG.targetFPS) end)
    pcall(function() syn.request and setfpscap(CFG.targetFPS) end)
end

-- ============================================================
-- LIGHTING NUKE
-- ============================================================
local function nukeLighting()
    local L = Lighting

    -- ปิด shadow / global illum
    pcall(function() L.GlobalShadows      = false end)
    pcall(function() L.ShadowSoftness     = 0 end)
    pcall(function() L.EnvironmentDiffuseScale = 0 end)
    pcall(function() L.EnvironmentSpecularScale = 0 end)

    -- ลด ambient ให้ flat แต่ยังมองเห็น
    pcall(function() L.Ambient            = Color3.fromRGB(178,178,178) end)
    pcall(function() L.OutdoorAmbient     = Color3.fromRGB(178,178,178) end)
    pcall(function() L.Brightness         = 2 end)
    pcall(function() L.ClockTime          = 14 end)
    pcall(function() L.FogEnd             = 1e6 end)
    pcall(function() L.FogStart           = 1e6 end)

    -- เทคโนโลยี lighting: ถอยลงมา Compatibility (เร็วสุด)
    pcall(function()
        L.Technology = Enum.LightingTechnology.Compatibility
    end)

    -- ลบ sky
    if CFG.disableSky then
        for _, obj in ipairs(L:GetChildren()) do
            if obj:IsA("Sky") then
                pcall(function() obj:Destroy() end)
            end
        end
    end

    -- ลบ post-processing effects
    if CFG.removeEffects then
        local effectTypes = {
            "BloomEffect","BlurEffect","ColorCorrectionEffect",
            "DepthOfFieldEffect","SunRaysEffect","SelectionEffect"
        }
        for _, obj in ipairs(L:GetDescendants()) do
            for _, t in ipairs(effectTypes) do
                if obj:IsA(t) then pcall(function() obj:Destroy() end) end
            end
        end
    end
end

nukeLighting()

-- ลบ effect ใหม่ที่ game อาจ spawn เข้ามา
if CFG.removeEffects then
    Lighting.DescendantAdded:Connect(function(obj)
        local effectTypes = {
            "BloomEffect","BlurEffect","ColorCorrectionEffect",
            "DepthOfFieldEffect","SunRaysEffect"
        }
        for _, t in ipairs(effectTypes) do
            if obj:IsA(t) then
                task.defer(function() pcall(function() obj:Destroy() end) end)
            end
        end
    end)
end

-- ============================================================
-- RENDER SETTINGS
-- ============================================================
local function applyRenderSettings()
    local RS = settings().Rendering

    pcall(function() RS.QualityLevel         = Enum.QualityLevel.Level01 end)
    pcall(function() RS.EditQualityLevel     = Enum.QualityLevel.Level01 end)

    -- shadow distance = 0
    pcall(function()
        RS.ShadowMapBias = 0
    end)
end

pcall(applyRenderSettings)

-- ============================================================
-- WORKSPACE SETTINGS
-- ============================================================
pcall(function()
    workspace.StreamingEnabled  = false   -- ปิด streaming throttle
end)
pcall(function()
    workspace.StreamingIntegrityMode = Enum.StreamingIntegrityMode.Disabled
end)

-- ============================================================
-- MESH / RENDER FIDELITY DOWNGRADE
-- ============================================================
local function downgradePart(obj)
    if obj:IsA("MeshPart") or obj:IsA("SpecialMesh") or obj:IsA("UnionOperation") then
        pcall(function() obj.RenderFidelity = Enum.RenderFidelity.Automatic end)
        if CFG.lowerMeshDetail then
            pcall(function() obj.RenderFidelity = Enum.RenderFidelity.Performance end)
        end
        -- ปิด cast shadow
        pcall(function() obj.CastShadow = false end)
    end
    if obj:IsA("BasePart") then
        pcall(function() obj.CastShadow = false end)
    end
end

-- ============================================================
-- PARTICLE / DECAL / TEXTURE REMOVER
-- ============================================================
local REMOVE_TYPES = {}
if CFG.removeParticles then
    table.insert(REMOVE_TYPES, "ParticleEmitter")
    table.insert(REMOVE_TYPES, "Trail")
    table.insert(REMOVE_TYPES, "Beam")
    table.insert(REMOVE_TYPES, "Fire")
    table.insert(REMOVE_TYPES, "Smoke")
    table.insert(REMOVE_TYPES, "Sparkles")
end
if CFG.removeDecals then
    table.insert(REMOVE_TYPES, "Decal")
end
if CFG.removeTextures then
    table.insert(REMOVE_TYPES, "Texture")
end

local function shouldRemove(obj)
    for _, t in ipairs(REMOVE_TYPES) do
        if obj:IsA(t) then return true end
    end
    return false
end

local function scanAndRemove(root)
    for _, obj in ipairs(root:GetDescendants()) do
        if shouldRemove(obj) then
            pcall(function() obj:Destroy() end)
        else
            downgradePart(obj)
        end
    end
end

-- Sound
local function nukeSound(root)
    if not CFG.removeSound then return end
    for _, obj in ipairs(root:GetDescendants()) do
        if obj:IsA("Sound") then
            pcall(function() obj.Volume = 0 end)
        end
    end
end

-- ============================================================
-- INITIAL FULL SCAN
-- ============================================================
local function fullScan()
    scanAndRemove(workspace)
    scanAndRemove(game:GetService("ReplicatedStorage"))
    nukeSound(workspace)
    nukeLighting()
end

task.spawn(fullScan)

-- ============================================================
-- PERIODIC SCAN (ลบ effect ที่ game spawn เพิ่มระหว่างเล่น)
-- ============================================================
workspace.DescendantAdded:Connect(function(obj)
    task.defer(function()
        if shouldRemove(obj) then
            pcall(function() obj:Destroy() end)
        else
            downgradePart(obj)
        end
        if CFG.removeSound and obj:IsA("Sound") then
            pcall(function() obj.Volume = 0 end)
        end
    end)
end)

-- rescan ทุก N วินาที สำหรับของที่หลุดรอด
task.spawn(function()
    while true do
        task.wait(CFG.pruneTick)
        task.spawn(fullScan)
    end
end)

-- ============================================================
-- CHARACTER SETUP (apply ทุกครั้งที่ respawn)
-- ============================================================
local function setupChar(char)
    if not char then return end
    for _, obj in ipairs(char:GetDescendants()) do
        downgradePart(obj)
        if shouldRemove(obj) then
            pcall(function() obj:Destroy() end)
        end
    end
    char.DescendantAdded:Connect(function(obj)
        task.defer(function()
            downgradePart(obj)
            if shouldRemove(obj) then
                pcall(function() obj:Destroy() end)
            end
        end)
    end)
end

lp.CharacterAdded:Connect(setupChar)
if lp.Character then setupChar(lp.Character) end

-- ============================================================
-- GUI
-- ============================================================
local function buildGUI()
    if lpg:FindFirstChild("RoBoostGUI") then return end
    local sg = Instance.new("ScreenGui")
    sg.Name = "RoBoostGUI"
    sg.ResetOnSpawn = false
    sg.Parent = lpg

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 180, 0, 60)
    frame.Position = UDim2.new(0, 8, 1, -70)
    frame.BackgroundColor3 = Color3.fromRGB(12,12,12)
    frame.BorderSizePixel = 0
    frame.Active = true
    frame.Draggable = true
    frame.Parent = sg
    Instance.new("UICorner", frame).CornerRadius = UDim.new(0,8)

    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(1,0,0,28)
    title.BackgroundTransparency = 1
    title.Text = "⚡ RoBoost — Active"
    title.TextColor3 = Color3.fromRGB(100,255,150)
    title.Font = Enum.Font.GothamBold
    title.TextSize = 12
    title.Parent = frame

    -- FPS counter
    local fpsLabel = Instance.new("TextLabel")
    fpsLabel.Size = UDim2.new(1,0,0,20)
    fpsLabel.Position = UDim2.new(0,0,0,28)
    fpsLabel.BackgroundTransparency = 1
    fpsLabel.Text = "FPS: ..."
    fpsLabel.TextColor3 = Color3.fromRGB(160,160,255)
    fpsLabel.Font = Enum.Font.Gotham
    fpsLabel.TextSize = 11
    fpsLabel.Parent = frame

    -- FPS tracker
    local fpsSamples = {}
    RunService.Heartbeat:Connect(function(dt)
        table.insert(fpsSamples, 1/dt)
        if #fpsSamples > 30 then table.remove(fpsSamples, 1) end
        local sum = 0
        for _,v in ipairs(fpsSamples) do sum=sum+v end
        local avg = math.floor(sum/#fpsSamples)
        local col = avg>=60 and Color3.fromRGB(100,255,150)
                    or avg>=30 and Color3.fromRGB(255,200,50)
                    or Color3.fromRGB(255,80,80)
        fpsLabel.Text = "FPS: "..avg
        fpsLabel.TextColor3 = col
    end)

    -- ping
    local pingLabel = Instance.new("TextLabel")
    pingLabel.Size = UDim2.new(1,0,0,16)
    pingLabel.Position = UDim2.new(0,0,0,44)
    pingLabel.BackgroundTransparency = 1
    pingLabel.Text = "Ping: ..."
    pingLabel.TextColor3 = Color3.fromRGB(160,160,160)
    pingLabel.Font = Enum.Font.Gotham
    pingLabel.TextSize = 10
    pingLabel.Parent = frame

    -- resize frame to fit all
    frame.Size = UDim2.new(0,180,0,65)

    task.spawn(function()
        while true do
            task.wait(2)
            local stats = game:GetService("Stats")
            local ping = pcall(function()
                pingLabel.Text = "Ping: "..math.floor(stats.Network.ServerStatsItem["Data Ping"]:GetValue()).."ms"
            end)
        end
    end)
end

buildGUI()

print("✅ RoBoost loaded — shadows off, particles off, FPS unlocked")
